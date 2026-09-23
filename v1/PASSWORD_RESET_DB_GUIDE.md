# Password reset — database and deploy guide

What has to happen on the database for the scoped password-reset work, in the
order it has to happen, and how to tell afterwards whether it worked.

This covers the two commits that have shipped so far and the role-sensitivity
step that gates the reset feature. It does not cover the reset endpoints
themselves; those are not built yet.

---


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
