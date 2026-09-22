# Configuration

Environment variables for the Remittance Auth & Directory API.

> **Where the deployed values live.** `.env` and `.env.example` are gitignored, so
> this file is the only in-repo record of what the service expects. CI does not
> build a `.env` from the repo — `.github/workflows/main.yml` runs
> `cp -rf ../.env.template ./.env`, copying a template that lives **outside the
> checkout on the runner** (`dev-auth-runner`). Any variable added here must also
> be added to that `.env.template`, or the service falls back to the defaults
> below.

Every variable has a fallback, so the service always boots. Missing or unsafe
values are reported on startup with `[config]` warnings — check the boot log
after a deploy.

---

## Authentication tokens

| Variable | Default | Purpose |
|---|---|---|
| `SECRET` | `secret` | Signing key for **access** tokens. |
| `REFRESH_TOKEN_SECRET` | falls back to `SECRET` (warns) | Signing key for **refresh** tokens. |
| `SECRET_PREVIOUS` | empty | Retired access-token key, kept live during a rotation. |
| `TOKEN_EXPIRE_TIME_SEC` | `900` (15 min) | Access-token lifetime, in **seconds**. |
| `REFRESH_TOKEN_EXPIRE_TIME_SEC` | `604800` (7 days) | Refresh-token lifetime, in **seconds**. |
| `REVOCATION_CACHE_TTL_MS` | `30000` (30 s) | How stale the revoked-session cache may get. |
| `REFRESH_REUSE_GRACE_MS` | `30000` (30 s) | How long a rotated refresh token is still accepted, so a lost rotation response can be retried. `0` restores strict single-use. |

### Seconds, not milliseconds

`TOKEN_EXPIRE_TIME_SEC` replaces the older `TOKEN_EXPIRE_TIME_MSEC`. The old name
was a misnomer: `jsonwebtoken` treats a numeric `expiresIn` as **seconds**, so
`864000` was never 864 seconds — it was 10 days.

The legacy name is still read (with a deprecation warning) so existing
deployments keep booting, but rename it in `.env.template` at the first
opportunity.

### The two lifetimes are a matched pair

**`REFRESH_TOKEN_EXPIRE_TIME_SEC` must be greater than `TOKEN_EXPIRE_TIME_SEC`.**
If the refresh token expires first, the refresh endpoint can never be reached and
the whole flow is dead code. Change both together, never one alone.

On startup an inverted pair is logged as an error and corrected to
`max(604800, TOKEN_EXPIRE_TIME_SEC * 2)`. Do not rely on that — set both.

### Rolling out short access tokens

Short access tokens require the frontend to implement refresh-on-401. Until it
does, keep the long-lived pair:

```bash
# Phase 1 — non-breaking, matches historical behaviour.
# Clients without a refresh interceptor keep working.
TOKEN_EXPIRE_TIME_SEC=864000            # 10 days
REFRESH_TOKEN_EXPIRE_TIME_SEC=2592000   # 30 days

# Phase 2 — target. Switch once the frontend ships the interceptor.
# A stolen access token is then useful for minutes, not days.
TOKEN_EXPIRE_TIME_SEC=900               # 15 minutes
REFRESH_TOKEN_EXPIRE_TIME_SEC=604800    # 7 days
```

Phase 2 also shrinks the revoked-session cache window dramatically — see below.

### Separate refresh key

Access and refresh tokens carry a `type` claim, but they should not share a
signing key: with distinct keys a refresh token fails signature verification on
access-protected routes before any claim is read.

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

If `REFRESH_TOKEN_SECRET` is unset the service falls back to `SECRET` and warns.
It also warns if the two are identical.

### Access-key rotation

`SECRET_PREVIOUS` lets you change `SECRET` without signing everyone out:

1. Copy the current `SECRET` value into `SECRET_PREVIOUS`.
2. Set `SECRET` to a new key.
3. Restart.

Tokens signed with the old key still verify, but are flagged as predating the
rotation. While the password policy's `forceOnPreviousKey` is enabled (the
default), those users must set a new password before they can use the API again —
so a key rotation doubles as a credential refresh. Clear `SECRET_PREVIOUS` once
traffic on the old key has stopped.

### Session revocation timing

Logout and account deactivation revoke refresh tokens **immediately** — those hit
the database on every use. Access tokens are stateless, so they are checked
against an in-process cache of recently revoked sessions, refreshed every
`REVOCATION_CACHE_TTL_MS`.

Consequence: an already-issued access token may remain usable for up to
`REVOCATION_CACHE_TTL_MS` after logout. Lower it for faster enforcement at the
cost of more database polling.

The cache holds sessions revoked within the last `TOKEN_EXPIRE_TIME_SEC` and is
capped at 50,000 entries. Hitting the cap is logged as an error and means the
oldest revocations stop being enforced — a long `TOKEN_EXPIRE_TIME_SEC` is what
makes that reachable, so Phase 2 is the fix.

---

## Database

| Variable | Default | Purpose |
|---|---|---|
| `MONGODB_URI` | `mongodb://localhost:27017/` | Connection string. Include the database name in the path. |
| `MONGODB_DB_MAIN` | `example_db` | Database name. Under `NODE_ENV=test`, `_test` is appended. |

Note the connection is built from `MONGODB_URI` alone, so the database name must
be part of that URI. A URI with no path segment connects to `test`.

The `refreshTokens` collection carries a TTL index on `expiresAt`; MongoDB reaps
expired tokens automatically, no cron required.

---

## Activity log retention and archive

`activityLogs` is append-only and is the busiest collection here — roughly
30,500 inserts a day, about 1 GB a month. **Nothing is ever deleted.** Entries
older than the retention window are moved to an **Atlas Online Archive**, where
they remain queryable.

### The archive rule

Configured in the **Atlas console**, not here — the application cannot set it,
and `.env` has no say over it. Recorded in this file so the settings are not
folklore.

| Setting | Value |
|---|---|
| Collection | `activityLogs` |
| Date field | `createdAt` |
| Age threshold | **7 days** (staged 180 → 90 → 30 → 7 on rollout) |
| Partition fields | `createdAt`, then `userId` |

The threshold is editable in Atlas at any time, with no deploy. **The partition
fields are permanent** — they become the storage path, and changing them means a
new archive.

**Never add a TTL index to this collection.** Online Archive moves documents;
TTL deletes them. The two look alike in a schema diff and only one of them
honours the never-delete requirement.

### What archiving changes

The application connects to the cluster, and the cluster holds only the live
window. So:

- `GET /v1/activityLogs`, `POST /v1/activityLogs/search`, `/user/:userId` and
  `/session/:sessionId` return **live entries only**, with no error — just a
  shorter list.
- `totalCount` reflects the live collection, not the whole history.
- `GET /v1/activityLogs/:id` for an archived entry returns **404**, and says the
  entry may be in the archive. (It previously returned 200 with a `null` body.)
- Cluster snapshots and point-in-time restore **do not cover archived data**.
  Restoring a pre-archive snapshot re-materialises archived documents, which
  then exist in both places with the same `_id`.
- Disk usage falls more slowly than document count — WiredTiger does not return
  freed extents to the filesystem, so snapshot savings lag by weeks.

### Reading the archive

`POST /v1/activityLogs/archive/search` — **super-admin only**, because each call
is billed by the volume it scans.

```json
{
  "from": "2026-06-01T00:00:00Z",
  "to":   "2026-06-30T23:59:59Z",
  "filters": { "userId": "...", "activity": "LOGIN" },
  "limit": 100
}
```

Archived data has **no indexes** — only date partitions — so every query is
bounded by construction:

| Rule | Why |
|---|---|
| `from` and `to` are **required** | The date is the partition key; without it a query reads the whole archive |
| Range capped (`ACTIVITY_ARCHIVE_MAX_RANGE_DAYS`, default 31) | Keeps the scanned volume, and the bill, predictable |
| `limit` capped at 500, defaults to 100 | Bounds the response |
| Filters are **exact match on an allowlist** | A pattern match cannot be indexed and would read every partition in range |

