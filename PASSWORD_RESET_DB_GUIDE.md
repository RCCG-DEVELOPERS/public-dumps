# Password reset — database and deploy guide

What has to happen on the database for the scoped password-reset work, in the
order it has to happen, and how to tell afterwards whether it worked.

This covers the two commits that have shipped so far and the role-sensitivity
step that gates the reset feature. It does not cover the reset endpoints
themselves; those are not built yet.

---

## Contents

1. [Before you touch anything](#1-before-you-touch-anything)
2. [What changes, and what does not](#2-what-changes-and-what-does-not)
3. [Step 1 — indexes on `passwordResets`](#3-step-1--indexes-on-passwordresets)
4. [Step 2 — `roles.sensitive`](#4-step-2--rolessensitive)
5. [Step 3 — widen the Organisation Init key](#5-step-3--widen-the-organisation-init-key)
6. [Verification](#6-verification)
7. [Rollback](#7-rollback)
8. [Known gaps, not yet addressed](#8-known-gaps-not-yet-addressed)

---

## 1. Before you touch anything

**The repository `.env` points at production.** A script that reads it without
care writes to the live Atlas cluster. Every script named here reads `.env`
*without overwriting what the shell already set*, so an explicit `MONGODB_URI`
wins, and every one of them echoes the host it is about to write to as its first
line. Read that line before you answer any prompt.

```
Target database host: 127.0.0.1:27017
```

Rules that do not bend:

- **Never drop anything.** No `drop()`, no `dropDatabase()`, no
  `deleteMany({})` — in any environment, tests included. This is not a style
  preference. A bulk delete wiped 55,000 production accounts on 29 August 2026.
- **Dry run first, every time.** Both scripts below take `--dry-run` or
  `--report` and print exactly what they would write.
- **Staging before production.** Nothing here is urgent enough to skip that.
- Take a snapshot of `roles` before step 2. It is 108 documents; the export
  costs nothing and turns a bad judgement into a one-line restore.

```bash
mongodump --uri "$MONGODB_URI" --collection roles --out ./backup-roles-$(date +%F)
```

---

## 2. What changes, and what does not

| Collection | Change | Migration needed? |
|---|---|---|
| `passwordResets` | Two compound indexes added | No — built at boot |
| `roles` | New `sensitive` field | No schema migration; the values are a judgement, set by script |
| `apiKeys` | Organisation Init key reaches one more path | Yes — one script, no new secret |
| `users` | Nothing | — |

**No collection is created, renamed or removed. No document is deleted. No
existing field changes meaning.** The only writes are two index builds and a
`$set` of one boolean on a handful of role documents.

**`roles.sensitive` needs no backfill.** Mongoose applies a schema default to
new documents only, so the 108 existing role documents will simply not carry the
field. That is correct and intended: absent, `false` and "never set" all mean the
same thing to the code, which tests `sensitive === true`. Running a backfill to
write `sensitive: false` everywhere would touch 108 documents to achieve nothing.

---

## 3. Step 1 — indexes on `passwordResets`

Two compound indexes were added to the schema:

```
{ email: 1, createdAt: -1 }
{ phone: 1, createdAt: -1 }
```

**Why.** The hardened reset flow now retires a caller's outstanding codes before
issuing a new one, and checks their lockout before comparing anything. Both
queries filter by identifier and sort by recency. Neither field was indexed, so
each was a full scan of the collection, on the one endpoint an attacker can call
without logging in.

**How they get built.** `MONGO_AUTO_INDEX` defaults to `true`, so mongoose
builds them when the application boots. Nothing to run.

**When to build them by hand instead.** If `passwordResets` is large, an index
build at boot delays startup and mongoose reports the failure only as a log line
the deploy will not notice. Check the size first:

```javascript
db.passwordResets.countDocuments()
db.passwordResets.getIndexes()
```

Under a few hundred thousand documents, let autoIndex do it. Above that, build
them in the background before deploying, and the boot-time build becomes a
no-op:

```javascript
db.passwordResets.createIndex({ email: 1, createdAt: -1 }, { background: true })
db.passwordResets.createIndex({ phone: 1, createdAt: -1 }, { background: true })
```

The existing TTL index on `expiresAt` is untouched. Do not alter it — MongoDB
cannot change a TTL index's key, only its duration, and the support record
planned for later depends on that index staying where it is.

---

## 4. Step 2 — `roles.sensitive`

### What the flag means

Holding a sensitive role makes someone a high-value target: taking over their
account yields money, national reach, or the power to grant more privilege.
**Nobody below super-admin may reset such a person's password**, however far
inside their own unit that person sits. National support is bound by it too.
That is the point of the flag, and it is the rule that stops Support resetting
the National Treasurer.

**It is not seniority.** "A province admin may not reset another province admin,
only a region admin may" is a separate rule, decided by comparing organisational
levels in code. It needs no flag, and flagging the geographic administrators to
express it would be actively wrong.

### The proposed list

The script proposes roles on three grounds. **This is a starting point for a
decision, not the decision.** Take the report to whoever owns the question
before you write anything.

| Group | Roles | Why |
|---|---|---|
| Hard floor | `super-admin`, `nat-support` | Also enforced in code, so a database edit cannot un-protect them |
| National tier | every role with `level_type: "national"` — about 43 | No geography contains them, so scope offers no protection and the flag is the only thing that does |
| Money above province | `cont-accountant`, `sub-cont-accountant`, `reg-accountant` | Account takeover reaches remittance funds |
| Infrastructure | `sub-cont-ict` | Can reach the systems themselves |

The national tier is selected **by query on `level_type`**, not by a list of
slugs. A slug list would go stale the first time a national role was added, and
the new role would silently be unprotected.

### Deliberately not proposed

`cont-admin`, `sub-cont-admin`, `reg-admin`, `prov-admin`, `area-admin`,
`parish-admin`, `co`, `sco`, `picr`, `picp`, `pic-zone`, `pic-area`,
`pic-parish`, `prov-accountant`, `training-manager`.

Two reasons. Seniority already governs the geographic administrators, so the
flag would add nothing. And flagging them would stop national support resetting
the very administrators it exists to help — the support job would not work.

`pic-parish` carries a third reason: 51,551 holders. Flagging it would strand
the entire parish tier behind super-admin.

`prov-accountant` is the closest call on the list. A region admin resetting a
province accountant inside their own region is ordinary support, and seniority
already stops a province admin reaching their own accountant's peer. **If the
business would rather province books were super-admin-only, that is a
defensible choice** — move the slug into `PROPOSED_SLUGS` in the script and say
so in the review.

### Running it

```bash
# 1. What exists, what would change, and what each would cost in holders.
MONGODB_URI=<target> node scripts/flagSensitiveRoles.js --report

# 2. Take that output to the decision-maker. Get an answer. Then:
MONGODB_URI=<target> node scripts/flagSensitiveRoles.js --dry-run

# 3. Write it.
MONGODB_URI=<target> node scripts/flagSensitiveRoles.js
```

The script is idempotent, never deletes a role, and writes no field but
`sensitive`. Re-running it after someone has flagged extra roles by hand leaves
those alone — it only adds, and `--report` lists them separately under
"already flagged but not proposed" so a human decision stays visible rather than
looking like drift.

`--clear` unflags everything. The hard floor survives it, because the hard floor
lives in code.

---

## 5. Step 3 — widen the Organisation Init key

The Organisation Init API key gains one path: `GET /v1/roles/sensitivity-audit`.
That is the read-only review a deploy runs before and after flagging roles.

Use `--update-scope`, **not** `--rotate`. It widens what the key may reach
without changing the secret, so a pipeline already holding the key keeps
working and nobody has to distribute a new one.

```bash
MONGODB_URI=<target> node scripts/seed-org-init-key.js --update-scope
```

The key can read the audit. It cannot set the flag. Which offices are sensitive
is a judgement, so it is made by the script under review or by a super-admin
through `PATCH /v1/roles/:id`, never by a pipeline.

---

## 6. Verification

**Indexes.** Both should appear:

```javascript
db.passwordResets.getIndexes()
```

**The flag.** Count, then spot-check a role you expect on each side:

```javascript
db.roles.countDocuments({ sensitive: true })
db.roles.findOne({ slug: "nat-support" }, { slug: 1, sensitive: 1 })
db.roles.findOne({ slug: "prov-admin" }, { slug: 1, sensitive: 1 })  // expect no flag
```

**End to end.** The audit endpoint is the real check, because it reads the same
constants the reset predicate will:

```bash
curl -H "x-api-key: $ORG_INIT_KEY" https://<host>/v1/roles/sensitivity-audit
```

Read three things in the response. `flaggedCount` matches what you approved.
`candidates` is empty, or holds only roles you consciously left alone.
`flaggedBeyondPolicy` holds only roles someone flagged on purpose.

Every call to that endpoint writes a `ROLE_SENSITIVITY_AUDIT` row to the
activity log, so the review itself is on the record.

---

## 7. Rollback

**Indexes.** Dropping them restores the previous behaviour, which was a
collection scan per reset request. There is no reason to want that, but:

```javascript
db.passwordResets.dropIndex("email_1_createdAt_-1")
db.passwordResets.dropIndex("phone_1_createdAt_-1")
```

**The flag.** `node scripts/flagSensitiveRoles.js --clear` unflags every role.
Nothing else is affected: no endpoint reads `sensitive` yet, so clearing it
today changes no behaviour at all. Once the reset endpoints ship, clearing it
would widen who may reset whom — down to the hard floor, never past it.

**The API key scope.** Re-run `--update-scope` from a checkout without the new
entry, or remove the one path from `endpointRules` by hand.

---

## 8. Known gaps, not yet addressed

State these plainly rather than discovering them later.

**`users.phone` carries no index.** `users.email` and `users.username` are both
unique-indexed; `phone` is not. Every lookup by phone number is a collection
scan across roughly 55,000 documents, including the one on the unauthenticated
reset path. Worth adding, but it belongs with the phone-normalisation work, not
here — adding the index before the values are normalised would index the
unnormalised form and have to be rebuilt.

**No rate limiting exists anywhere in the application.** The reset endpoints are
protected by the per-identifier lockout and nothing else. The lockout is a real
control because it is counted in the database; a limiter would not be, on
several Fargate tasks, unless it were database-backed too.

**`trust proxy` is not set.** Behind the load balancer, `req.ip` is the balancer,
so anything keyed on IP address collapses into a single bucket. This matters the
moment IP-keyed limits are introduced, and not before.

**`scripts/backfillUserStatus.ts` has still never been run** against a real
database. Unrelated to this work, but outstanding, and it touches `users`.

**The activity log has no TTL** and grows without bound. The audit rows this
work adds are small and infrequent, but the log is becoming load-bearing for
more than debugging.
