# Geofencing — API Reference

Restricting which countries a person may log in from, and granting exceptions.

- [Why this was rewritten](#why-this-was-rewritten)
- [How a decision is made](#how-a-decision-is-made)
- [Rule fields](#rule-fields)
- [Rules API](#rules-api)
- [Exemptions — granting an exception](#exemptions--granting-an-exception)
- [Use cases](#use-cases)
- [What a blocked login looks like](#what-a-blocked-login-looks-like)
- [Activity log events](#activity-log-events)
- [Errors](#errors)
- [Known gaps](#known-gaps)
- [Frontend guidance](#frontend-guidance)
- [Deployment](#deployment)

---

## Why this was rewritten

Geofencing was enabled for the UK sub-continent. A user in the Netherlands was
refused with:

> Access denied: login from **NL** is not permitted. Your parish is registered in
> **Netherlands**.

NL and Netherlands are the same country. He was in the right place and was
refused anyway. Three independent faults produced that:

| | |
|---|---|
| **One global switch** | the settings were a singleton, so "enable it for the UK" enabled it everywhere — the 97 `LOGIN_BLOCKED` events span **nine sub-continents** |
| **The parish lookup was wrong for 98% of users** | `findOne({ parishRef: user.parishRef })`, and 55,933 of 56,931 users have no `parishRef`. Mongoose strips an `undefined` query value, so it became `findOne({})` and returned the collection's first row — a parish in the Netherlands |
| **Different formats on each side** | `geoip` returns ISO alpha-2; `parishDirectory.country` holds alpha-2 on 52,898 rows and full names on 95 |

All three are fixed. The country normalisation lives in
`src/utils/countryCodes.ts`; the parish resolution in `resolveParish`.

---

## How a decision is made

At login, in this order:

```
1.  rule = resolveForUser(user)          narrowest-wins, see below
2.  rule.enabled false                   -> ALLOW
3.  caller is super-admin                -> ALLOW   (never a total lockout)
4.  Origin header not in the rule's
    allowedClientOrigins (when set)      -> ALLOW   (rule does not apply here)
5.  loginCountry = toAlpha2(geoip)
    unresolvable                         -> ALLOW + log LOGIN_GEO_UNKNOWN
6.  rule.allowedCountries non-empty
      -> ALLOW iff loginCountry is in that list
    otherwise
      -> parish = resolveParish(user)
         no parish, or no usable country -> ALLOW + log LOGIN_GEO_UNKNOWN
         -> ALLOW iff sameCountry(loginCountry, parish.country)
7.  refused -> active exemption for (userId, loginCountry)?  -> ALLOW
8.  otherwise                            -> 403
```

### Resolution — narrowest wins

A person is governed by the rule **closest to them**. The resolver reads their
own scope columns and walks from parish upward, taking the first rule that
exists:

```
parish 211617 -> area ECANLD0010 -> zone ECZNLD0004 -> province ECPNLD0002
  -> region ECREM0001 -> sub-continent ECS0001 -> continent EC0001 -> global
```

One `$or` query, not seven round trips per login.

**The whole document wins, `enabled` included.** A region can therefore set
`enabled: false` and **opt itself out** of a rule its sub-continent switched on.
That is the point — it is what stops one switch doing what it just did.

A level the user has no code for is never a candidate, so a rule for a unit they
do not belong to can never reach them.

### `allowedCountries` replaces the parish rule

| | |
|---|---|
| **non-empty** | ONLY those countries may log in. The parish's own country is irrelevant |
| **empty** | falls back to "your login country must match your own parish's country" |

### Country codes are checked, not trusted

Everything normalises through ISO 3166 (`country-state-city`, already a
dependency — no alias table to drift):

- `sameCountry("NL", "Netherlands")` → `true`
- `toAlpha2("ROMANIA")` → `"RO"`; `toAlpha2("Romania")` → `"RO"`
- **Two unknowns never match** — `sameCountry("XX", "XX")` is `false`, so an
  unparseable country can never equal another one

A rule listing `"UK"` instead of `"GB"` is **refused by name on write**. It would
otherwise permit nobody, and the first sign would be a locked-out sub-continent.

### Everything unknown fails OPEN

A geofence that cannot tell where someone is must not guess. These allow the
login and write `LOGIN_GEO_UNKNOWN`:

- `geoip` cannot place the IP
- the person's parish cannot be resolved
- the parish has no usable country

`resolveParish` keys on `parishCode`, consults `parishRef` only when it is a
non-empty string, and **can never issue a query that matches everything**.

---

## Rule fields

| Field | Type | Notes |
|---|---|---|
| `levelType` | string | `global`, `continent`, `sub-continent`, `region`, `province`, `zone`, `area`, `parish` |
| `scopeCode` | string | the unit's code — `EC09SC01`. `""` when global |
| `scopeName` | string | denormalised for display; filled from the directory on write |
| `enabled` | boolean | default `false` |
| `allowedCountries` | string[] | ISO alpha-2, uppercased and validated on write |
| `allowedClientOrigins` | string[] | the rule applies only to requests whose `Origin` is listed. Empty = every origin |
| `note` | string | free text, max 500 |
| `updatedBy` | string | set from the caller |

Unique index on `{ levelType, scopeCode }` — one rule per unit.

**Authority:** standing **at or above** the unit, or super-admin. An `sco` of
ECS0001 manages ECS0001 and any region beneath it, never a sibling
sub-continent. Decided from the unit's **own chain read from the database**,
never from anything the caller asserted. The global rule is super-admin only.

---

## Rules API

Mount: `/v1/geofencing/settings`, bearer token.

| Route | Auth |
|---|---|
| `GET /` | any authenticated caller |
| `PUT /` | **super-admin** (the global rule) |
| `GET /rules` | **super-admin** |
| `GET /effective` | yourself; naming another user is super-admin |
| `PUT /rules` | standing at or above that unit, or super-admin |
| `DELETE /rules` | same |
| `GET /countries` | any authenticated caller |
| `POST /admin/repair-country-codes` | **super-admin** |

Exemptions, mounted at `/v1/geofencing/exemptions`:

| Route | Auth |
|---|---|
| `GET /`, `GET /:id`, `GET /user/:userId`, `POST /search` | **sub-continent, continent, or super-admin** |
| `POST /`, `PATCH /:id`, `DELETE /:id` | **sub-continent, continent, or super-admin** |

### GET /v1/geofencing/settings

The global rule. Auto-created (disabled) if it does not exist.

```json
{
  "id": "6aa2d324ec1c88b43d0e46b6",
  "levelType": "global",
  "scopeCode": "",
  "enabled": false,
  "allowedCountries": [],
  "allowedClientOrigins": [],
  "note": "",
  "updatedBy": "system"
}
```

### PUT /v1/geofencing/settings

The global rule only. **Super-admin** — it governs every user no narrower rule
reaches.

```json
{
  "enabled": false,
  "allowedCountries": [],
  "allowedClientOrigins": ["https://portal.rccg.org"],
  "note": "off pending review"
}
```

Returns the stored rule. `levelType` and `scopeCode` are **not** accepted here —
scoped rules go to `PUT /rules`, so there are never two ways to write one
document.

### GET /v1/geofencing/settings/rules

Every rule at every level. Super-admin, because the list spans units the caller
may have no standing over.

```json
{
  "totalCount": 2,
  "records": [
    { "levelType": "global", "scopeCode": "", "enabled": false, "allowedCountries": [] },
    { "levelType": "sub-continent", "scopeCode": "EC09SC01", "scopeName": "EUROPE UK",
      "enabled": true, "allowedCountries": ["GB", "IE"],
      "allowedClientOrigins": ["https://portal.rccg.org"],
      "note": "UK office policy", "updatedBy": "adenijio@rccg.co.uk" }
  ]
}
```

### PUT /v1/geofencing/settings/rules

Create or update one unit's rule. Upserts on `{levelType, scopeCode}`.

```jsonc
{
  "levelType": "sub-continent",
  "scopeCode": "EC09SC01",
  "enabled": true,
  "allowedCountries": ["GB", "IE"],
  "allowedClientOrigins": ["https://portal.rccg.org"],
  "note": "UK office policy, council minute 2026/14"
}
```

Only the fields you send are changed; omitted fields keep their stored values.

**Refusals**

| Status | Cause |
|---|---|
| `400` | an unknown country code — **named** in the message |
| `400` | `scopeCode` does not resolve to a real unit |
| `400` | `levelType` is not a level |
| `400` | a non-global rule with no `scopeCode` |
| `400` | `enabled: true` with an empty `allowedClientOrigins` |
| `403` | no standing at or above that unit |

### DELETE /v1/geofencing/settings/rules

```
DELETE /v1/geofencing/settings/rules?levelType=sub-continent&scopeCode=EC09SC01
```

```json
{ "removed": 1, "levelType": "sub-continent", "scopeCode": "EC09SC01" }
```

Anyone that rule governed falls back to the next rule up. The **global rule
cannot be deleted** — disable it instead; deleting it would leave the resolver
with no last resort.

### GET /v1/geofencing/settings/effective

**Ask this before enabling anything.** It answers "would this block me" and "why
am I blocked" in one call. Without `userId` it answers for the caller.

```
GET /v1/geofencing/settings/effective?userId=65368e03b596ba6fd9f71eb6
```

```jsonc
{
  "user": {
    "id": "65368e03b596ba6fd9f71eb6",
    "username": "vapschiedam@rccgeurope.org",
    "parishCode": "211617",
    "subContinent": "ECS0001"
  },
  "parish": {
    "parishCode": "211617",
    "parishName": "Victory Assembly Parish",
    "countryStored": "Netherlands",   // what the directory holds
    "countryResolved": "NL"           // what it is compared as
  },
  "appliedRule": {
    "levelType": "global", "scopeCode": "",
    "enabled": false, "allowedCountries": [], "allowedClientOrigins": []
  },
  "candidates": [
    { "levelType": "parish",        "scopeCode": "211617",     "rank": 7, "matched": false },
    { "levelType": "area",          "scopeCode": "ECANLD0010", "rank": 6, "matched": false },
    { "levelType": "zone",          "scopeCode": "ECZNLD0004", "rank": 5, "matched": false },
    { "levelType": "province",      "scopeCode": "ECPNLD0002", "rank": 4, "matched": false },
    { "levelType": "region",        "scopeCode": "ECREM0001",  "rank": 3, "matched": false },
    { "levelType": "sub-continent", "scopeCode": "ECS0001",    "rank": 2, "matched": false },
    { "levelType": "continent",     "scopeCode": "EC0001",     "rank": 1, "matched": false },
    { "levelType": "global",        "scopeCode": "",           "rank": 0, "matched": true  }
  ]
}
```

`countryStored` beside `countryResolved` is deliberate: it shows the exact pair
whose mismatch caused the original refusal.

### GET /v1/geofencing/settings/countries

```json
{
  "totalCount": 250,
  "records": [
    { "isoCode": "AF", "name": "Afghanistan" },
    { "isoCode": "AL", "name": "Albania" }
  ]
}
```

Sorted by name. Use it to drive a picker so nobody types `UK`.

### POST /v1/geofencing/settings/admin/repair-country-codes

Normalises stored values to alpha-2 — 95 parish rows and 32 users hold names.
Defaults to a **dry run**; only the literal `dryRun=false` applies it.

```jsonc
{
  "dryRun": true,
  "collections": [
    { "collection": "parishDirectory", "scanned": 52993, "wouldChange": 95, "changed": 0,
      "samples": [{ "id": "211617", "from": "Netherlands", "to": "NL" }],
      "unresolved": [], "unresolvedCount": 0 },
    { "collection": "users", "scanned": 56749, "wouldChange": 32, "changed": 0,
      "samples": [{ "id": "someone@rccg.org", "from": "Germany", "to": "DE" }],
      "unresolved": [], "unresolvedCount": 0 }
  ]
}
```

**Not a prerequisite.** The geofence normalises at read time, so enforcement is
already correct. This is for every *other* reader — including the password-reset
phone-code lookup, which called `getCountryByCode(user.country).phonecode`
unguarded and **threw** for the 32 accounts whose country is a name.

Anything it cannot resolve is reported, never guessed at and never blanked.

---

## Exemptions — granting an exception

An exemption lets **one named person** log in from **one named country**, until a
date. It is the answer to "our regional pastor is travelling to Kenya for three
weeks" without weakening the rule for anyone else.

Mount: `/v1/geofencing/exemptions`, bearer token.

### Who may manage one

**Sub-continent officers, continent officers, and super-admins.** Nobody else —
the whole mount is gated, reads included.

The gate is by **level**, not by a list of role slugs: `standingOf` reads each of
the caller's roles, maps its `level_type`, and records their own code at that
level. So it admits the sub-continent roles — `sco`, `asco`, `sub-cont-admin`,
`sub-cont-accountant`, `sub-cont-ict` — and all four continent ones — `co`,
`aco`, `cont-admin`, `cont-accountant` — without naming any of them, and will
admit a role added at those levels later.

**`training-manager` is excluded.** It sits at sub-continent level but carries no
administrative authority: its remit is training, and it may create and manage the
training managers under it, nothing else. `standingOf` never records a unit for
it, so it reaches none of the administrative surfaces — geofencing rules or
exemptions, principal-officer appointment, hierarchy moves, headquarters
assignment, or approvals. The list is
`NON_ADMINISTRATIVE_ROLES` and is set from the environment:

```bash
NON_ADMINISTRATIVE_ROLES=training-manager
```

Unset, or empty, keeps that default — an empty line is likelier to be an accident
than a deliberate widening. Clearing it must be said out loud as
`NON_ADMINISTRATIVE_ROLES=none`. Inline `#` comments are stripped, because
dotenv 4 does not strip them and an uncleaned slug would match no role and fail
open.

An officer whose profile is **missing its code** at that level is refused:
`standing.units` only carries a level when a code exists, and without a unit
there is nothing to act in however senior the role sounds.

A refusal is `403` and is logged as `GEOFENCING_EXEMPTION_DENIED`:

```json
{
  "message": "Geofencing exemptions are managed at sub-continent or continent level. An exemption lets one person bypass the geofence entirely, so it is not something a parish or province role can grant. Ask a sub-continent or continent officer, or a super-admin."
}
```

> **Why this level and not lower.** An exemption is a hole punched through the
> geofence for one named person. A province or parish role that could grant one
> could grant it to itself, which would make the rule above it decorative.

### How it is consulted

Only after a rule has already refused the login. The lookup is exact:

```js
{ userId, allowedCountry, status: "ACTIVE", expiresAt: { $gt: new Date() } }
```

All four must hold. In particular:

- **`allowedCountry` must be the ISO alpha-2 code**, because it is compared
  against `geoip`'s alpha-2. An exemption recorded as `"Netherlands"` or `"UK"`
  matches nothing and silently fails to exempt anyone — which is why the field is
  now validated. It had **no validation at all** before.
- **`expiresAt` is required**, so there is no such thing as a permanent
  exemption. It is *not* validated as being in the future — a date already past
  is accepted on write and then simply never matches, which is silent. Check it
  in the UI.
- **`parishId` is stored but never consulted.** It is a note, not a scope.
- An exemption is per **country**, not per rule. Someone needing two countries
  needs two exemptions.

### Fields

| Field | Type | Notes |
|---|---|---|
| `userId` | string | **required** — the user's `_id` |
| `allowedCountry` | string | **required** — ISO alpha-2, uppercased on save |
| `expiresAt` | date | **required** |
| `username` | string | for display in listings and logs |
| `parishId` | string | recorded, not used in the lookup |
| `reason` | string | why it was granted |
| `createdBy` | string | who granted it |
| `status` | string | defaults to `ACTIVE`; anything else stops it matching |

### POST /v1/geofencing/exemptions — grant one

```jsonc
{
  "userId": "65368e03b596ba6fd9f71eb6",
  "username": "vapschiedam@rccgeurope.org",
  "allowedCountry": "KE",
  "expiresAt": "2026-10-31T23:59:59.000Z",
  "reason": "Mission trip, approved by the regional office",
  "createdBy": "adenijio@rccg.co.uk"
}
```

`201` with the stored record:

```json
{
  "id": "6aa3f1c40389f6c7a260e701",
  "userId": "65368e03b596ba6fd9f71eb6",
  "username": "vapschiedam@rccgeurope.org",
  "allowedCountry": "KE",
  "expiresAt": "2026-10-31T23:59:59.000Z",
  "reason": "Mission trip, approved by the regional office",
  "status": "ACTIVE",
  "createdBy": "adenijio@rccg.co.uk"
}
```

`400` when `allowedCountry` is not a real ISO code — `"Kenya"`, `"UK"` and
`"XX"` are all refused, by name.

### GET /v1/geofencing/exemptions — list

```
GET /v1/geofencing/exemptions?pageNo=1&pageSize=20
```

```json
{ "totalCount": 3, "records": [ /* … */ ], "pageNo": 1, "pageSize": 20 }
```

### GET /v1/geofencing/exemptions/user/:userId — one person's exemptions

The quickest way to answer "why can this person log in from there?"

```json
{ "records": [ /* … */ ], "totalCount": 1 }
```

### GET /v1/geofencing/exemptions/:id

Returns the single record.

### PATCH /v1/geofencing/exemptions/:id — amend or revoke

```jsonc
{ "expiresAt": "2026-11-30T23:59:59.000Z" }     // extend
{ "status": "REVOKED" }                          // revoke without deleting
{ "allowedCountry": "TZ" }                       // re-point; validated
```

Returns the string `"Updated successfully"`, or `400 "Failed to update"` when
nothing matched.

> **Revoking is preferable to deleting.** Setting `status` to anything other
> than `ACTIVE` stops it matching immediately while leaving the record, its
> reason and its grantor in place for audit.

### DELETE /v1/geofencing/exemptions/:id

Removes it permanently and returns the deleted record.

### POST /v1/geofencing/exemptions/search

The repository's generic search shape:

```jsonc
{
  "orAnd": "and",
  "params": [
    { "columnName": "allowedCountry", "columnValue": "KE" },
    { "columnName": "status", "columnValue": "ACTIVE" }
  ]
}
```

`?pageNo=` and `?pageSize=` apply. Returns `{ totalCount, records, pageNo, pageSize }`
— note that search echoes `pageNo` zero-based while the plain listing returns it
one-based, an inconsistency that predates this work.

---

## Use cases

### 1. A sub-continent restricts itself, and only itself

```bash
curl -X PUT -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"levelType":"sub-continent","scopeCode":"EC09SC01","enabled":true,
       "allowedCountries":["GB","IE"],
       "allowedClientOrigins":["https://portal.rccg.org"]}' \
  "$API_HOST/v1/geofencing/settings/rules"
```

People under EC09SC01 may log in from GB or IE. Everyone else is untouched —
confirm with `/effective` against someone in a neighbouring sub-continent.

### 2. One region inside it needs different countries

```bash
-d '{"levelType":"region","scopeCode":"ECREM0001","enabled":true,
     "allowedCountries":["NL","BE"],
     "allowedClientOrigins":["https://portal.rccg.org"]}'
```

Narrowest wins, so that region follows `NL`/`BE` while the rest of the
sub-continent stays on `GB`/`IE`.

### 3. One region opts out entirely

```bash
-d '{"levelType":"region","scopeCode":"ECREM0001","enabled":false}'
```

The whole document wins, so `enabled: false` beats the sub-continent's `true`.
Nobody in that region is geofenced.

### 4. "Everyone must be in the country their parish is in"

Set `enabled: true` and leave `allowedCountries` empty. Each person is judged
against their own parish's country — no list to maintain.

### 5. A person is travelling

Grant an exemption (above) rather than widening the rule — it expires by itself,
where a widened rule stays wide. A **sub-continent or continent officer** does
this; the traveller's own province admin cannot.

### 6. Someone is blocked and nobody knows why

```bash
curl -H "Authorization: Bearer $JWT" \
  "$API_HOST/v1/geofencing/settings/effective?userId=<id>"
```

`appliedRule` names the rule, `candidates` shows what it beat, and
`countryStored` vs `countryResolved` shows the comparison itself.

### 7. Turning it off in a hurry

```bash
curl -X DELETE -H "Authorization: Bearer $JWT" \
  "$API_HOST/v1/geofencing/settings/rules?levelType=sub-continent&scopeCode=EC09SC01"
```

Or `PUT` it with `{"enabled": false}`. Super-admins are never geofenced, so a
super-admin can always get in to do this.

---

## What a blocked login looks like

`POST /auth/login` → `403`:

```json
{
  "message": "Access denied: login from Kenya is not permitted — the countries permitted for sub-continent EC09SC01 are GB, IE."
}
```

Or, on the parish-country fallback:

```json
{
  "message": "Access denied: login from Kenya is not permitted — your parish is registered in United Kingdom."
}
```

The message names the **rule that refused** and the countries it permits, so the
person can say something useful when they call the office.

---

## Activity log events

| Activity | Module | When |
|---|---|---|
| `LOGIN_BLOCKED` | `AUTH` | a login was refused; `details` names the rule and both countries |
| `LOGIN_GEO_UNKNOWN` | `AUTH` | allowed because the country could not be determined |
| `UPDATE_GEOFENCING_SETTINGS` | `GEOFENCING_SETTINGS` | a rule was written or removed; names the authorising standing |
| `REPAIR_COUNTRY_CODES` | `GEOFENCING_SETTINGS` | the repair endpoint ran |
| `CREATE_GEOFENCING_EXEMPTION` | `GEOFENCING_EXEMPTIONS` | an exemption was granted |
| `UPDATE_GEOFENCING_EXEMPTION` | `GEOFENCING_EXEMPTIONS` | amended or revoked |
| `DELETE_GEOFENCING_EXEMPTION` | `GEOFENCING_EXEMPTIONS` | deleted |
| `GEOFENCING_EXEMPTION_DENIED` | `GEOFENCING_EXEMPTIONS` | someone without sub-continent or continent standing tried to reach the mount |
| `VIEW_*` / `LIST_*` / `SEARCH_*` | both | reads |

**Watch `LOGIN_GEO_UNKNOWN` after enabling anything.** A spike means the geofence
is failing open for a population it cannot place — which is safe, but means it is
not actually protecting them.

---

## Errors

| Status | Meaning |
|---|---|
| `400` | unknown country code (named), unknown unit, bad level, enabling with no origins |
| `403` | no standing at or above the unit; or `PUT /settings` without super-admin |
| `403` | any exemption route without sub-continent or continent standing |
| `403` | on login: geofenced, with the reason in `message` |
| `404` | `/effective?userId=` names a user that does not exist |

---

## Known gaps

`PUT /v1/geofencing/settings` and `/v1/geofencing/exemptions` both previously sat
behind `isAuthenticated` alone. Both are now guarded — the global rule is
super-admin, exemptions are sub-continent and continent.

**Exemptions are not yet scoped to the granter's own unit.** A sub-continent
officer may currently grant an exemption to any user, including one in another
sub-continent. The level gate stops a parish or province account from punching a
hole for itself, which was the urgent problem; narrowing each officer to their
own people is a further step and is not implemented. Every grant records its
actor, so it is auditable in the meantime.

---

## Frontend guidance

1. **Drive the country field from `/countries`.** Never a free-text box — `"UK"`
   is the mistake that permits nobody.
2. **Call `/effective` before showing an enable toggle**, and show which rule is
   already in force. A rule inherited from above is not obvious otherwise.
3. **Show `candidates`** when explaining a block. "Your sub-continent's rule
   applies because no region or province rule exists" is the answer people want.
4. **Warn when `allowedCountries` is empty and `enabled` is true** — that is the
   parish-country mode, and it behaves very differently from an explicit list.
5. **Make `expiresAt` required and default it to something short** when granting
   an exemption. A long default is how a temporary exception becomes permanent.
6. **Surface `countryStored` vs `countryResolved`** on a parish record. A parish
   whose country is stored as a name still works, but it is worth fixing.

---

## Deployment

The unique index is created by the org init endpoint (production runs
`MONGO_AUTO_INDEX=false`):

```bash
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388?dryRun=true"
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388"
```

Optionally, then:

```bash
curl -X POST -H "Authorization: Bearer $SUPERADMIN_JWT" \
  "$API_HOST/v1/geofencing/settings/admin/repair-country-codes?dryRun=true"
```

The pre-existing settings document is **adopted in place** as the global rule on
first read — it holds whatever was configured, and creating a second one beside
it would be ambiguous. Nothing is lost and no migration step is required.

Geofencing is currently **disabled** in production (`enabled: false`, no scoped
rules).