Errors carry a `code`: `DATES_REQUIRED`, `BAD_DATE`, `RANGE_INVERTED`,
`RANGE_TOO_WIDE`, `UNKNOWN_FILTER`. A `503` means the archive is not configured
in this environment.

| Variable | Default | Purpose |
|---|---|---|
| `ACTIVITY_ARCHIVE_URI` | empty | Atlas **federated** connection string. Empty disables the endpoint (503). Secret. |
| `ACTIVITY_ARCHIVE_MAX_RANGE_DAYS` | `31` | Widest span one query may cover. |
| `ACTIVITY_ARCHIVE_MAX_TIME_MS` | `45000` | Server-side limit for an archive query. Higher than the live 30s cap because federated reads take seconds. |

The archive connection is opened **lazily**, on first use, and is separate from
the application's own. It does not inherit the 30s `maxTimeMS` applied to
cluster queries, so it sets its own.

### Indexes

Five compound indexes, each ending in `createdAt` so listings are served in
order rather than sorted in memory:

```
{ createdAt: -1 }                    findAll, and the field the archive rule scans
{ userId: 1,    createdAt: -1 }      findByUser
{ sessionId: 1, createdAt: 1 }       findBySession (ascending)
{ activity: 1,  createdAt: -1 }      "every LOGIN, newest first"
{ module: 1,    createdAt: -1 }      "everything the AUTH module did"
```

These replace fourteen single-field indexes, cutting the B-trees written per
insert from fifteen to six. Build them with
`npx ts-node scripts/createActivityLogIndexes.ts --dry-run` first — out of band,
because with `MONGO_AUTO_INDEX` on every task asserts every index at boot.

Removing a declaration does **not** drop an index; mongoose never drops. The old
fourteen must be dropped by hand, and only after `$indexStats` shows them unused.

### Ingest

`POST /v1/activityLogs/ingest` (API key) takes one entry, or up to 100 as
`{ "logs": [...] }`. It answers **202** and writes behind the response; add
`?sync=true` for the old 201-with-id behaviour.

**There is deliberately no rate limit.** A limiter on an audit endpoint sheds
audit records, which defeats its purpose. The bounds are on size instead:
`details` is capped at 2000 characters (rejected with 400 here, truncated on the
internal path — a shortened record beats a lost one), a batch at 100 entries,
and the JSON body at its own limit. A misbehaving client is dealt with by
revoking its API key, and volume is worth an **alert**, not a throttle.

## CORS

| Variable | Default | Purpose |
|---|---|---|
| `CORS_WHITELIST` | built-in list | Comma-separated allowed origins. Blank uses the built-in list. |
| `CORS_DISABLED` | `false` | `true` disables CORS restrictions. **Local development only.** |

---

## API keys

| Variable | Default | Purpose |
|---|---|---|
| `API_KEY_ALLOWED_IPS` | empty | Global IP/CIDR allowlist for all API-key requests. Empty = no global restriction; per-key `allowedIPs` still applies. |
| `API_KEY_ALLOWED_DOMAINS` | empty | Global Origin allowlist (include the scheme). Empty = no global restriction. |

---

## Role authority

| Variable | Default | Purpose |
|---|---|---|
| `NON_ADMINISTRATIVE_ROLES` | `training-manager` | Comma-separated role slugs that sit at a geographic level but carry **no administrative authority** there. |

`standingOf` grants authority purely from a role's `level_type`. That is right
for an `sco` or a `cont-admin` and wrong for a role that merely happens to
operate at that level. A slug listed here contributes no unit to a caller's
standing, so it reaches none of the six administrative surfaces: geofencing
rules, geofencing exemptions, principal-officer appointment, hierarchy moves,
headquarters assignment, and approvals.

```bash
NON_ADMINISTRATIVE_ROLES=training-manager
NON_ADMINISTRATIVE_ROLES=training-manager,region-training-manager
```

**Every ambiguous value fails towards LESS authority, not more:**

| Value | Result |
|---|---|
| unset | the default, `training-manager` |
| empty, or the whole line commented out | the default — an empty line is likelier to be an accident than a deliberate widening |
| `none` | genuinely empty. Clearing the list has to be said out loud |
| `Training-Manager` | lowercased; slugs are lowercase everywhere |
| `training-manager # trainers only` | the comment is stripped |

That last row matters. **dotenv 4 does not strip inline `#` comments**, so
without special handling the slug would be stored as
`training-manager # trainers only`, match no role at all, and silently restore
the authority the setting exists to remove. The same fault once left New Relic
log forwarding disabled for weeks.

What this does **not** change: a listed role still sees its own unit's data
(`scopeResolver` reads `level_type` directly), can still be assigned and held,
and can still be a principal office. It governs what holding the role lets a
person do to others, nothing else.

> When region, province, zone, area and parish training-manager roles are
> created, **each new slug must be added here** or it will silently inherit
> administrative authority at its own level.

### Who may restructure a unit

| Variable | Default | Purpose |
|---|---|---|
| `HIERARCHY_RESTRUCTURE_ROLES` | `prov-admin,reg-admin,sub-cont-admin,cont-admin` | Role slugs that may use `POST /v1/hierarchy-transfers/scoped/promote` and `/scoped/realign` — promote or repair a unit within their own unit. |

Deliberately narrower than standing. Standing says a `picp` acts at province
level; this says only the `prov-admin` *restructures* the province. Heads of
unit (`picp`, `picr`, `sco`, `co`) and every assistant (`prov-asst-admin`,
`reg-asst-admin`, `apicp-admin`, …) are excluded by default. Standing for these
two endpoints is then computed from the qualifying roles **alone**, so a
`prov-admin` who is also `picr` of a region does not reach the region through
them.

Same parsing as `NON_ADMINISTRATIVE_ROLES`: unset or empty keeps the default,
`none` clears it (which locks the scoped endpoints to super-admin), inline `#`
comments are stripped, slugs are lowercased.

```bash
HIERARCHY_RESTRUCTURE_ROLES=prov-admin,reg-admin,sub-cont-admin,cont-admin
```

See [HIERARCHY_SCOPED_OPERATIONS_DOCS.md](HIERARCHY_SCOPED_OPERATIONS_DOCS.md).

### Who acts with super-admin authority over the hierarchy

| Variable | Default | Purpose |
|---|---|---|
| `ELEVATED_ROLES` | `nat-support` | Role slugs treated as **super-admin-equivalent** over the hierarchy, principal officers, HQ assignment and approvals — and over nothing else. |

`standingOf` marks a caller holding one of these as `isElevated`, and a single
predicate — `isUnbounded` in `@/utils/officerAuthority` — is what the widened
surfaces read. The bound is therefore a matter of *which call sites read it*,
and those are exactly:

| Surface | What an elevated caller may do |
|---|---|
| Hierarchy | `POST /v1/hierarchy-transfers/transfer`, `/admin/move`, `/promote`, `/admin/realign`, `/scoped/*`, `GET /jobs`, `GET /integrity` — unbounded, including cross-province moves and `absorbCodes`. |
| Principal officers | every `/v1/principal-officers` route: roster reads, appoint, end, transfer, `/admin/strip-roles`. |
| HQ assignment | reads, `/assign`, `/vacate`, `/resolve-conflict`, `/vacancies`, `/conflicts`, `GET /integrity`. |
| Approvals | approve, reject or cancel a request of **any** type, including the officer types that are otherwise super-admin only. |

