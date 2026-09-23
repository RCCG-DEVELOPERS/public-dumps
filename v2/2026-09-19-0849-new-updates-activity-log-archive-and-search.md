# 2026-09-19 08:49 — Archived activity history, approvals inbox, indexed search

**Commits:** `acb781e`, `fa55ed8`, `0fd211e`, `b797bd0`, `89a6510`, `da5ee28`, `e592971`

---

## 1. Archived activity history

`activityLogs` reached 3.2M documents / 3.3 GB at ~30,500 inserts a day. Rows
older than the hot window move to an Atlas Online Archive and are queried
through a separate endpoint.

### `POST /v1/activityLogs/archive/search`

**Guard:** super-admin.

```http
POST /v1/activityLogs/archive/search
Content-Type: application/json

{
  "from": "2026-06-01",
  "to": "2026-06-30",
  "activity": "LOGIN",
  "username": "jadesola.o",
  "limit": 100
}
```

`from` and `to` are **mandatory** and capped at 31 days. Archive reads are
billed per GB scanned and there are no indexes on archived data — only date
partition pruning — so an unbounded query is refused rather than run.

**Filterable fields, exact match only, no regex:**

```
sessionId  userId  username  activity  module  status
parishId   affectedId  affectedUsername
area  zone  province  region  subContinent  continent
```

**Errors:** `DATES_REQUIRED`, `RANGE_TOO_WIDE`, `UNKNOWN_FILTER`, `BAD_DATE`,
and `503` when the archive is not configured on that environment.

### The live endpoints

| Method | Path | Notes |
|---|---|---|
| GET | `/v1/activityLogs` | paged listing |
| POST | `/v1/activityLogs/search` | filtered search; counts capped at 10,000 |
| GET | `/v1/activityLogs/session/:sessionId` | one session, capped at 500 rows |
| GET | `/v1/activityLogs/user/:userId` | one person's history |
| GET | `/v1/activityLogs/:id` | one row |
| POST | `/v1/activityLogs/ingest` | external ingest, **API key**, answers 202 |

> **Standing rule: `activityLogs` must never acquire a TTL.** It is the business
> record of deliberate actions. The archive is how it stays affordable. The
> privileged audit trail is a separate collection precisely so this one takes on
> no more write load or indexes.

Indexes are built out of band by `scripts/createActivityLogIndexes.ts`, using
**mongoose's own default index names** — the same key under a different name is
refused with `IndexOptionsConflict` (code 85), reported on a connection event
nothing in this repo listens to.

## 2. Approvals inbox

`GET /v1/approvals` gained the fields an inbox needs: `treated`, `canDecide` and
a display label per row, plus `decidable=true` to narrow to what *you* can act
on, and summary counts.

Who may decide what:

| Request type | Decided by |
|---|---|
| `OFFICER_TRANSFER`, `OFFICER_PROMOTION` | any administrator whose unit **contains** the office |
| `USER_TRANSFER` | a province admin at **either** end |
| `UNIT_TRANSFER` | the **receiving** province, or a region above it |

Nobody but a super-admin may decide a request they raised themselves.

## 3. Indexed search endpoints

`/query`, `/queryAndFilter`, `/queryFlex` and their `WithPastorData` twins sit
beside the existing regex `/search` routes. `/query` forces equality, `/queryFlex`
forces prefix — both index-served, where the regex form scans.

Same addition on the user search.

## 4. Parish country codes

`parishDirectory.country` was free text: alpha-2 on most rows, full names on
others, empty on some. Comparing it with `===` is what produced *"login from NL
is not permitted, your parish is registered in Netherlands"*.

A derived `countryCode` now sits beside it, ISO 3166-1 alpha-2, resolved from the
record — country field, address, state, hierarchy names, dialling code or
currency, in that order of confidence. It never guesses: ambiguous input
resolves to `""`.

`scripts/backfillParishCountryCode.ts` fills the existing rows; it is idempotent.

## 5. Split units became movable

A split unit — one whose directory rows disagree about their ancestors — used to
refuse every move. Now the **dominant** variant carries the decision and minority
variants come back as warnings, so a stray row can no longer make a unit
immovable by the administrator whose province holds nearly all of it.

`realign` keeps the strict all-pairs rule: it is the operation that decides which
half of a split wins.