What it does **not** grant, on purpose: geofencing rules, config and exemptions;
AppIcons; Secondaryroles; account status and forced password change;
`/impersonate` and `/impersonate-role`; role granting (`roleGrants`,
`roleAssignment`); API keys; `/admin/backfill-codes`; the HQ repair routes; the
login geofence bypass; any route behind `jwtConfig.requireSuperAdmin`. Nor the
self-approval exemption — an elevated caller still cannot approve a request they
raised themselves.

Every action taken on elevated standing is audit-logged with
`authorisedVia: "elevated:<slug>"`, never `"super-admin"`, so the two are never
confused in a log. Same parsing as `NON_ADMINISTRATIVE_ROLES`: unset or empty
keeps the default, `none` hands every surface above back to super-admin alone.

```bash
ELEVATED_ROLES=nat-support
ELEVATED_ROLES=none
```

Elevation on the standing-based routes is judged on the roles a user **holds**
(`users.roles`), not on an active secondary role — so an elevated slug should be
held as a primary role.

### Who may assign a role

| Variable | Default | Purpose |
|---|---|---|
| `ROLE_ASSIGNMENT_ENFORCEMENT` | `off` | `off`, `warn` or `enforce`. Whether a caller's authority is checked before roles are written. |
| `ROLE_GRANT_UNRESTRICTED` | the heads and administrators, below | Comma-separated role slugs that may assign **any** role at their level or below. |

`vetRoles` has always answered *what* may be assigned — never `super-admin`,
never an occupied principal office. It never answered *who*, and none of the
three role-writing endpoints (`POST /v1/users`, `PATCH /v1/users/:id`,
`PATCH /v1/users/:id/roles`) carried a caller guard, so any authenticated
account could grant itself `picp` or `cont-admin` by patching its own id.

The rule, judged on the caller's **active role** — the one they switched into,
not the set of roles they hold:

1. `super-admin` may assign anything except `super-admin`
2. a role **above** the caller's level is refused — `NOT_YOUR_LEVEL`
3. a target **outside** the caller's own unit is refused — `NOT_YOUR_UNIT`
4. a caller in `ROLE_GRANT_UNRESTRICTED` may assign anything at their level or below
5. anyone else may assign roles sharing their own `level_scope` — the discipline:
   `pastor`, `admin`, `accountant`, `ict`, `department` — and nothing else,
   otherwise `NOT_YOUR_DISCIPLINE`

So `prov-accountant` may appoint `prov-asst-accountant` and `parish-accountant`,
but not `prov-admin`; `picp` of LA47 may appoint any role in LA47 and below, but
nothing in a neighbouring province and nothing at region level.

**Why the active role and not the set held.** Take the broadest rank from one
role and the union of the disciplines from all of them, and you grant authority
no single role confers:

| Caller holds | Alone permits |
|---|---|
| `reg-accountant` | accountant roles, region and below |
| `parish-admin` | any role, parish only |
| *the two, blended* | *any role, region and below* — **including `reg-admin`** |

The rank comes from one and the unrestricted flag from the other, and they meet
in the middle. With one active role there is nothing to blend.

#### The three modes

| Value | Effect |
|---|---|
| `off` (default, and anything unrecognised) | no check, no extra query. The diff is invisible. |
| `warn` | roles are assigned as before, and every refusal the rule *would* have made is written to the activity log with status `WARNING` |
| `enforce` | refused roles are stripped, each with a code and an explanation, exactly like an occupied principal office |

As with `NON_ADMINISTRATIVE_ROLES`, an unrecognised value means `off` — a
misspelling, a blank, or an uncleaned inline `#` comment must never switch on a
breaking change by surprise.

> **Read the `warn` logs before setting `enforce`.** Only 63 of 108 roles carry
> a geographic `level_type`; the other 45 are `national`, `house`, `rpms`,
> `department` and so on, and rule 2 refuses all of them for having no level to
> assign from. Whether the national tier genuinely assigns roles is the number
> this rollout exists to discover — add whatever it turns out to need to
> `ROLE_GRANT_UNRESTRICTED` rather than guessing now.

```bash
ROLE_ASSIGNMENT_ENFORCEMENT=warn
ROLE_GRANT_UNRESTRICTED=pic-parish,pic-area,pic-zone,picp,picr,sco,co,parish-admin,area-admin,prov-admin,reg-admin,sub-cont-admin,cont-admin
```

The default for `ROLE_GRANT_UNRESTRICTED` is the head and the administrator of
each unit, exactly as listed above. **Their assistants are deliberately absent**
— `apicp-admin`, `apicp-csr`, `apicp-csr-2`, `prov-asst-admin`, `reg-asst-admin`,
`apicr`, `apicr-csr`, `asco`, `aco` fall under the discipline rule, so an assistant accountant cannot appoint the
administrator above them. It accepts the same ambiguous values as
`NON_ADMINISTRATIVE_ROLES`: unset or empty keeps the default, `none` clears it,
inline comments are stripped.

`GET /v1/users/me/allowed-roles` answers "what may I assign?" for the caller's
current active role, under **every** mode including `off`, so a role picker can
be built and shipped before enforcement is turned on. Its `mode` field says
whether the list is advisory or binding.

---

## Which commit is running

`GET /health` reports the commit the process was built from:

```jsonc
{
  "status": "healthy",
  "version": "374f961c1e2d…",          // APP_VERSION, as before
  "commit": "374f961c1e2d3a4b…",       // full hash, or "N/A"
  "build": {
    "commit": "374f961c1e2d3a4b…",
    "shortCommit": "374f961",
    "branch": "dev",                    // when known
    "builtAt": "2026-09-15T10:00:00Z",  // when the build stamped it
    "source": "env"                     // env | build-info | git | none
  }
}
```

It is resolved **once**, from the first of these that has an answer, and is
`"N/A"` when none does — the endpoint never fails on account of it. No package
and no `git` subprocess is involved.

| Order | Source | Where it comes from |
|---|---|---|
| 1 | `GIT_COMMIT` | said out loud in the environment. Optional; `GIT_BRANCH` beside it is shown when set |
| 2 | `APP_VERSION` | the ECS image tag. `deploy-main.yml` passes the commit as `--build-arg APP_VERSION`, so on Fargate this is the answer. Only counted when it *looks like* a hash — the default `unknown` is skipped |
| 3 | `build/build-info.json` | stamped by `scripts/writeBuildInfo.js` as the last step of `npm run build`. On an EC2 box that reads the checkout's `.git`; inside a Docker build there is no `.git` and it records `N/A`, which is then skipped in favour of 2 |
| 4 | `.git/HEAD` | read directly from the working directory upward — the EC2/PM2 case, where the checkout stays on the box |
| — | `"N/A"` | |

`source` says which one answered, so a surprising value can be traced.

**The stamp never fails a build.** `scripts/writeBuildInfo.js` catches everything,
always writes a file, and always exits 0. It is the one file under `scripts/`
that `.dockerignore` lets into the image build, because `npm run build` calls it.

---

## Runtime

| Variable | Default | Purpose |
|---|---|---|
| `GIT_COMMIT` | unset | Optional explicit commit hash for `/health`. Normally unnecessary — see "Which commit is running" above. |
| `MAX_PAGE_SIZE` | `100000` | Ceiling on `?pageSize=` for every paged listing. **Deliberately generous for now** so nothing that works today breaks; it bounds the absurd, not the merely large — lower it to a few hundred once the bulk callers page. A request above it gets the maximum, and the response's `pageSize` shows the value actually applied. `?pageSize=abc`, `0` or a negative falls back to the endpoint's default (20 unless stated); `?pageNo=0` or a negative is page 1. Before this cap `?pageSize=500000` was honoured, which is how `/v1/users/search` came to return 80 MB responses. Unparseable or non-positive means the default. |
| `NODE_ENV` | `development` | `development`, `production`, or `test`. |
| `PORT` | `8484` | HTTP listen port. |
| `CRON_ENABLED` | `true` | `false` stops this process registering the birthday and parish-snapshot crons. Used on ECS, where the API tasks run with it off and one dedicated task runs with it on, so each job fires once instead of once per task. Leave unset on a single PM2 instance. |

## Privileged audit trail

One durable record per request made by a super-admin, an elevated role, or
anyone inside an impersonated session — written to `privilegedAuditLogs`, which
is separate from `activityLogs` so the 3.2M-document collection takes no extra
write load or index.

`activityLogs` cannot answer "who was really behind this": `logActivity` reads
the actor from the JWT, and under impersonation that claim *is* the person being
impersonated. This collection records both identities, plus the session id that
is stable across token refresh, so a borrowed session can be replayed end to end.

| Variable | Default | Meaning |
|---|---|---|
| `PRIVILEGED_AUDIT_MODE` | `impersonation` | `off` \| `impersonation` \| `elevated`. `impersonation` captures every request of a borrowed session — a few hundred rows a day. `elevated` adds all super-admin and nat-support work, which is 15k–90k a day; watch collection growth before promoting to it. An unrecognised value falls back to `impersonation`, never to the expensive one. |
| `PRIVILEGED_AUDIT_MAX_BODY_CHARS` | `4096` | Request bodies are truncated, never rejected. The real size is recorded separately. |
| `PRIVILEGED_AUDIT_RETENTION_DAYS` | `400` | TTL, on a dedicated `expiresAt` field so a legal hold can pin one session by unsetting it. Unlike `activityLogs`, this collection *does* expire — it is a request firehose with bodies attached, not a business record. |
| `PRIVILEGED_AUDIT_FLUSH_MS` | `1000` | Buffer window. On SIGKILL up to this much is lost. |
| `PRIVILEGED_AUDIT_FLUSH_MAX_DOCS` | `50` | Flush early once this many are queued. |
| `PRIVILEGED_AUDIT_MAX_QUEUE` | `1000` | Overflow spills the oldest to stderr rather than dropping it silently. |

Bodies are redacted with the same `redactDeep` the request log uses, plus a local
list covering identity and bank fields. Reads are super-admin only via
`/v1/privileged-audit`, except `/me/sessions`, which is scoped server-side to the
caller. **Run `scripts/createPrivilegedAuditIndexes.ts` before enabling.**

## User hierarchy sync

Repairs users whose `area`/`zone`/`province`/`region`/`subContinent`/`continent`
codes have drifted from `parishDirectory`. Transfers already cascade a *unit*
move onto its members; this covers what they cannot see — a user whose own
parish changed, a directory row edited outside a transfer, and users whose
parish code exists nowhere.

Users are grouped by their distinct hierarchy before anything is decided, so
~55,000 users become ~25,000 tuples and a handful of writes.

**Never modified, in either mode:** a user whose parish code is in no directory
row (orphan), one whose parish is a DEPARTMENT row, and one whose parish is split
across rows that disagree. All three are reported for a human. Repair a split
with `POST /v1/hierarchy-transfers/admin/realign`.

| Variable | Default | Meaning |
|---|---|---|
| `HIERARCHY_SYNC_ENABLED` | *(off)* | `true` schedules the cron. Opted into rather than inherited: unlike the other crons, this one writes to user records. |
| `HIERARCHY_SYNC_MODE` | `report` | `report` \| `apply`. Report writes nothing and still says exactly what it would change. Run it, read three weeks of numbers, then flip. |
| `HIERARCHY_SYNC_CRON` | `20 2 * * 0` | 02:20 Sunday. Weekly, because drift arrives from rare events and a nightly pass only raises the chance of colliding with a transfer. An invalid expression is refused and logged, not thrown at boot. |
| `HIERARCHY_SYNC_MAX_MODIFY_PCT` | `20` | Circuit breaker. A run that would rewrite more than this share of users is aborted: at that scale the fault is in the directory, and propagating it is the incident. |
| `HIERARCHY_SYNC_MAX_MODIFY` | `0` | Absolute ceiling; `0` means percentage only. |
| `HIERARCHY_SYNC_MIN_PARISHES` | `10000` | Refuses to apply if fewer parish codes loaded than this — a half-loaded directory would classify most of the population as stale. |
| `HIERARCHY_SYNC_CHUNK` | `50` | Tuples per write chunk. |
| `HIERARCHY_SYNC_PAUSE_MS` | `250` | Pause between chunks. |
| `HIERARCHY_SYNC_REPORT_CAP` | `500` | Entries per list on the run record. A capped run sets `changesComplete: false`. |
| `HIERARCHY_SYNC_MAX_TIME_MS` | `600000` | Per-aggregation cap. Required: the connection-wide 30s limit would kill both passes. |
| `HIERARCHY_SYNC_ON_SWITCH` | `true` | Refreshes one user's codes from the directory when they switch role, so the new token carries a current scope rather than a stale one. One indexed `findOne` — production carries `{ parishCode: 1 }` as `parish_code_unique`, verified 2026-09-22. Set `false` to disable. An unresolvable parish never blocks the switch; the user carries on with what they have. |

Inspect with `GET /v1/hierarchy-sync/runs` and `/orphans` (elevated), trigger with
`POST /v1/hierarchy-sync/run` (super-admin for `{"mode":"apply"}`), or run
`scripts/syncUserHierarchy.ts`, which reports by default and needs `--apply` typed.

---

## Password policy

Not environment-driven — stored in the `authPasswordPolicy` collection and
managed at runtime:

- `GET /auth/password-policy` — read current policy (super-admin)
- `POST /auth/password-policy` — update it (super-admin)
  - `forceAll: true` requires every user whose password predates this moment to
    set a new one; `false` clears the requirement
  - `forceOnPreviousKey` toggles whether a key rotation forces a change

Users needing a password change are blocked from every route except
`/auth/change-password`, `/auth/logout` and `/auth/logout-all`, and receive
`403` with `code: "PASSWORD_CHANGE_REQUIRED"` and a `reason` of
`ADMIN_REQUIRED`, `SIGNING_KEY_ROTATED`, or `GLOBAL_POLICY`.

---

## S3 uploads

`POST /v1/uploads` — the single upload endpoint for **every** file type: images,
video, audio and documents. Accepts `multipart/form-data`, uploads to S3, returns
URLs and metadata. Authenticate with a bearer token (frontend) **or** an
`x-api-key` (microservices). `GET /v1/uploads/health` reports whether uploads are
usable without uploading anything.

The frontend does not choose an endpoint or declare a file type. Each file's type
is detected from its own bytes, and that decides three things: which size limit
applies, which subfolder it is filed under, and whether it is compressed.

| Detected | Subfolder | Default limit | Compressed |
|---|---|---|---|
| image | `images/` | 10 MB | yes, via sharp |
| video | `videos/` | 50 MB | no |
| audio | `audio/` | 50 MB | no |
| document | `docs/` | 25 MB | no |

`POST /v1/uploads/images` still exists and still accepts images only. It predates
the unified endpoint and is kept so existing callers keep working; new callers
should use `POST /v1/uploads`.

| Variable | Default | Purpose |
|---|---|---|
| `AWS_REGION` | — | Bucket region. Required. |
| `S3_BUCKET` | — | Target bucket. Required. |
| `S3_PUBLIC_BASE_URL` | empty | CDN origin for returned URLs. Blank uses the S3 virtual-hosted URL. |
| `S3_KEY_PREFIX` | empty | Forced prefix for every object; a caller cannot write outside it. |
| `UPLOAD_MAX_IMAGE_BYTES` | `10485760` | Per-file ceiling for images. |
| `UPLOAD_MAX_VIDEO_BYTES` | `52428800` | Per-file ceiling for video (50 MB). |
| `UPLOAD_MAX_AUDIO_BYTES` | `52428800` | Per-file ceiling for audio (50 MB). |
| `UPLOAD_MAX_DOCUMENT_BYTES` | `26214400` | Per-file ceiling for documents (25 MB). |
| `UPLOAD_ALLOWED_IMAGE_MIME` | image set | Allowlist matched against the **detected** type. |
| `UPLOAD_ALLOWED_VIDEO_MIME` | video set | mp4, mov, webm, mkv, avi, m4v, 3gp, flv, ogv. |
| `UPLOAD_ALLOWED_AUDIO_MIME` | audio set | mp3, m4a, wav, ogg, flac, amr. |
| `UPLOAD_ALLOWED_DOCUMENT_MIME` | document set | pdf, doc(x), xls(x), ppt(x), odt, ods, rtf, txt, csv. |
| `UPLOAD_FOLDER_IMAGE` | `images` | Subfolder name for images. |
| `UPLOAD_FOLDER_VIDEO` | `videos` | Subfolder name for video. |
| `UPLOAD_FOLDER_AUDIO` | `audio` | Subfolder name for audio. |
| `UPLOAD_FOLDER_DOCUMENT` | `docs` | Subfolder name for documents. |
| `UPLOAD_MAX_FILE_BYTES` | `10485760` | Legacy name. Now only the default for `UPLOAD_MAX_IMAGE_BYTES`. |
| `UPLOAD_ALLOWED_MIME` | image set | Legacy name, still read by `POST /v1/uploads/images`. |

The multipart parser's own ceiling is derived as the **largest** of the four
limits, because it aborts a request before any handler can know the file's type.
Raising `UPLOAD_MAX_VIDEO_BYTES` therefore also raises how much a request can
buffer in memory before being rejected — the per-category limit is what actually
refuses an oversized file, and it does so after sniffing.

Credentials come from the AWS SDK default provider chain — an IAM task/instance
role is used when available, so no keys need to sit in the environment.
`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` still work for local development.
The IAM policy needs `s3:PutObject`, plus `s3:DeleteObject` for rollback and
`s3:ListBucket` for the health probe.

### Request

```
POST /v1/uploads
Authorization: Bearer <token>          # or: x-api-key: <key>
Content-Type: multipart/form-data

files=@photo.jpg                       # any field name; repeat for several files
files=@service.mp4                     # mixed types in one request are fine
path=parishes/RC12345/gallery          # optional destination prefix
category=image,document                # optional; restrict to broad groups
accept=image/*,application/pdf         # optional; restrict to exact types
```

Two optional guards, both enforced server-side. Use whichever fits:

`category` restricts by broad group — `image`, `video`, `audio`, `document`,
comma-separated. A profile-picture form sends `category=image` and a video is
refused. Folder names work as aliases, so `docs` and `document` are equivalent.

`accept` restricts by exact type, using the same vocabulary as the `accept`
attribute on `<input type="file">`: `image/*`, `application/pdf`, `.pdf`. Reach
for it when a form wants *some* of a category but not all of it.

A "scanned document" form is the motivating case. `category=image,document`
accepts the scans and the PDF — but `document` is the whole group, so `.docx`,
`.xlsx`, `.csv` and `.txt` get in too. `accept=image/*,application/pdf` refuses
those while still filing scans under `images/` and the PDF under `docs/`:

```
                              category=image,document    accept=image/*,application/pdf
scan.png / scan.jpg           accepted -> images/         accepted -> images/
passport.pdf                  accepted -> docs/           accepted -> docs/
notes.docx / budget.xlsx      accepted -> docs/           TYPE_NOT_ACCEPTED
data.csv                      accepted -> docs/           TYPE_NOT_ACCEPTED
clip.mp4                      CATEGORY_NOT_ALLOWED        TYPE_NOT_ACCEPTED
```

Both may be combined; a file must satisfy both. Note that `accept` matches the
**detected** type, so `.pdf` cannot be satisfied by renaming a PNG to `scan.pdf`
— that is refused with `TYPE_NOT_ACCEPTED`, and a renamed binary is refused
earlier still with `UNKNOWN_FILE_TYPE`. The client-side `accept` attribute
remains a UI hint; this is the enforcement.

Stored names are unguessable: a 24-hex-character random prefix leads, followed by
the original name lowercased with spaces and punctuation folded to single dashes,
capped at 60 characters. The extension comes from the detected type, never from
the supplied filename. So `My Vacation Photo (2026)!!.PNG` is stored as
`c360f2b1e329c2f5ebfbc8d5-my-vacation-photo-2026.png`. Uploading the same
filename twice yields two distinct keys — nothing is ever silently overwritten.

### Response

```json
{
  "success": true,
  "message": "Uploaded 2 file(s)",
  "path": "parishes/RC12345/gallery",
  "count": 2,
  "files": [{
    "url": "https://cdn.example.org/parishes/RC12345/gallery/images/c360f2b1e329c2f5ebfbc8d5-photo.jpg",
    "key": "parishes/RC12345/gallery/images/c360f2b1e329c2f5ebfbc8d5-photo.jpg",
    "bucket": "rccg-assets",
    "originalName": "photo.jpg",
    "storedName": "c360f2b1e329c2f5ebfbc8d5-photo.jpg",
    "category": "image",
    "folder": "images",
    "mimeType": "image/jpeg",
    "declaredMimeType": "image/jpeg",
    "extension": "jpg",
    "size": 184320,
    "originalSize": 902144,
    "compressed": true,
    "savedPercent": "80%",
    "compressionSkippedReason": "",
    "width": 1920,
    "height": 1080,
    "checksumSha256": "9f86d081...",
    "etag": "d41d8cd98f00b204e9800998ecf8427e",
    "uploadedAt": "2026-08-20T09:15:00.000Z"
  }, {
    "url": "https://cdn.example.org/parishes/RC12345/gallery/videos/4852d4a7d73e6c2602047247-service.mp4",
    "key": "parishes/RC12345/gallery/videos/4852d4a7d73e6c2602047247-service.mp4",
    "bucket": "rccg-assets",
    "originalName": "service.mp4",
    "storedName": "4852d4a7d73e6c2602047247-service.mp4",
    "category": "video",
    "folder": "videos",
    "mimeType": "video/mp4",
    "declaredMimeType": "video/mp4",
    "extension": "mp4",
    "size": 31457280,
    "originalSize": 31457280,
    "compressed": false,
    "savedPercent": "0%",
    "compressionSkippedReason": "NOT_AN_IMAGE",
    "width": 0,
    "height": 0,
    "checksumSha256": "a1b2c3d4...",
    "etag": "e5f6...",
    "uploadedAt": "2026-08-20T09:15:02.000Z"
  }],
  "failed": []
}
```

`width` / `height` are populated for images only; parsing A/V containers for
dimensions is not worth the cost. Non-images always report `compressed: false`
with `compressionSkippedReason: "NOT_AN_IMAGE"` — there is no general-purpose
transcoder here, and re-encoding a video inside a request would not be viable.

A mixed batch returns `200` with the successes in `files` and the rejects in
`failed`, each carrying a `code`:

| Code | Meaning |
|---|---|
| `UNKNOWN_FILE_TYPE` | Content matched no supported format. Covers a renamed executable, and a bare `.zip`. |
| `CATEGORY_NOT_ALLOWED` | Real type is not in the requested `category` (or the endpoint is images-only). |
| `TYPE_NOT_ACCEPTED` | Real type does not match the requested `accept` patterns. |
| `MIME_NOT_ALLOWED` | Detected type is not in that category's allowlist. |
| `TOO_LARGE` | Over the limit **for its own category** — the message names both. |
| `EMPTY_FILE` | Zero bytes. |
| `UPLOAD_FAILED` | S3 write failed. |

If **every** file fails the status is `400`; if S3 itself is unconfigured it is
`503`, since that is a server fault.

### Type detection

Detection reads magic bytes, never the multipart `mimetype` header — that is
client-supplied, so a `.exe` announced as `image/png` would otherwise be stored.
`Content-Type` on the S3 object is set from the detected type, so a browser never
content-sniffs an asset either.

Two consequences worth knowing:

- **A bare `.zip` is rejected**, deliberately. It is a container for arbitrary
  content, so accepting it would defeat the point of sniffing. `.docx`/`.xlsx`/
  `.pptx`/`.odt`/`.ods` are ZIPs too, and are accepted because their internal
  part names identify them.
- **CSV vs plain text** is the one place the filename is consulted, and only
  after the bytes are already proven to be text — CSV has no signature.

### Compression

Images are compressed between validation and the S3 write, using `sharp`
(libvips). Measured on representative inputs:

| Input | Stored | Saved |
|---|---|---|
| 4000x3000 JPEG q95 (phone photo) | 271 KB, resized to 2560x1920 | **79%** |
| 2000x1500 photographic PNG | 849 KB | **85%** |
| 1200x800 JPEG q90 | 45 KB | 37% |
| 800x600 JPEG already at q60 | unchanged | 0% — kept original |
| 3000x2000 JPEG, `convertToWebp` | 59 KB | **88%** |

| Variable | Default | Purpose |
|---|---|---|
| `IMAGE_COMPRESSION_ENABLED` | `true` | `false` stores originals untouched. |
| `IMAGE_MAX_WIDTH` / `IMAGE_MAX_HEIGHT` | `2560` | Scale-down ceiling. Never upscales. |
| `IMAGE_QUALITY` | `82` | 1-100. Lossy encoders; PNG palette quantisation. |
| `IMAGE_CONVERT_TO_WEBP` | `false` | Transcode all but animated GIF to WebP. |
| `IMAGE_MAX_PIXELS` | `50000000` | Decoded-pixel ceiling; decompression-bomb guard. |

Per-request overrides may be sent as form fields alongside the file:
`compress=false`, `maxWidth`, `maxHeight`, `quality`, `convertToWebp`. Numeric
values are clamped, so a hostile value cannot force an enormous resize.

Behaviour worth knowing:

- **Never larger than the original.** Re-encoding an already-optimised image
  usually grows it. When the result is bigger and neither dimensions nor format
  changed, the original bytes are kept and `compressionSkippedReason` is
  `NO_GAIN`.
- **Animated GIFs are passed through byte-for-byte** (`ANIMATED`). Re-encoding
  them as a still image would silently destroy the animation.
- **EXIF is stripped, rotation is not lost.** The orientation tag is applied to
  the pixels first, so portrait phone photos stay upright; the metadata — which
  includes GPS coordinates on camera photos — is then dropped. These objects are
  served publicly, so that matters.
- **HEIC/HEIF and BMP are transcoded to JPEG**, since browsers cannot render
  them. Storing one verbatim yields an asset the frontend cannot display.
- **Never fails an upload.** A decode error stores the original and reports
  `COMPRESSION_FAILED` rather than losing the file.
- Oversized images are refused with `TOO_MANY_PIXELS`, checked from the header
  before any decode.

Each response file carries `originalSize`, `size`, `compressed`, `savedPercent`
and `compressionSkippedReason`, and the S3 object gets `original-size` and
`compressed` user metadata.

`sharp` is a native module. The lockfile records every platform variant
(including `@img/sharp-linuxmusl-x64`), so `npm ci` in the `node:20-alpine`
image resolves the correct binary with no Dockerfile change.

### Security notes

- The declared content type is ignored. Files are identified from their own
  leading bytes, so a binary renamed `.png` is rejected.
- `path` is hostile input and is sanitised, not trusted: traversal, absolute
  paths, backslashes, control characters and shell metacharacters are stripped.
  `../../etc/passwd` becomes `etc/passwd`, and `S3_KEY_PREFIX` still applies on
  top, so the object cannot escape its prefix.
- Object `Content-Type` is set explicitly so browsers never content-sniff.
- Stored names are randomised, so uploads cannot overwrite each other or be
  guessed.

---

## Postcode address verification

`GET /v1/address/postcode?country=GBR&postcode=SW1A%201AA` (also accepts POST with
a JSON body).

> **Temporarily unauthenticated.** The postcode lookup and
> `/postcode/countries` are open, so the address form can be used during
> registration before a token exists. `/v1/address/health` still requires a
> bearer token or `x-api-key`. Melissa bills per lookup, so the remaining
> guards carry the load: input validation before the outbound call, a 24h
> response cache, and a per-caller rate limit that keys on the client IP when
> there is no authenticated actor. Restore auth by putting
> `apiKeyConfig.isAuthenticatedOrApiKey` back on the `/v1/address` mount in
> `src/routes/index.ts`.

This moved off the frontend, which called Melissa directly with the licence key
embedded in the React bundle as `REACT_APP_MELISSA_EXPRESS_ENTRY_API_KEY` —
readable by anyone with devtools and chargeable against the account's quota. The
key is now server-side only and is never echoed in a response or a log.

| Variable | Default | Purpose |
|---|---|---|
| `MELISSA_EXPRESS_ENTRY_API_URL` | — | Express Entry endpoint. Required. |
| `MELISSA_EXPRESS_ENTRY_API_KEY` | — | Licence key. Required. |
| `MELISSA_LOOKUP_API_URL` | empty | Second Melissa product; configured but not yet used. |
| `MELISSA_LOOKUP_API_KEY` | empty | Also read from `ADMELISSA_LOOKUP_API_KEY` (see below). |
| `MELISSA_SUPPORTED_COUNTRIES` | 20-country default | ISO-3 codes accepted. Blank list = no restriction. |
| `MELISSA_TIMEOUT_MS` | `8000` | Upstream timeout. |
| `MELISSA_CACHE_TTL_MS` | `86400000` | How long a successful lookup is reused. |
| `MELISSA_CACHE_MAX_ENTRIES` | `5000` | LRU bound on the cache. |
| `MELISSA_RATE_LIMIT_PER_MINUTE` | `60` | Lookups per caller per minute; `0` disables. |

**The lookup key is currently named `ADMELISSA_LOOKUP_API_KEY`**, which looks like
a typo for `MELISSA_LOOKUP_API_KEY`. Both names are read, preferring the
correctly-spelled one, so it can be renamed without a code change.

### Response

```json
{
  "success": true,
  "code": "OK",
  "message": "Postcode address list for 'SW1A 1AA' was retrieved successfully, please select from the list",
  "country": "GBR",
  "postcode": "SW1A 1AA",
  "count": 2,
  "cached": false,
  "results": [ /* Melissa's own records, unchanged */ ],
  "addresses": [
    { "addressLine1": "10 Downing Street", "addressLine2": "", "city": "London",
      "state": "Greater London", "postcode": "SW1A 2AA",
      "country": "United Kingdom", "addressKey": "..." }
  ]
}
```

`results` is Melissa's array verbatim, so existing frontend rendering works with
no changes — the migration is just swapping the URL. `addresses` is a flattened
equivalent for new code; Melissa's field names vary by product and country
dataset, so each value is read from a list of candidate keys and both nested
(`{Address:{...}}`) and flat record shapes are handled.

### Failure codes

| HTTP | `code` | When |
|---|---|---|
| 400 | `BAD_REQUEST` | Missing/invalid country or postcode |
| 400 | `UNSUPPORTED_COUNTRY` | Country not in the supported list |
| 404 | `NOT_FOUND` | No addresses, or Melissa returned an `ErrorString` |
| 429 | `RATE_LIMITED` | Caller exceeded the per-minute limit |
| 502 | `UPSTREAM` | Melissa unreachable, timed out, or returned unparseable data |
| 503 | `NOT_CONFIGURED` | URL or key missing on the server |

`GET /v1/address/postcode/countries` returns the accepted country codes, so the
frontend can drop its hard-coded `postCodeCountries` list. **Align
`MELISSA_SUPPORTED_COUNTRIES` with whatever that list contained** — the built-in
default is a reasonable guess, not a copy of it.

`GET /v1/address/health` reports configuration, cache and limit state without
spending a lookup.

### Cost and safety notes

- Melissa bills per lookup, so successful responses are cached for 24h keyed on
  country plus the postcode with spaces and case normalised — `SW1A 1AA` and
  `sw1a1aa` share one slot.
- Callers are rate limited per minute so a compromised token cannot drain the
  quota.
- Postcodes are validated against `[A-Za-z0-9][A-Za-z0-9 -]{0,11}` before the
  outbound call, so a value like `AB1&id=stolen` cannot tamper with the upstream
  query string.
- The request carries an explicit timeout; without one a hung upstream would
  hold the connection until the client or proxy gave up.

---

## Observability

Four independent integrations: **Sentry** (errors), **New Relic** (APM), **AWS
X-Ray** (tracing), and **CloudWatch EMF** (metrics). Structured JSON logging and
the health probes are always on and need no configuration.

Every integration is **off unless configured, and cannot break the service**. A
missing variable, a bad credential, or an unreachable collector degrades to
"that feature is disabled", logged once at startup. This is verified — the
service boots and serves `/health/live` with an entirely empty environment, and
with each of a garbage Sentry DSN, a bogus New Relic key, X-Ray with no daemon,
and metrics with no agent.

At startup the service logs exactly what is live:

```json
{"event":"OBSERVABILITY_READY","newRelic":false,"sentry":true,"xray":false,"metrics":false}
```

### Health probes

| Endpoint | Purpose | Checks |
|---|---|---|
| `GET /health/live` | Liveness — the ECS container health check | Process is responding |
| `GET /health/ready` | Readiness — the ALB target group | MongoDB `readyState === 1`; minimal body |
| `GET /health` | Detailed status for humans | Same checks, full body with version and memory |
| `GET /v1/health`, `/v1/health/live`, `/v1/health/ready` | Aliases | Same handlers, for checkers that expect a `/v1` prefix |

Both are unauthenticated by design: neither ECS nor the ALB can present a token.
Liveness stays cheap so a slow database never triggers a container restart;
readiness fails on a lost database so the load balancer stops routing while the
task stays up.

The container health check reads `PORT` (defaulting to 8484) rather than a
hardcoded 3000, and allows a 60s start period — the service takes roughly 14s to
bind locally, and longer on a cold task.

### Sentry — error reporting

| Variable | Default | Purpose |
|---|---|---|
| `SENTRY_DSN` | empty | Project DSN. **Empty disables Sentry entirely.** |
| `SENTRY_ENVIRONMENT` | `NODE_ENV` | Environment label in the Sentry UI. |
| `SENTRY_RELEASE` | `APP_VERSION` | Release for regression tracking. |
| `SENTRY_SAMPLE_RATE` | `1` | Fraction of errors sent. Lower only to protect quota. |

Only **5xx** responses are reported, from the central error handler. A 4xx is the
caller getting it wrong; sending those would bury real faults and burn quota.

Pinned to Sentry v7 deliberately. v8+ instruments HTTP through OpenTelemetry,
which would be a third agent patching the `http` module alongside New Relic and
X-Ray. Performance tracing is off (`tracesSampleRate: 0`) for the same reason —
New Relic covers APM.

Request bodies and headers are never attached, and `beforeSend` additionally
strips `authorization`, `cookie` and `x-api-key`, and redacts `password`,
`token`, `refreshToken` and `accessToken` if they ever appear.

### New Relic — APM

| Variable | Default | Purpose |
|---|---|---|
| `NEW_RELIC_LICENSE_KEY` | empty | **Empty makes the agent inert.** |
| `NEW_RELIC_APP_NAME` | `remittance-auth-and-directory` | Service name in APM. |
| `NEW_RELIC_HOST` | US collector | **EU accounts must set `collector.eu01.nr-data.net`.** |
| `NEW_RELIC_LOG_LEVEL` | `info` | Agent log verbosity. |
| `NEW_RELIC_LOG_FORWARDING` | `false` | `true` also ships app logs to New Relic. |

Configured by `newrelic.js` **at the repository root** — the only place the agent
looks. A copy previously sat at `src/utils/newrelic.ts`, where the agent could
never find it; it has been removed.

The agent is required at the very top of `src/index.ts`, before any import. It
instruments express, mongodb and http as they load, so requiring it later
produces an agent that silently reports nothing.

Log forwarding defaults to **off**: stdout already goes to CloudWatch, and
enabling both duplicates every line and its cost. Agent logs go to stdout rather
than a file, which a non-root container cannot write.

Credentials are excluded from traces: `authorization`, `cookie`, `x-api-key`,
and the `password` / `token` / `refreshToken` parameters.

### AWS X-Ray — distributed tracing

| Variable | Default | Purpose |
|---|---|---|
| `XRAY_ENABLED` | `false` | Must be `true` to enable. |
| `XRAY_DAEMON_ADDRESS` | `127.0.0.1:2000` | X-Ray daemon / sidecar address. |

Opt-in rather than credential-driven, for two reasons. It monkey-patches global
`http`/`https`, so every outbound call in the process — including S3 uploads —
then runs through it. And it needs a daemon sidecar; without one, every traced
call emits a failed UDP send. Requires the `AWSXRayDaemonWriteAccess` policy on
the task role.

### CloudWatch metrics (EMF)

| Variable | Default | Purpose |
|---|---|---|
| `METRICS_ENABLED` | `false` | Must be `true` to enable. |
| `AWS_EMF_ENVIRONMENT` | auto-detect | **Set to `Local` to emit EMF to stdout.** |

Emits `RequestCount`, `ResponseTime`, `ErrorCount` and `SuccessCount` under the
`MyApp/Microservices` namespace.

By default the library posts to a CloudWatch agent on TCP `25888`. With no agent
running, every request logs `Metrics error: connect ECONNREFUSED 0.0.0.0:25888`
— harmless, since the failure is caught and the response is unaffected, but
noisy. **Set `AWS_EMF_ENVIRONMENT=Local`** to print EMF to stdout instead;
CloudWatch Logs parses it into metrics with no sidecar at all. That is the
recommended setting for ECS.

### Logging and trace correlation

Structured JSON via pino, always on, stamped with service, version, environment,
`containerId` and `taskArn`. Every request is assigned a trace id and a request
id, returned as `x-trace-id` / `x-request-id` and propagated from an inbound
`x-trace-id` when present, so a trace can be followed across services. Held in
`AsyncLocalStorage`, so every log line for a request is correlated without
threading a context object through call sites.

### Graceful shutdown

ECS sends `SIGTERM` before stopping a task — on a deploy, a scale-in, or a
Fargate Spot reclamation. The handler stops accepting connections, drains
in-flight requests, closes MongoDB, and flushes Sentry and New Relic before
exit, so errors from the last seconds of a task's life are not lost. Unhandled
rejections and uncaught exceptions are logged (and reported) rather than
vanishing.

### General

| Variable | Default | Purpose |
|---|---|---|
| `SERVICE_NAME` | `remittance-auth-and-directory` | Name in logs, metrics and traces. |
| `APP_VERSION` | `unknown` | Set by the Dockerfile `APP_VERSION` build arg. |
| `ECS_TASK_ARN` | empty | Injected by ECS; used only as a log field. |

### Fail-soft: crash-on-missing-config

Three module-level client constructions previously threw during import, taking
the **whole service** down over one optional credential:

| Where | Missing variable | Was | Now |
|---|---|---|---|
| `components/Auth/service.ts` | `RESEND_API_KEY` | Boot crash | Password-reset email not sent; logged |
| `crons/birthday.ts` | `RESEND_API_KEY` | Boot crash | Birthday send fails and is logged as FAILED |
| `components/Auth/index.ts` | `SECRET_V2` | Boot crash | Login succeeds; `authToken` field omitted |

All three now build their client on first use. The access token itself never
depended on `SECRET_V2` — only the extra encrypted `authToken` field did.

---

## Log redaction

Credentials are stripped from output before it is written. This was added after
Node's `DEP0170` deprecation warning printed the full Mongo connection string —
username and password — into the PM2 log on every boot.

| Variable | Default | Purpose |
|---|---|---|
| `LOG_REDACTION_ENABLED` | `true` | `false` disables the `console` wrapper only. The warning interceptor and logger scrubbing always run. |

That leak is the reason redaction is applied in five places rather than one: the
warning went straight to stderr from Node's own handler, so **no logger-level
configuration could have caught it**.

| Path | How |
|---|---|
| Node warnings | Node's default printer is removed and replaced with a scrubbing one ([redact.ts](src/utils/redact.ts)) |
| `console.*` | Wrapped at the entrypoint, before anything can emit |
| winston (`Logger`) | A `redactFormat` in the format chain |
| pino (structured logs) | A `logMethod` hook — pino's own `redact` only matches object paths, not text inside a message |
| Sentry | `beforeSend` runs the whole event through `redactDeep`, covering exception messages, stack frames and breadcrumbs |

Two complementary strategies. **Pattern-based** catches URI userinfo, sensitive
query parameters (`id`, `key`, `token`, `password`, `dsn`, …), `Bearer` tokens,
JWTs and `rccg_` API keys — including secrets this process has never seen.
**Value-based** replaces exact matches of the secrets actually in the
environment (`SECRET`, `REFRESH_TOKEN_SECRET`, `MONGODB_URI`'s password, the
Melissa keys, `NEW_RELIC_LICENSE_KEY`, `SENTRY_DSN`, `RESEND_API_KEY`,
`TERMII_API_KEY`, the AWS keys), which does not depend on recognising the
surrounding syntax.

Diagnostic value is preserved deliberately — only the password is removed from a
connection string, not the whole URI:

```
failed to connect to mongodb://auth_dir_user28:[redacted]@remittancecluster0-shard-00-02.tj34xo.mongodb.net:27017/remittanceAuthDirGlobal
```

Host, port, database and username all survive, so the line is still useful.

### This is defence in depth, not a substitute for fixing the cause

The `DEP0170` warning fires because a multi-host `mongodb://` seed list is not a
valid WHATWG URL. **Node will throw on it in a future release**, so switch the
connection string to the SRV form:

```
mongodb+srv://USER:PASS@remittancecluster0.tj34xo.mongodb.net/remittanceAuthDirGlobal?authSource=admin
```

One host, no `replicaSet`/`ssl` parameters needed, and the warning disappears.

Any credential already written to a log must be treated as compromised —
redaction going forward does not unwrite history. Rotate it.

---

## Observability switches

Four independent signals. Each is controlled on its own — none depends on
another, and none requires ECS.

| Variable | Default | Needs a sidecar? |
|---|---|---|
| `SENTRY_DSN` | — | No. Plain HTTPS. |
| `SENTRY_ENABLED` | inferred from `SENTRY_DSN` | — |
| `NEW_RELIC_LICENSE_KEY` | — | No. Plain HTTPS. |
| `NEW_RELIC_ENABLED` | inferred from the licence key | — |
| `XRAY_ENABLED` | `false` | **Yes** — the X-Ray daemon over UDP |
| `METRICS_ENABLED` | inferred from `AWS_EMF_ENVIRONMENT=Local` | **Yes**, unless printing to stdout |

Each switch is tri-state: leave it unset to infer from whether the credential is
present, `true` to force on (which surfaces the real failure instead of skipping
silently), `false` to force off without deleting the credential.

**Sentry and New Relic need no sidecar.** They talk to their backends over
HTTPS, so they behave identically on a laptop, on the EC2/PM2 dev box and on ECS.
Setting the credentials is all that is required.

**Testing metrics off ECS.** `aws-embedded-metrics` will print the EMF JSON to
stdout instead of shipping it to the CloudWatch agent:

```bash
METRICS_ENABLED=true AWS_EMF_ENVIRONMENT=Local npm start
```

Requests then emit lines containing `"_aws":{...,"CloudWatchMetrics":[...]}`,
which is enough to confirm the metric names and dimensions are right before
anything is deployed. Set `AWS_EMF_NAMESPACE` too — the library otherwise
defaults to the placeholder `MyApp/Microservices`.

**X-Ray is the only one that genuinely needs infrastructure.** Enabling it with
no daemon reachable produces a stream of failed UDP sends, which is why no
credential can imply it.

### Why they were silently off

`dotenv.config()` used to run only when `@/config/env` was first imported, which
happened *after* observability initialised. Both the New Relic gate and Sentry's
DSN check therefore read an unpopulated `process.env` and concluded "not
configured" — with both credentials sitting in `.env`. The flags were also
captured in a module-level constant, so they stayed `false` even once the values
loaded. `.env` is now loaded as the first statement in the entrypoint, and the
flags are resolved per call.

The boot log now reports *why* each disabled signal is disabled, so this class of
problem is visible rather than inferred:

```json
{"event":"OBSERVABILITY_READY","disabledReasons":{"xray":"set XRAY_ENABLED=true (needs the X-Ray daemon)"},
 "newRelic":true,"sentry":true,"xray":false,"metrics":true}
```

---

## Deployment checklist

- [ ] `SECRET` and `REFRESH_TOKEN_SECRET` set, and different from each other
- [ ] `TOKEN_EXPIRE_TIME_SEC` < `REFRESH_TOKEN_EXPIRE_TIME_SEC`
- [ ] Legacy `TOKEN_EXPIRE_TIME_MSEC` renamed
- [ ] `MONGODB_URI` includes the database name
- [ ] `CORS_DISABLED` is not `true` outside local development
- [ ] New variables added to `../.env.template` on the runner, not just locally
- [ ] Boot log checked for `[config]` warnings
- [ ] `OBSERVABILITY_READY` shows the signals you expect; `disabledReasons` explains any that are off
- [ ] Boot log grepped for credentials (`grep` the DB password) — expect no hits
- [ ] `AWS_REGION` and `S3_BUCKET` set, and `GET /v1/uploads/health` returns `ok: true`
- [ ] Melissa URL/key set, `GET /v1/address/health` returns `ok: true`, and `MELISSA_SUPPORTED_COUNTRIES` matches the frontend's old list

## Refresh-token reuse grace

Refresh tokens rotate on every use and the retired one is revoked immediately,
so presenting it again is normally replay and costs the whole session
(`revokeSession(REUSE_DETECTED)`).

With no tolerance at all that also punishes two things the client cannot avoid:

- the rotation succeeded but its response never arrived — dropped connection,
  request timeout, the tab closed — so the client still holds the old token and
  has no way to know the replacement exists;
- the page reloaded mid-refresh, so the replacement was never persisted.

Both are ordinary network behaviour, and both logged the user out of every tab.

`REFRESH_REUSE_GRACE_MS` accepts the retired token for a short period after
rotation and reissues, rather than revoking. It is deliberately narrow — all
three must hold:

1. the token was retired by `ROTATED`, not by a logout, password change or
   account action, which stay final;
2. it was retired within the window;
3. its replacement is **itself still live**.

Condition 3 is what keeps this safe. Once the chain has moved past the
replacement, the legitimate client clearly received it, so anything from
further back is replay and is still treated as theft. An attacker would have to
replay inside the window *and* get there before the legitimate client used the
replacement.

On a honoured retry the replacement the client never received is stood down in
favour of the pair being returned, so the session still only ever has one live
refresh token.

This matters more the shorter the access-token lifetime: at the Phase 2 target
of 900 s, refreshes — and therefore rotations — happen roughly 960x more often
than at the currently deployed 10 days, and the race surface scales with them.
Land this before that change, not after.
