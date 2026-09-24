# RCCG Parish Management System API — API reference

**Generated from `docs/openapi.json` — do not edit by hand.** Regenerate with `npm run docs` after changing any endpoint. Version 1.0.

**248 endpoints** across 29 groups. 243 require a token; **5 are public**.

## Before you start

### Base URL

The examples below use a `$BASE_URL` shell variable for the scheme and host of the environment you are calling. Set it once before running any of them:

```bash
export BASE_URL=https://<api-host>
```

Every path below already includes the `/api` global prefix. The OpenAPI document stores paths *without* it, because the prefix is applied at runtime — so if you point a code generator at the raw spec, set the server base URL to `$BASE_URL/api`.

### Authentication

Verify-only. **This API does not issue tokens** — the legacy application never did either. Tokens are minted by the external central RCCG auth service and verified here with HS256 against `LAB_KEY`. There is no login endpoint.

Send the token in any one of these headers — all six are accepted, matching legacy behaviour:

```http
authtoken: <jwt>
token: <jwt>
x-auth-token: <jwt>
auth-token: <jwt>
Authorization: Bearer <jwt>
Authorization: <jwt>
```

A missing or invalid token returns **401**. Note this differs from legacy, which returned **200 with an empty result set** — its auth helper returned an object that call sites read `->parish` off, yielding `null` and silently scoping the query to nothing.

A *valid* token carrying no parish claim returns **403** on parish-scoped endpoints. It fails closed deliberately: an empty result would be indistinguishable from a parish that genuinely has no data.

### Tenancy

Every parish-scoped endpoint derives its scope from **the token**, never from a value in the URL or body. Sending another parish’s code does not widen access — it is either ignored or rejected with **403**. Four tables (`variables`, `churches`, `tasks`, `users_lists`) have no tenancy column at all and are estate-wide by design.

### Response envelopes

The port preserves the legacy shapes exactly, and they are not consistent with each other:

| Shape | Where | Notes |
|---|---|---|
| `{"data": [...]}` | most list endpoints | Laravel `ResourceCollection`. No pagination metadata |
| `{"data": [...], "links": {...}, "meta": {...}}` | `variables`, `transfers` | The only two endpoints with a **live** `paginate()`. Laravel 6 shape: `links` is a sibling of `meta`, not nested inside it |
| bare object | single-record reads | Not wrapped |
| `{"success": true, "total": n, "data": [...]}` | `department-transfers` | Hand-built envelope |
| `{"message": "..."}` | most writes | Wording differs per module and is part of the contract |

### Status codes you should expect

| Code | Meaning |
|---|---|
| 200 | Success. **Also returned on some “not found” cases** — several legacy `delete` handlers answer 200 with `"…not found: Provide the correct parameter"`. Check the message, not just the status |
| 201 | Created. Chosen by PHP truthiness of `id`, so `0`, `"0"` and `false` all create and return 201 |
| 401 | Missing or invalid token |
| 403 | Valid token, but the resource is outside your scope, or your token carries no parish claim |
| 404 | Not found — used where legacy raised an unhandled error and returned 500 |
| 409 | Duplicate key. Reports the offending **field** but never the value |
| 422 | Validation failed. Body is `{"message": "The given data was invalid.", "errors": {"field": ["..."]}}`, matching Laravel |
| 429 | Rate limited — 60 requests per minute per IP, matching the legacy `throttle:60,1` middleware |

### A caution on validation

Validation reproduces the legacy `FormRequest` rules **including their defects**, because the `errors` map is a response contract. Consequences worth knowing: several columns the handler writes are validated by nothing; `attendances` validates `chidren` while the controller reads `children`; and a **partial update is destructive** — every column the handler assigns is written, so an omitted optional field becomes `null`. Send the full record on update unless you intend to clear fields.

---

## Contents

- [Attendances](#attendances) — 10 endpoints
- [Centers](#centers) — 7 endpoints
- [Children](#children) — 6 endpoints
- [Churches](#churches) — 7 endpoints
- [Department Lists](#department-lists) — 7 endpoints
- [Department transfers](#department-transfers) — 4 endpoints
- [Departments](#departments) — 12 endpoints
- [Employments](#employments) — 8 endpoints
- [Events](#events) — 6 endpoints
- [Expenditures](#expenditures) — 7 endpoints
- [First Timers](#first-timers) — 6 endpoints
- [Followups](#followups) — 15 endpoints
- [HF Coordinator](#hf-coordinator) — 3 endpoints
- [Incomes](#incomes) — 7 endpoints
- [Members](#members) — 43 endpoints
- [Members Attendance Logs](#members-attendance-logs) — 10 endpoints
- [Members Attendance Schedules](#members-attendance-schedules) — 7 endpoints
- [RPMS Stats](#rpms-stats) — 17 endpoints
- [Sermons](#sermons) — 6 endpoints
- [Service Feedback](#service-feedback) — 2 endpoints
- [Spirituals](#spirituals) — 6 endpoints
- [Tasks](#tasks) — 6 endpoints
- [Testimonies](#testimonies) — 9 endpoints
- [Tithings](#tithings) — 6 endpoints
- [Transfers](#transfers) — 6 endpoints
- [Users Lists](#users-lists) — 6 endpoints
- [Variables](#variables) — 11 endpoints
- [Visitations](#visitations) — 6 endpoints
- [health](#health) — 2 endpoints

---

## Attendances

### `GET /api/v1/backend/attendances`

Every attendance return for the caller’s parish

A plain `{"data": […]}` `ResourceCollection` — the legacy handler called `->get()`, not `->paginate()`, so there are no `links` or `meta` blocks. Rows are ordered as MyISAM returned them (insertion order). The commented-out `centers` join at `AttendancesController:32-35`, which would have added `center_name`, is **not** ported: it was never live, and being an inner join it would also have dropped the 53 rows whose `center_code` matches no centre. `total`, `children`, `teenager`, `adult`, `altar_call` and `offering` are emitted as the **strings** their `varchar` columns hold; `men`, `women` and the first-timer counts are numbers or `null`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Attendance returns for the parish, `{"data": […]}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/attendances" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/attendancesByMonths/{year}/{month}`

Attendance returns for one month of the caller’s parish

**The legacy route was unreachable.** `routes/api.php:180` declares the path as `attendancesByMonths/{year}/{month)` — a `)` where a `}` belongs, so Laravel treated `{month)` as a literal segment — and names the controller action `AttendancesController@attendancesByMonths/{year}/{month)`, which is not a valid method name. The corrected path is served here. `month` matches **both stored spellings** of the named month: the column holds 18 distinct values for 12 months, and parish `211716` wrote both `Oct` and `October` in 2024, so the literal `where(month, …)` returned half that month. Matching is case-insensitive, as MySQL’s `utf8mb4_unicode_ci` collation was. `year` is compared exactly — every live value is a 4-digit number, so there is no case to differ on and the index stays usable.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `year` | string | **yes** | Four-digit year as stored in the `varchar` column. Live data spans 2020–2026. |
| `month` | string | **yes** | Month name. Matched case-insensitively **and across both stored spellings**, so `Oct` and `October` return the same rows. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Attendance returns for that month, `{"data": […]}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `year` or `month` is blank. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/attendancesByMonths/{year}/{month}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/bmd_registrations`

Birth, marriage and death registrations for the caller’s parish

Parish-scoped `{"data": […]}`. The legacy handler wrapped `BMD` models in an **`AttendanceCollection`** rather than a BMD one; that is behaviourally identical, because a `ResourceCollection` without its own `toArray` emits each model’s own attributes, so the rows carry `bmd_registrations` columns either way. `births`, `deaths` and `marriages` are numbers; `month` and `year` are strings.

**Responses**

| Status | Meaning |
|---|---|
| `200` | BMD rows for the parish, `{"data": […]}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/bmd_registrations" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/bmd_registrations/viewByMonths/{year}/{month}`

BMD registrations for one month of the caller’s parish

**The legacy route had two independent defects and has never returned data.** It carries the same `{month)` typo as the attendance route, and it points at `AttendancesController@viewBMDRegistrationByMonths/{year}/{month)` — a method that exists under **no** spelling, so there was not even an implementation to correct. Implemented as the BMD analogue of the attendance monthly report, which the route name (`api.viewBMDRegistrationByMonths`) and its sibling make the evident intent: the caller’s parish, filtered by year and month, with the same month-spelling tolerance.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `year` | string | **yes** | Four-digit year as stored in the `varchar` column. Live data spans 2020–2026. |
| `month` | string | **yes** | Month name. Matched case-insensitively **and across both stored spellings**, so `Oct` and `October` return the same rows. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | BMD rows for that month, `{"data": […]}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `year` or `month` is blank. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/bmd_registrations/viewByMonths/{year}/{month}" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/bmd_registrations/save`

Create or update a BMD registration

**201 when the body carries no `id`, 200 when it does** — a legacy quirk preserved verbatim. All twelve columns the legacy handler assigned are written, so an omitted one is stored as `null` and a partial save is destructive; `d1` and `d2` are never assigned, so an existing value in either survives. `parish_code` must be the caller’s own — legacy trusted the body — and an unknown `id` is a 404 rather than an insert at that primary key. The six above-parish codes are still taken from the body, as legacy required: they are denormalised labels, nothing in the API scopes on them, and enforcing them against the token would lock out any caller whose token omits a level. `births`/`deaths`/`marriages` were `required` but **not** `numeric` in the legacy rules, so a non-numeric value is still accepted and coerced on write.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `parish_code` | string | **yes** | Must equal the caller’s own parish claim. Legacy took it from the body with no check. | `"990033"` |
| `births` | object | **yes** | Births recorded. `required` only — the legacy rules declared no `numeric`, so a non-numeric value was accepted and MySQL coerced it on insert into an `int(11)`. | `23` |
| `deaths` | object | **yes** | Deaths recorded. `required` only. | `0` |
| `marriages` | object | **yes** | Marriages recorded. `required` only. | `2` |
| `month` | string | **yes** | Month name, stored verbatim. | `"Dec"` |
| `year` | string | **yes** | Four-digit year, stored as a `varchar`. | `"2025"` |
| `area_code` | string | **yes** | Area code, `required`. | `"AR0000035970"` |
| `zone_code` | string | **yes** | Zone code, `required`. | `"ZN0000035970"` |
| `prov_code` | string | **yes** | Province code, `required`. | `"REDEMPTIONCITY01"` |
| `region_code` | string | **yes** | Region code, `required`. | `"RCITYREG01"` |
| `subcont_code` | string | **yes** | Sub-continent code, `required`. | `"CNTRCITYSUBCNT01"` |
| `cont_code` | string | **yes** | Continent code, `required`. | `"CNTRCITY"` |
| `id` | object | no | Existing `bmd_registrations.id` (or a 24-character `_id`) to update. Absent means create and answers 201; present answers 200. An unknown `id` is a 404. | `1` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "parish_code": "990033",
  "births": 23,
  "deaths": 0,
  "marriages": 2,
  "month": "Dec",
  "year": "2025",
  "area_code": "AR0000035970",
  "zone_code": "ZN0000035970",
  "prov_code": "REDEMPTIONCITY01",
  "region_code": "RCITYREG01",
  "subcont_code": "CNTRCITYSUBCNT01",
  "cont_code": "CNTRCITY"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "parish_code": "990033",
  "births": 23,
  "deaths": 0,
  "marriages": 2,
  "month": "Dec",
  "year": "2025",
  "area_code": "AR0000035970",
  "zone_code": "ZN0000035970",
  "prov_code": "REDEMPTIONCITY01",
  "region_code": "RCITYREG01",
  "subcont_code": "CNTRCITYSUBCNT01",
  "cont_code": "CNTRCITY",
  "id": 1
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Saved; the body carried an `id`. |
| `201` | Saved; the body carried no `id`. |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | The `id` does not exist in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/bmd_registrations/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"parish_code":"990033","births":23,"deaths":0,"marriages":2,"month":"Dec","year":"2025","area_code":"AR0000035970","zone_code":"ZN0000035970","prov_code":"REDEMPTIONCITY01","region_code":"RCITYREG01","subcont_code":"CNTRCITYSUBCNT01","cont_code":"CNTRCITY"}'
```

### `POST /api/v1/backend/attendances/save`

Create or update an attendance return

**201 when the body carries no `id`, 200 when it does** (`AttendancesController:85`), even though `id` is not persisted. `adult` and `total` are **computed, never accepted**: they are `floatval` sums written into `varchar` columns, so an omitted `men` counts as zero rather than being skipped and a partial save silently understates the headcount — legacy behaviour, kept. Every column the legacy handler assigned is assigned on an update too, so an omitted `sermon_title` becomes `null`; `a1`, `a2`, `a3` and the six above-parish hierarchy columns are never assigned, so an existing value there survives. `offering` is stored as the original string with a numeric mirror maintained alongside for aggregation. `parish_code` must be the caller’s own — legacy took it from the body, which let one parish file another’s return — and an unknown `id` is a 404 rather than an insert at that caller-chosen primary key. Note that the legacy `FormRequest` validates **`chidren`**, not `children`: both keys are accepted, only `children` is stored, and only `chidren` is checked for being numeric.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `center_code` | string | **yes** | `centers.center_code`. Not checked for existence: the legacy database has no foreign keys and 53 live attendances already carry a `center_code` no centre matches. | `"RCCGHF26T6XGAPA"` |
| `parish_code` | string | **yes** | Must equal the caller’s own parish claim. The legacy handler took this straight from the body, which let any authenticated caller file an attendance return against another parish. | `"211716"` |
| `week` | string | **yes** | Week of the month. Live values are `1`–`5`, stored as a `varchar`. | `"1"` |
| `month` | string | **yes** | Month name. Stored verbatim, and the live column holds both `Oct` and `October` — see the monthly report endpoint, which matches either spelling. | `"Oct"` |
| `year` | string | **yes** | Four-digit year, stored as a `varchar`. | `"2025"` |
| `service_date` | string | **yes** | `date_format:Y-m-d` in the legacy `FormRequest`, reproduced exactly. | `"2025-10-05"` |
| `children` | object | no | Children present. **Not validated** — the legacy `FormRequest` spells this rule `chidren`, so any value is accepted here and stored verbatim in the `varchar` column (live rows include `02` and `05`, whose leading zero survives). | `"12"` |
| `chidren` | object | no | The misspelled rule key from `AttendanceRequest::rules()`. Validated as `numeric` and then **discarded**, exactly as legacy did — nothing reads it. Send `children` instead. | `"12"` |
| `teenager` | object | no | Teenagers present. `numeric`. | `"8"` |
| `men` | object | no | Men present. `numeric`. NULL in 279 of the 387 live rows. | `40` |
| `women` | object | no | Women present. `numeric`. NULL in 279 of the 387 live rows. | `55` |
| `altar_call` | object | no | Altar-call responses. `numeric`. | `"3"` |
| `first_timer` | object | no | First timers. `numeric`. | `2` |
| `first_timer_input` | object | no | `nullable` in the legacy rules, so anything is accepted. An `int(11)` column, NULL in 286 of the 387 live rows. | `2` |
| `sermon_title` | object | no | Sermon title. Unvalidated — the `numeric` rule is commented out in the legacy `FormRequest`, and the live column holds free text as well as `--` and `0`. | `"Arise, Shine"` |
| `offering` | object | no | Offering. `numeric`, stored as the original **string** in a `varchar` column — live values include `1000.00` and `5.00`. A numeric mirror is maintained alongside for aggregation; reads always return this string. | `"1000.00"` |
| `a1` | object | no | Spare column, `nullable` in the legacy rules. Validated and then **not persisted**: the legacy `post()` never assigned `a1`, `a2` or `a3`, so an existing value survives a save. | `""` |
| `a2` | object | no | Spare column. Accepted, never persisted. | `""` |
| `a3` | object | no | Spare column. Accepted, never persisted. | `""` |
| `id` | object | no | Existing `attendances.id` (or a 24-character `_id`) to update. Absent means create, and flips the success status from 200 to 201 — a legacy quirk preserved verbatim (`AttendancesController@post`, `:85`). An `id` that does not exist in the caller’s parish is a 404; legacy inserted a new row *at that primary key*. | `387` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "center_code": "RCCGHF26T6XGAPA",
  "parish_code": "211716",
  "week": "1",
  "month": "Oct",
  "year": "2025",
  "service_date": "2025-10-05"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "center_code": "RCCGHF26T6XGAPA",
  "parish_code": "211716",
  "week": "1",
  "month": "Oct",
  "year": "2025",
  "service_date": "2025-10-05",
  "children": "12",
  "chidren": "12",
  "teenager": "8",
  "men": 40,
  "women": 55,
  "altar_call": "3",
  "first_timer": 2,
  "first_timer_input": 2,
  "sermon_title": "Arise, Shine",
  "offering": "1000.00",
  "a1": "",
  "a2": "",
  "a3": "",
  "id": 387
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Saved; the body carried an `id`. |
| `201` | Saved; the body carried no `id`. |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | The `id` does not exist in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/attendances/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"center_code":"RCCGHF26T6XGAPA","parish_code":"211716","week":"1","month":"Oct","year":"2025","service_date":"2025-10-05"}'
```

### `POST /api/v1/backend/attendances/delete`

Soft-delete an attendance return

Answers 200 with `{"message": "Attendance not found: Provide the correct parameter"}` when there is no such row, which is legacy behaviour — clients branch on the text, not the status. The lookup is scoped to the caller’s parish; legacy’s `Attendance::find($id)` was not, so an integer id alone could delete any parish’s return. That also makes a foreign id indistinguishable from a missing one, which is the point.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | **yes** | Legacy `attendances.id`, or a 24-character MongoDB `_id`. | `387` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": 387
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` is missing. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/attendances/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":387}'
```

### `POST /api/v1/backend/attendances/{attendance}/restore`

Restore a soft-deleted attendance return

The legacy handler read `id` from the **request body** and ignored the `{attendance}` path segment entirely — `$request->get()` never consults route parameters — so the id in the URL was decorative. Both are accepted, body first, so existing clients keep working and the documented route finally works too. Legacy also called `->restore()` on a possibly-null model with no tenancy condition, making an unknown id an unconditional 500; it is a 404 here, within the caller’s parish.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `attendance` | string | **yes** | Legacy `attendances.id`, or a 24-character MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `attendances.id`, or a 24-character `_id`. Takes precedence over the `{attendance}` path segment, which the legacy handler never read. | `387` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": 387
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | `{"no_content": true}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such row in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/attendances/{attendance}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/attendances/{attendance}/force-delete`

Permanently delete an attendance return

The legacy route names `AttendancesController@forceDelete`, a method that was never written on the controller or on its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish, and able to reach an already soft-deleted row — which is the row it exists to purge.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `attendance` | string | **yes** | Legacy `attendances.id`, or a 24-character MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | `{"no_content": true}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such row in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/attendances/{attendance}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/attendances/{attendance}`

One attendance return by id

`AttendancesController@form` was **never written**, on the controller or on its base class, so this route has always raised `BadMethodCallException` — a 500. Implemented as the single-record read every sibling `form` in the legacy app performs, scoped to the caller’s parish. A bare object rather than a `data` envelope, matching the single-`Resource` return those siblings used. A row whose `center_code` matches no centre is returned normally: there is no join here, and 53 live rows are orphaned.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `attendance` | string | **yes** | Legacy `attendances.id`, or a 24-character MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The attendance return. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such row in the caller’s parish. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/attendances/{attendance}" \
  -H 'authtoken: $TOKEN'
```

---

## Centers

### `GET /api/v1/backend/centers`

List every centre in the caller’s parish, with its leader’s name

Wrapped in `{"data": […]}` because the legacy handler returned a `CenterCollection`, which is a Laravel `ResourceCollection`. Keys follow the MySQL column order with `id` first, then the two joined columns `hf_leader_fname` and `hf_leader_lname` — the aliases from `select("centers.*", "members.first_name as hf_leader_fname", …)` — and `_id` last.

The join is a **LEFT** join on `members.user_code = centers.hf_leader`, so the 237 centres with a null leader and the 8 whose leader code is an orphan all still appear, with both name keys null. The comparison is case-insensitive, matching MySQL’s `utf8mb4_unicode_ci` — measured: 478 of the 486 distinct leader codes match byte-for-byte, 8 match nothing, and **0** match only case-insensitively, so this costs nothing today but is required for correctness.

**One legacy behaviour is not reproduced**, matching the decision `hf-coordinator` already took: a SQL join emits one row per match, so two members sharing a `user_code` duplicated the centre. Each centre appears once here. Measured: 0 live leader codes are held by more than one member, so no response changes.

Not paginated: the `paginate(20)` at `CentersController.php:25-26` is commented out. Soft-deleted centres are excluded — 30 of the 783 live rows are trashed.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Centres for the parish, with leader names. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/centers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/centers/coordinates`

List the caller’s centres that carry usable map coordinates

Legacy `centreCoordinates`: `whereNotNull("longitude")->whereNotNull("latitude")` plus `!= "-"` on each, which collapses to `{$nin: [null, "-"]}` per column. `$ne: null` also excludes a **missing** field, which is what a column migrated from SQL NULL needs. A blank string was included by MySQL (`"" != "-"` is true) and is included here.

Coverage note: **no live row holds `"-"` in either column**, so that guard has never excluded anything and no historical response exercises it. 389 rows qualify estate-wide.

This method does **not** join `members`, so its rows carry the plain `centers` key set with no `hf_leader_fname`/`hf_leader_lname`. That difference from `GET centers` is faithful to the legacy code, not an oversight.

**Declared before `GET centers/:center`.** Both are one-segment paths, so reversing the order would make this endpoint unreachable — it would resolve as a lookup for a centre whose id is the string `coordinates`, answering an empty 200.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Centres with a latitude and a longitude. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/centers/coordinates" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/centers/save`

Create or update a house-fellowship centre

The message is "House Fellowship Center **Created** Successfully" on both branches, the update included, while the status is `$request->get("id") ? 200 : 201`. Clients branch on the text, so the misleading wording is a contract.

**`center_code` is regenerated on every save, an update included** (`CentersController.php:89-94`). Because `center_code` is the join key for `members.center_code`, `attendances.center_code` and `visitations.center_code`, and none of those is updated to follow, editing a centre silently orphans its whole history. The migrated data carries the fingerprint: **15 visitation rows across 12 distinct codes, and 53 attendances, reference a centre that no longer exists.** Reproduced — it is a data-integrity defect rather than an authorisation hole, and `center_code` is a value clients read back and store — and **escalated as a product decision**.

All nineteen request-sourced columns are written, so an omitted optional is stored as `null`: a save omitting `latitude`/`longitude` removes the centre from `GET centers/coordinates` entirely. Six of them are unvalidated (`country`, `state`, `city`, `hf_leader`, `landmark`, `county`), plus `id`. `hf_leader` is **not** checked against `members` — 8 of the 486 distinct live values are orphans and the legacy database has no foreign keys.

`parish_code` must be the caller’s own. Legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s centre, *move* it, and re-key it in one request.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took it from the body with no tenancy check and matched `firstOrNew` on the primary key alone, so a save could overwrite another parish’s centre and *move* it. Both closed. | `"211774"` |
| `name` | string | **yes** | Centre name. 627 distinct live values in **61 case-variant groups** — the largest such column in this batch (`RHEMA` / `Rhema`, `House of Mercy` / `House Of Mercy` / `House of MERCY` / `HOUSE OF MERCY`, `Hope House` / `hope house`). Never used as a query predicate by *this* controller, only assigned and returned — but `hf-coordinator`’s `?search=` does match it, and it does so case-insensitively for exactly this reason. | `"Nehemiah House Fellowship"` |
| `meeting_day` | string | **yes** | Which day the centre meets. Exactly seven live values — the seven weekday names — with no case variants. The legacy rule is a bare `required`, so no enum is imposed. | `"Sunday"` |
| `meeting_time` | string | **yes** | Free-text clock time; the column is a `varchar` and the legacy rule is a bare `required`, so no format is imposed. 182 distinct live values, one carrying a leading zero. | `"6:00 AM"` |
| `address` | string | **yes** | Street address. 732 distinct live values in **12 case-variant groups** (`Address` / `address`, `Lagos` / `LAGOS`, `ADUWAWA` / `Aduwawa`). Never a query predicate here. | `"174, Durham Road"` |
| `postcode` | string | **yes** | Postcode. 403 distinct live values in **5 case-variant groups** (`al7 2hx` / `AL7 2HX`, `AL10` / `Al10`, `HU7 3DD` / `hu7 3dd`) and **18 carrying a leading zero**, so only the string round-trips. No format rule — the live column holds UK postcodes, Nigerian ones and bare digits like `110115`. | `"SG1 2TA"` |
| `date_created` | string | **yes** | Stored as a `varchar`, not a date. `required\|date_format:Y-m-d`, reproduced including the calendar-validity check `DateTime::createFromFormat` performed — all 445 distinct live values already match, so nothing stored today is rejected. | `"2024-03-03"` |
| `latitude` | object | no | Latitude, stored as a `varchar`. Legacy rule is `["nullable"]`, so **no numeric rule** — and it must stay that way: 360 of the 392 populated live values are decimal strings, but the rest include `57.1499° N`, a degree symbol and a hemisphere letter. A numeric rule would reject rows the API stores today. Two values carry a leading zero. | `"56.4589"` |
| `longitude` | object | no | Longitude, stored as a `varchar`. Same story as `latitude`: 295 decimals among 389 populated values, plus `2.0938° W`. Negative values are common (`-2.9734`). | `"-2.9734"` |
| `operational_status` | object | no | Two live values, `Active` and `Inactive`, with no case variants; null in 5 of 783 rows. Legacy rule is `["nullable"]`, so no enum is imposed. | `"Active"` |
| `country` | object | no | **Not validated** yet assigned. 14 distinct live values, and they are not a consistent vocabulary: ISO-2 codes (`GB`, `NG`, `NL`, `AD`) sit alongside the currency code `NGN`. No enum is imposed, because one would reject that row. | `"GB"` |
| `state` | object | no | **Not validated** yet assigned. 106 distinct live values mixing counties, cities and countries (`Hertfordshire`, `Enugu`, `United Kingdom`). | `"Hertfordshire"` |
| `city` | object | no | **Not validated** yet assigned. 321 distinct live values in **36 case-variant groups** (`LONDON` / `London` / `london`, `Redemption City` / `REDEMPTION CITY`, `City` / `city`) — the second-largest such column here. Not a predicate in this controller, but `hf-coordinator`’s `?search=` matches it case-insensitively. | `"Stevenage"` |
| `hf_leader` | object | no | The `user_code` of the centre’s leader. **Not validated** yet assigned, and deliberately **not checked for existence**: the legacy database has zero foreign keys, and 8 of the 486 distinct live values name no member. 237 of 783 rows leave it null. The 24-character hex form dominates, but UUIDs also appear, so no format rule applies either. | `"67891f3c6726c62e4a2bda3f"` |
| `landmark` | object | no | **Not validated** yet assigned. Populated in only 8 of 783 rows, and the values are largely placeholder text (`Kkb`, `Jsjsjs`, `100`), so no live data meaningfully exercises it. | `"National Theater"` |
| `county` | object | no | **Not validated** yet assigned, and **null in all 783 live rows** — this column has never been written, so no live data exercises it. | `null` |
| `c1` | object | no | Undocumented legacy spare column. Legacy rule is `["nullable"]` — a rule list with no actual constraint. **Null in all 783 live rows.** | `null` |
| `c2` | object | no | Undocumented legacy spare column. Null in all 783 live rows. | `null` |
| `c3` | object | no | Undocumented legacy spare column. Null in all 783 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "parish_code": "211774",
  "name": "Nehemiah House Fellowship",
  "meeting_day": "Sunday",
  "meeting_time": "6:00 AM",
  "address": "174, Durham Road",
  "postcode": "SG1 2TA",
  "date_created": "2024-03-03"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "parish_code": "211774",
  "name": "Nehemiah House Fellowship",
  "meeting_day": "Sunday",
  "meeting_time": "6:00 AM",
  "address": "174, Durham Road",
  "postcode": "SG1 2TA",
  "date_created": "2024-03-03",
  "latitude": "56.4589",
  "longitude": "-2.9734",
  "operational_status": "Active",
  "country": "GB",
  "state": "Hertfordshire",
  "city": "Stevenage",
  "hf_leader": "67891f3c6726c62e4a2bda3f",
  "landmark": "National Theater",
  "county": {},
  "c1": {},
  "c2": {},
  "c3": {},
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no live centre in the caller’s parish. |
| `409` | No unique `center_code` could be allocated. Unreachable in practice. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/centers/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"parish_code":"211774","name":"Nehemiah House Fellowship","meeting_day":"Sunday","meeting_time":"6:00 AM","address":"174, Durham Road","postcode":"SG1 2TA","date_created":"2024-03-03"}'
```

### `POST /api/v1/backend/centers/delete`

Soft-delete a centre

Answers 200 with `{"message": "Center not found: Provide the correct parameter"}` when there is no such centre, which is legacy behaviour — clients branch on the text, not the status.

The id comes from the **body**; this route has no path segment in `api.php`. The lookup is scoped to the caller’s parish; legacy’s `Center::find($id)` was not, so an integer id alone could soft-delete any parish’s centre — and with it the anchor for that centre’s attendance and visitation history.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `centers.id`, or a 24-character MongoDB `_id` for a row created since. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/centers/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"5"}'
```

### `POST /api/v1/backend/centers/{center}/restore`

Restore a soft-deleted centre

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, which the legacy handler ignored entirely.

This path is exercised by migrated data: **30 of the 783 live centres are soft-deleted.**

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `center` | string | **yes** | Legacy `centers.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `centers.id`. Unvalidated, exactly as before. Wins over the `{center}` path segment, which the legacy handler ignored entirely. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such centre in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/centers/{center}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/centers/{center}/force-delete`

Permanently delete a centre

The legacy route named a `CentersController@forceDelete` method that was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row.

It does **not** cascade. Purging a centre leaves its `members`, `attendances` and `visitations` rows pointing at a `center_code` that no longer resolves — which is the legacy shape (zero foreign keys, 1,166 pre-existing orphans). Cascading would destroy reporting history the legacy app kept.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `center` | string | **yes** | Legacy `centers.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such centre in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/centers/{center}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/centers/{center}`

Read one centre

**The legacy `CentersController@form` method does not exist.** The route at `api.php:166` names it, but it is absent from the controller and from its base class, so `Illuminate\Routing\Controller::__call` threw `BadMethodCallException` and this endpoint has always returned 500. There is therefore no historical response to be byte-identical to.

Implemented on the shape every sibling `form` used — a `{"data": […]}` collection of zero or one row — **without** the leader join, matching `centreCoordinates`, the other method returning `CenterCollection` from an unjoined query. The route has only an id segment, so unlike the `sermons` and `events` `form` methods there is no parish in the URL to abuse. A non-numeric id yields an empty collection with a 200, matching MySQL’s int coercion.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `center` | string | **yes** | Legacy `centers.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one centre. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/centers/{center}" \
  -H 'authtoken: $TOKEN'
```

---

## Children

### `GET /api/v1/backend/children`

List the parish’s children’s-ministry members

**This endpoint does not read the `children` table.** The legacy handler queries `members` — `Member::where('church','like','%children%')->where('parish_code', $userInfo->parish)` — and wraps the result in a resource named `ChildCollection`. So the rows are **member** records and the emitted key set is the members column list, not the children one. That mismatch is legacy behaviour and is preserved.

**Case sensitivity is decisive here.** `church LIKE '%children%'` matches **225** rows under MySQL's `utf8mb4_unicode_ci` collation and **1** under a byte comparison — `Children` 179 rows across 90 parishes, `Little Children` 45 across 31, and `children` exactly 1. A literal port would return that single row with a 200 and no error: a roster showing 0.4% of the data. The comparison folds case.

Scoped to the caller’s parish, which the legacy handler already did. `paginate(20)` is commented out in the legacy source, so the envelope is a bare `{"data": […]}` with no `links` or `meta`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A `{"data": […]}` collection of member records. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/children" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/children/save`

Create or update a dependant

The body is `{"saved": true}` on both branches with no message, so a client tells a create from an update only by the status: `$request->get("id") ? 200 : 201`. A PHP-falsy `id` — absent, `null`, `0`, `"0"`, `false`, or `""` folded to null by the global middleware — creates and answers **201**.

**Every legacy rule is `["nullable"]`**, so an empty body is valid and stores a row of nulls — a dependant with no name and no owner, which no scoped read will ever return. Reproduced; adding the `required` rules a reader would expect would reject requests the legacy API accepted. All fourteen columns are written from the request, so a **partial save is destructive**: omitting `photo` erases a working image reference.

**`twitter`, `instagram` and `parish`-side `parent_usercode` are never written** — the handler assigns fourteen of the table’s seventeen business columns. `parent_usercode` is the column the model’s parent relation joins on, so the documented parent link is unreachable through the only write endpoint the table has.

**Closed:** the write was entirely unscoped. `user_code` came from the body and nothing checked it, so any authenticated caller could file a dependant against any parish’s member — on a table with no `parish_code` to notice. The named member must now be in the caller’s parish (403), and on an update the existing row must be in scope too, so a caller cannot overwrite and thereby **move** another parish’s dependant. A truthy `id` naming no live in-scope row is a **404**; legacy inserted at the caller’s chosen primary key.

A create with **no** `user_code` is still allowed and stores an unowned row, for parity — such a row is in nobody’s scope and readable by nobody, so it is junk data rather than a leak.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | object | no | The member this dependant belongs to. `["nullable"]`, so it may be omitted — and legacy then stored an **unowned** row, which no scoped read can ever return. Reproduced. When it *is* supplied, the named member must be in the caller’s parish, else 403: the `children` table has no `parish_code`, so this code is the only route to a tenancy decision. | `"65e7daaea42f7669370efdcf"` |
| `photo` | object | no | Stored path or URL. No upload handling on this route; the string is taken verbatim. | `"uploads/children/ada.jpg"` |
| `first_name` | object | no | Given name. Legacy rule `["nullable"]`. | `"Ada"` |
| `last_name` | object | no | Family name. Legacy rule `["nullable"]`. | `"Okoye"` |
| `gender` | object | no | Free text; no enum was imposed. | `"Female"` |
| `student` | object | no | Whether the dependant is in education, stored as a bare string rather than a flag. | `"Yes"` |
| `dob` | object | no | Date of birth. **`varchar` in the live table, and the legacy rule is `["nullable"]` with no `date` or `date_format`** — so any string is accepted and stored verbatim. No format is imposed here for the same reason, and the value is returned exactly as it was sent. | `"2015-04-02"` |
| `address` | object | no | Free text. | `"12 Allen Avenue, Ikeja"` |
| `phone` | object | no | Free text; **not** validated as numeric, unlike some sibling controllers. | `"+2348012345678"` |
| `email` | object | no | Free text; no `email` rule, so a malformed address is accepted and stored. | `"ada@example.com"` |
| `facebook` | object | no | Social handle. Note the table also has `twitter` and `instagram`, which this endpoint neither validates nor writes — `facebook` is the only one of the three it touches. | `"ada.okoye"` |
| `c1` | object | no | Undocumented legacy spare column. No live row exercises it — the table is empty. | `null` |
| `c2` | object | no | Undocumented legacy spare column. | `null` |
| `c3` | object | no | Undocumented legacy spare column. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. The body is `{"saved": true}` on both branches. A truthy `id` naming no live in-scope row is a **404**; legacy inserted at the caller’s chosen primary key. | `"3"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "user_code": "65e7daaea42f7669370efdcf",
  "photo": "uploads/children/ada.jpg",
  "first_name": "Ada",
  "last_name": "Okoye",
  "gender": "Female",
  "student": "Yes",
  "dob": "2015-04-02",
  "address": "12 Allen Avenue, Ikeja",
  "phone": "+2348012345678",
  "email": "ada@example.com",
  "facebook": "ada.okoye",
  "c1": {},
  "c2": {},
  "c3": {},
  "id": "3"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or the named member belongs to another parish. |
| `404` | A truthy `id` naming no live in-scope dependant. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/children/save" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/children/delete`

Soft-delete a dependant

The id comes from the **body**: this route has no `{child}` segment at all (`api.php:250`), unlike its `restore` and `force-delete` siblings — an inconsistency in `api.php`, reproduced.

Legacy validated nothing and called `->delete()` on a possibly-null model, so an absent or unknown id was an unconditional **500**. That is a **404** here. The success body `{"no_content": true}` with a 200 is preserved.

**Closed:** the delete was unscoped, so any authenticated caller could remove any parish’s dependant by id. Resolution now runs through `children.user_code → members.parish_code`; an out-of-scope id is a 404, indistinguishable from a missing one.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `children.id`, or a 24-character MongoDB `_id`. Unvalidated, matching a route that declared no rules at all. | `"3"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "3"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Soft-deleted. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such in-scope dependant; legacy answered 500. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/children/delete" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/children/{child}/restore`

Restore a soft-deleted dependant

Legacy validated nothing and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500 — a **404** here. The body `id` wins over the `{child}` path segment, which `$request->get("id")` never consulted; the segment is honoured as a fallback so the documented route finally works. Scoped, where legacy was not.

**No live data exercises this path** — the `children` collection is empty, so there is nothing to restore and no historical response to match. It rests entirely on synthetic fixtures.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `child` | string | **yes** | Legacy `children.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `children.id`. Unvalidated. Wins over the `{child}` path segment, which the legacy handler ignored entirely. | `"3"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "3"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such in-scope dependant; legacy answered 500. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/children/{child}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/children/{child}/force-delete`

Permanently delete a dependant

The legacy route names a `forceDelete` method that was never written, so this endpoint has always returned 500. Implemented as the scoped hard delete every sibling `forceDelete` in this conversion performs, able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `child` | string | **yes** | Legacy `children.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Permanently deleted. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such in-scope dependant. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/children/{child}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/children/{child}`

Read one dependant

The route names `ChildrenController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `{"data": […]}` collection of zero or one row every sibling scaffold produces, and **scoped**, which legacy’s reads of this table were not.

Scoping is indirect — the dependant’s `user_code` must name a member in the caller’s parish — so a dependant belonging to another parish, or one orphaned from `members` entirely, yields an empty collection rather than a 403: there is no parish segment in the URL for a caller to be told they got wrong. A garbage `{child}` likewise yields an empty collection with a 200, because MySQL coerced it to 0 against an `int` primary key and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `child` | string | **yes** | Legacy `children.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one dependant. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/children/{child}" \
  -H 'authtoken: $TOKEN'
```

---

## Churches

### `GET /api/v1/backend/churches/getChurchList`

The age-band list as a bare name/value array

Legacy returned the Eloquent collection **directly**, not through a resource, so the body is a bare JSON array with **no `data` envelope** — and because of `select("name","value")` each element carries only those two keys, with no `id`.

Unscoped, and correctly so: these seven rows are the age bands every parish shares, and `value` is what `members.church` stores. This is also the one endpoint on this controller that has always worked, which is why implementing `GET churches` as the same list exposes nothing new.

Two data quirks are returned verbatim: `elders` appears **twice** (rows 5 and 6, the second named `new church`), and row 7’s code is the typo `infannt`. Neither is corrected or de-duplicated — members store these strings, so a "fix" would orphan every member carrying `infannt`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A bare array of `{name, value}`. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/churches/getChurchList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/churches`

List every age-band category

**This endpoint has always returned 500.** The legacy handler is `Church::where("parish_code", $parish_code)->get()` (`ChurchesController.php:29-31`) against a table that has **no `parish_code` column** — verified against the dump’s `CREATE TABLE churches` and against all 7 live documents — so MySQL raised error 1054 `Unknown column 'parish_code' in 'where clause'`, Laravel wrapped it in a `QueryException`, and every request failed. `post()` confirms it: it assigns six columns and not `parish_code`, because there is nowhere to put one.

Implemented as the unscoped reference list the table supports, wrapped in `{"data": […]}` because `ChurchCollection` is a Laravel `ResourceCollection`. The same reasoning the repo already applied to the never-written `form` and `forceDelete`: an endpoint that has only ever produced a 500 has no response contract to preserve. This exposes nothing new, since `GET churches/getChurchList` already returns every row unscoped.

Not paginated — `paginate(20)` at `:32` is commented out, and 7 rows is one page regardless. Soft-deleted rows are excluded, though 0 of the 7 are trashed so no live data exercises it.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Every age-band category. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/churches" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/churches/create`

Create or update an age-band category

The path is `create`, not `save` — every sibling module in this batch spells it `save`, and `api.php:314` spells this one differently. Preserved.

The message is "New Church Category Added Successfully" on both branches, the update included, while the status is `$request->get("id") ? 200 : 201`. So a truthy `id` answers **200** and still claims an addition; absent, `null`, `0`, `"0"` or `false` answers **201**.

All six columns are written from the request, so an omitted optional is stored as `null` — a save omitting `age_range` **erases the band**, and all 7 live rows carry one. Four of the six (`age_range`, `c1`–`c3`) carry no validation rule at all; the handler also validates `name` and `value` twice, once via `ChurchRequest` and once via an unreachable inline validator.

A truthy `id` naming no row is a 404; legacy inserted at the caller’s chosen primary key.

**Authenticated but not authorised**, because there is no tenancy column to authorise against: any valid token can add or edit an age band every parish sees. Legacy behaviour, preserved, and escalated as a product decision alongside the identical situation in `variables`.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `name` | string | **yes** | The display label for the age band. 7 distinct live values, **0 case-variant groups** — `Children`, `Teen`, `Youth/Young Adult`, `Adult`, `Elders`, `Infants`, and `new church`, the last plainly a test row that reached production. Required by both the FormRequest and the handler’s inline validator. | `"Children"` |
| `value` | string | **yes** | The stored code, which is what `members.church` holds. 6 distinct values across 7 rows: `elders` appears **twice** (rows 5 and 6) and one is the typo `infannt`. Neither is corrected — clients store and compare these strings, so a "fix" would orphan every member carrying `infannt`. There is no unique constraint on the column. Required by both validators. | `"children"` |
| `age_range` | object | no | Free-text age band. **Not validated at all** — neither the live FormRequest rules nor the inline validator mentions it, yet `post()` assigns it. 6 distinct live values, and they are neither contiguous nor consistently formatted: `0-12`, `13-19`, `20-35`, `36 Above`, `60-90`, `0-2`. `36 Above` and `60-90` overlap, and nothing between 36 and 60 has a band of its own. No format rule is imposed, because one would reject rows stored today. | `"0-12"` |
| `c1` | object | no | Undocumented legacy spare column. **Not validated** yet assigned. Null in all 7 live rows — never written, so no live data exercises it. | `null` |
| `c2` | object | no | Undocumented legacy spare column. Null in all 7 live rows. | `null` |
| `c3` | object | no | Undocumented legacy spare column. Null in all 7 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. The message is "New Church Category Added Successfully" on both branches, so an update reports an addition. | `"1"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "name": "Children",
  "value": "children"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "name": "Children",
  "value": "children",
  "age_range": "0-12",
  "c1": {},
  "c2": {},
  "c3": {},
  "id": "1"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `404` | A truthy `id` that names no live category. |
| `422` | Validation failed: `name` or `value` missing. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/churches/create" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"name":"Children","value":"children"}'
```

### `POST /api/v1/backend/churches/delete`

Soft-delete an age-band category

Legacy validated nothing and checked nothing — `Church::find($id)` followed straight by `->delete()` — so an absent or unknown `id` called a method on `null` and this endpoint returned an unconditional **500**. Its `spirituals` and `department_lists` batch-mates both declare `id => required|numeric` *and* check `isset()`, answering 200 with a message; this one and `variables` do neither. Fixed to a **404**, matching how every sibling `restore` in this conversion was fixed: a null-pointer 500 is not a response contract.

On success the body is `{"no_content": true}` with a 200, which *is* preserved. Note the consequence of deleting a row here: `members.church` stores the `value` string and there are no foreign keys, so every member in that band keeps a code no list endpoint will return.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `churches.id`, or a 24-character MongoDB `_id`. Unvalidated, matching the legacy handler, which declared no rules for this route at all. | `"1"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "1"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted. |
| `401` | Missing or invalid token. |
| `404` | No such category; legacy answered 500 here. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/churches/delete" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/churches/{church}/restore`

Restore a soft-deleted age-band category

Legacy validated nothing and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here. The body `id` wins over the path segment, which the legacy handler ignored entirely.

**No live data exercises this path**: `deleted_at` is null in all 7 rows, so there is nothing to restore and no historical response to be identical to. It rests entirely on synthetic fixtures.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `church` | string | **yes** | Legacy `churches.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `churches.id`. Unvalidated. Wins over the `{church}` path segment, which the legacy handler ignored entirely. | `"1"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "1"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `404` | No such category. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/churches/{church}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/churches/{church}/force-delete`

Permanently delete an age-band category

The legacy route named a `ChurchesController@forceDelete` method that was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `church` | string | **yes** | Legacy `churches.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `404` | No such category. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/churches/{church}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/churches/{church}`

Read one age-band category

The route names `ChurchesController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row, unscoped because the table has no scope. A garbage `{church}` yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `church` | string | **yes** | Legacy `churches.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one category. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/churches/{church}" \
  -H 'authtoken: $TOKEN'
```

---

## Department Lists

### `GET /api/v1/backend/departmentLists`

List the department catalogue for the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned a `DepartmentListCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `DepartmentListsController.php:24-25` is commented out and a bare `get()` took its place.

Soft-deleted rows are excluded, matching the Eloquent global scope. That is load-bearing on this collection: **319 of the 4,889 live rows are trashed**, so a missing scope would inflate the estate’s listing by 7%.

Note the table naming, which is inverted from intuition: `department_lists` is the department *master catalogue*, and `departments` is the member↔department join table.

**Responses**

| Status | Meaning |
|---|---|
| `200` | The parish’s department catalogue. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departmentLists" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/getDepartmentLists`

Alias of GET departmentLists

`api.php:287` wires this path to `DepartmentListsController@index`, **not** to the controller’s `getDepartmentList()` method — so despite the name it returns the same parish-scoped `{"data": […]}` collection as `GET departmentLists`, not a `dept_code`/`name` projection. Preserved as an alias because clients call it.

`getDepartmentList()` itself is unrouted dead code and is not ported; it would have read the whole catalogue across every parish with no scoping.

**Responses**

| Status | Meaning |
|---|---|
| `200` | The parish’s department catalogue. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/getDepartmentLists" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/departmentLists/save`

Create or update a department in the catalogue

**This endpoint regenerates `dept_code` on every save, updates included** (`DepartmentListsController.php:56-61`). `dept_code` is the only link `departments` has to this catalogue, nothing updates `departments` to follow, and the database has no foreign keys — so renaming a department silently orphans every membership in it. The migrated data already carries the fingerprint: 6 of the 473 distinct `departments.dept_code` values name no catalogue row. Reproduced, pinned by a spec, and escalated as a product decision.

The message is "Department Added Successfully" on both branches, the update included, while the status is `$request->get("id") ? 200 : 201`. So a truthy `id` answers **200** and still claims an addition; absent, `null`, `0`, `"0"` or `false` answers **201**.

All six columns are written from the request, so an omitted optional is stored as `null`. `leader_usercode`, `d1` and `d2` are null in every live row, so there is nothing to lose today. A `dept_code` sent in the body is accepted by the FormRequest and then discarded.

`parish_code` must be the caller’s own. Legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s department and *move* it. A truthy `id` naming a **trashed** row is a 404 — legacy created a duplicate at the same id instead.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `name` | string | **yes** | The department name. 1,329 distinct live values in **172 case-variant groups** — the heaviest case-variant column in this database (`MUSIC`/`Music`, `PRAYER`/`Prayer`/`prayer`, `PROTOCOL`/`Protocol`/`protocol`). Never a query predicate in *this* controller, only assigned and returned, so no matcher applies here; anything that ever filters or groups on it must compare case-insensitively or it will split one department into three. | `"MUSIC"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took it from the body with no tenancy check, so any authenticated caller could file a department into another parish — or, combined with the unscoped `firstOrNew`, *move* an existing one. All 423 distinct live values are all-digits. | `"211774"` |
| `dept_code` | object | no | Accepted and then **discarded**. The legacy rule is `["nullable"]`, but `post()` overwrites the column with a newly generated `RPMSDPMT…` code on every save, updates included. Documented because the field is in the FormRequest and a client may reasonably think it is honoured. | `"RPMSDPMTX52HVT"` |
| `leader_usercode` | object | no | Intended to hold the department leader’s `members.user_code`. Legacy rule is `["nullable"]`. **Null in all 4,889 live rows** — this column has never been written, so no migrated data exercises it. | `null` |
| `d1` | object | no | Undocumented legacy spare column. Null in all 4,889 live rows. | `null` |
| `d2` | object | no | Undocumented legacy spare column. Null in all 4,889 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all** — `DepartmentListRequest` does not mention it. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. The message is "Department Added Successfully" on both branches. | `"1"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "name": "MUSIC",
  "parish_code": "211774"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "name": "MUSIC",
  "parish_code": "211774",
  "dept_code": "RPMSDPMTX52HVT",
  "leader_usercode": {},
  "d1": {},
  "d2": {},
  "id": "1"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no live department in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departmentLists/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"name":"MUSIC","parish_code":"211774"}'
```

### `POST /api/v1/backend/departmentLists/delete`

Soft-delete a department from the catalogue

Answers 200 with `{"message": "Department not found: Provide the correct parameter"}` when there is no such row, which is legacy behaviour — clients branch on the text, not the status. Unlike the `churches` and `variables` handlers in this batch, this one checks before deleting and so never 500s.

The id comes from the **body**; this route has no path segment in `api.php`. The lookup is scoped to the caller’s parish; legacy’s `DepartmentList::find($id)` was not, so an integer id alone could soft-delete any parish’s department — and with it, in effect, every membership pointing at that `dept_code`.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `department_lists.id`, or a 24-character MongoDB `_id` for a new row. | `"1"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "1"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departmentLists/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"1"}'
```

### `POST /api/v1/backend/departmentLists/{departmentList}/restore`

Restore a soft-deleted department

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, which the legacy handler ignored entirely.

Well exercised by migrated data: 319 live rows are soft-deleted, more than any other collection in this batch.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `departmentList` | string | **yes** | Legacy `department_lists.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `department_lists.id`. Unvalidated, exactly as before. Wins over the `{departmentList}` path segment, which the legacy handler ignored entirely. | `"1"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "1"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such department in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departmentLists/{departmentList}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/departmentLists/{departmentList}/force-delete`

Permanently delete a department

The legacy route named a `DepartmentListsController@forceDelete` method that was never written, so this endpoint has always returned 500 — on the collection with the most trashed rows in the batch. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `departmentList` | string | **yes** | Legacy `department_lists.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such department in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departmentLists/{departmentList}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departmentLists/{departmentList}`

Read one catalogue department

The route names `DepartmentListsController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row, scoped to the caller’s parish. A garbage `{departmentList}` yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `departmentList` | string | **yes** | Legacy `department_lists.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one department. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departmentLists/{departmentList}" \
  -H 'authtoken: $TOKEN'
```

---

## Department transfers

### `GET /api/v1/backend/department-transfers`

List a member’s department history, within the caller’s parish

**Security fix — this was a whole-estate member lookup.** The legacy handler validated `user_code` and then filtered on **that alone**, with no token read and no parish condition, so any authenticated caller who knew or guessed a member’s code received that member’s complete department history — `parish_code` included. The filter is now the caller’s parish intersected with the requested `user_code` (with `$and`, never a spread), so a code belonging to another parish returns an empty envelope: the same answer as an unknown code, deliberately, so the endpoint cannot be used as an oracle for which parish a member belongs to.

**The envelope is hand-built and is not a Laravel resource collection.** It is `{"success": true, "total": N, "data": [ … ]}` with exactly **eleven keys per row**, in this order: `id`, `user_code`, `parish_code`, `department_name`, `date_from`, `date_to`, `t1`, `t2`, `t3`, `created_at`, `updated_at`. Note it **omits `deleted_at`** even though the column exists, and that there is no `links`/`meta` — this endpoint has **no paginator**, unlike `GET transfers` whose `paginate(20)` is live. `total` is the length of `data`, not a table-wide count.

Sorted by `date_from` **descending**, compared as a string, which is what MySQL did — the column is a `varchar`, not a `date`.

**No live data exercises any of this.** `department_transfers` is empty in the dump, so this envelope has never been serialised for a real row.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | object | **yes** | The member whose department history to list. `required\|string`, so it cannot be omitted. **Security note:** this parameter alone used to be sufficient. The legacy handler filtered on nothing else, so any authenticated caller could read any member’s entire department history — `parish_code` included — by supplying their code. The query is now additionally scoped to the caller’s own parish, so a code belonging to another parish returns an empty envelope. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The member’s department history, newest first. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `user_code` missing, or not a string. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/department-transfers" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/department-transfers/save`

Create or update a department transfer record

The message is `"Transfer record saved successfully."` — lower-case, with a trailing full stop. Its `transfers` sibling says `"Transfer Record Saved Successfully"` in title case with no stop. Two controllers, two spellings, both reproduced verbatim. The status is `!$request->get("id") ? 201 : 200`, so absent, `null`, `0`, `"0"` or `false` creates and answers **201**.

`user_code`, `parish_code` and `department_name` are `required|string`, which means a JSON **number** is a **422** here where `transfers/save` would have cast it — the rules genuinely differ. `max:255` on `department_name` is a plain **length** check, because Laravel only treats a numeric value as its own size when the ruleset also carries a `numeric` rule, which this one does not; so `"300"` is accepted.

`date_from` and `date_to` are `nullable|date`, which is much looser than the sibling `transfers/save`’s `date_format:Y-m-d`: any format PHP recognises is accepted provided it parses to a real calendar day, so `2024-03-06 10:30:00` and `03/14/2024` pass here and fail there. `date_to` additionally carries `after_or_equal:date_from`, compared on timestamps so equal dates pass — and skipped entirely when `date_from` is null, because PHP compared against `false`.

All eight columns are written from the request, so an omitted optional is stored as `null`: a partial save is **destructive**, reproduced rather than quietly narrowed.

`parish_code` must be the caller’s own, and the value **stored** is the token’s. Legacy took it from the body with no check, and `firstOrNew` matched the primary key alone, so one parish could overwrite another’s record and *move* it. A truthy `id` naming no live record in the caller’s parish is a **404**; legacy INSERTed at that caller-chosen primary key instead. A truthy `id` naming a **trashed** record is also a 404 — legacy created a duplicate row at the same id.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | object | **yes** | The member this department history belongs to. `required\|string` — so unlike `transfers/save`, a JSON **number** is a 422 rather than being cast. No existence check, and none is added: the legacy database has zero foreign keys, so orphaned references must be tolerated. | `"65e7e542a42f7669370efe01"` |
| `parish_code` | object | **yes** | Must be the caller’s own parish claim. Legacy took it straight from the body with no tenancy check, so any authenticated caller could file a department record into another parish; and `firstOrNew(["id" => …])` matched the primary key alone, so a save could *move* another parish’s record. Both closed — a mismatch is a **403**. | `"211618"` |
| `department_name` | object | **yes** | The department’s name, stored as free text rather than a reference into `department_lists`. `required\|string\|max:255` — a **length** check, because Laravel only treats a numeric value as its own size when the ruleset also carries a `numeric` rule, which this one does not. So `"300"` is accepted. Never used as a query predicate by this controller. | `"Sanctuary Keeper"` |
| `date_from` | object | no | `nullable\|date`. **Deliberately looser than the sibling `transfers/save`**, which uses `date_format:Y-m-d`: Laravel’s `date` accepts any format PHP recognises provided the parse yields a real calendar day, so `2024-03-06 10:30:00` and `03/14/2024` are accepted here and refused there. `tomorrow`, `14/03/2024` and `2024-13-45` are refused by both. Stored verbatim in a `varchar` column. | `"2024-03-06"` |
| `date_to` | object | no | `nullable\|date\|after_or_equal:date_from`. The comparison is on **timestamps**, so a `date_to` equal to `date_from` passes. When `date_from` is absent or null the comparison is skipped, because PHP compared against `false` and `$timestamp >= false` is true — a body carrying only `date_to` was accepted and still is. | `"2024-09-30"` |
| `t1` | object | no | Undocumented legacy spare column. The rule is `["nullable"]` — a rule list with no actual constraint — so anything scalar is stored verbatim. **No live row exists to exercise it:** the `department_transfers` table is empty in the dump. | `null` |
| `t2` | object | no | Undocumented legacy spare column. No live row exists to exercise it. | `null` |
| `t3` | object | no | Undocumented legacy spare column. No live row exists to exercise it. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. The legacy status is `!$request->get("id") ? 201 : 200`, so a PHP-truthy value updates and answers **200** while absent, `null`, `0`, `"0"` or `false` creates and answers **201**. **This is the parameter that made the write dangerous.** Legacy matched on the primary key alone, so a caller could name any parish’s row and overwrite it, or INSERT at a primary key of their choosing. A truthy `id` naming no live record **in the caller’s parish** is now a 404. Validation is left as loose as it was so the `errors` map is unchanged. | `"12"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "65e7e542a42f7669370efe01",
  "parish_code": "211618",
  "department_name": "Sanctuary Keeper"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "user_code": "65e7e542a42f7669370efe01",
  "parish_code": "211618",
  "department_name": "Sanctuary Keeper",
  "date_from": "2024-03-06",
  "date_to": "2024-09-30",
  "t1": {},
  "t2": {},
  "t3": {},
  "id": "12"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no live record in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/department-transfers/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"65e7e542a42f7669370efe01","parish_code":"211618","department_name":"Sanctuary Keeper"}'
```

### `POST /api/v1/backend/department-transfers/delete`

Soft-delete a department transfer record

**Answers a real 404 for a missing row**, with `{"message": "Record not found."}` — unlike `POST transfers/delete`, which answers HTTP 200 with a not-found message. Both are preserved as written: this controller simply got it right and its sibling did not, and each is what its clients see today. Success is `{"message": "Transfer record deleted."}` with a 200.

The id comes from the **body**; this route has no path segment in `api.php`. The rule is `required|numeric`, widened only to also accept a 24-character `_id` so that rows this API created (which have no `legacy_id`) are deletable at all; the rejection message keeps Laravel’s `numeric` wording so the `errors` map is unchanged.

**Security fix:** the lookup had no tenancy condition, so an integer id alone soft-deleted any parish’s record. It is now scoped, and an out-of-parish row yields the same 404 as a missing one.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `department_transfers.id`, or a 24-character MongoDB `_id` for a row created since. | `"12"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "12"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Soft-deleted. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such live record in the caller’s parish. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/department-transfers/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"12"}'
```

### `POST /api/v1/backend/department-transfers/{id}/restore`

Restore a soft-deleted department transfer record

**The one route in either transfer module whose path parameter genuinely works.** The legacy signature is `restore(Request $request, $id)` and the handler reads `$id`, the route parameter — so unlike `POST transfers/{transfer}/restore`, whose handler read `$request->get("id")` and therefore ignored its segment entirely, this path has always been functional. **No body `id` is accepted**, because adding one would give the endpoint a source the legacy handler never consulted.

It also already 404s on a missing row (`{"message": "Record not found."}`) instead of calling `->restore()` on null, which is what makes the `transfers` equivalent an unconditional 500. Success is `{"message": "Transfer record restored."}` with a 200.

**Security fix:** the lookup had no tenancy condition, so an integer id alone restored any parish’s record. It is now scoped, and reaches a trashed row via the soft-delete plugin’s restore exemption.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `id` | string | **yes** | Legacy `department_transfers.id`, or a MongoDB `_id`. **The only source** — see below. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/department-transfers/{id}/restore" \
  -H 'authtoken: $TOKEN'
```

---

## Departments

### `GET /api/v1/backend/departments/getDepartmentsByHierarchy/{type}/{type_code}`

Department requests joined to their member, filtered by hierarchy level or department

Reproduces the legacy `departments LEFT JOIN members` row shape, in which the seven column names the two tables share are taken from `members` — so `id` is the **member’s** primary key and `status` is the member’s, not the request’s. The legacy handler had no tenancy scoping at all: the caller’s own scope is now intersected in with `$and`. Its `parish` branch read an undefined `$parish_code` and therefore always returned an empty list; it uses the caller’s parish here. `filterByDepartment` matches case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did — `%choir%` is 128 live rows that way and 13 byte for byte.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | `parish` \| `area` \| `zone` \| `province` \| `region` \| `continent` \| `user_code` \| `department` \| `filterByDepartment` | **yes** | Branch selector, compared **case-sensitively** exactly as PHP `==` did. Anything not listed falls through to the caller’s own parish, including `subContinent` — the legacy handler has no sub-continent branch and one was not invented. |
| `type_code` | string | **yes** | Code at the requested level. Ignored by the `parish` branch, which uses the caller’s own parish claim. For `filterByDepartment` it is a substring of `departments.name`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Joined department rows in scope, `{"data": […]}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no hierarchy claim, or the branch needs a parish claim the token lacks. |
| `422` | Validation failed. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments/getDepartmentsByHierarchy/{type}/{type_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departments/pendingDepartmentRequestList`

Department requests awaiting a decision in the caller’s parish

`approval = 0`. Wrapped in `{"data": […]}` because the legacy handler returned a `DepartmentCollection` over a plain collection — no paginator, so no `links` or `meta`. Measured across the live table: 168 rows are pending, 411 approved and 63 disapproved.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Pending requests for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments/pendingDepartmentRequestList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departments/approvedDepartmentRequestList`

Approved department requests in the caller’s parish

`approval = 1`, in a `{"data": […]}` envelope.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Approved requests for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments/approvedDepartmentRequestList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departments/disapprovedDepartmentRequestList`

Disapproved department requests in the caller’s parish

`approval = 9`, in a `{"data": […]}` envelope.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Disapproved requests for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments/disapprovedDepartmentRequestList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departments/approveDepartmentRequest/{user_code}/{dept_code}`

Approve a member’s department request (legacy verb: GET)

Sets `approval = 1`, `status = "Approved"`, `action_by = "RPMS_Admin"` and emails the member. A mutating `GET`, preserved because clients call it this way. The legacy update matched on `dept_code` and `user_code` only, so any authenticated caller could approve any parish’s request; the caller’s parish is now part of the filter. The status and message are unchanged — 200 `Request Approved Successfully` even when nothing matched — but the email is no longer sent in that case, since legacy sent it regardless and thereby let an unauthorised caller mail an arbitrary member. A mail failure never fails the request.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code` whose department request is being actioned. |
| `dept_code` | string | **yes** | `departments.dept_code` being actioned. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | `{"message": "Request Approved Successfully"}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments/approveDepartmentRequest/{user_code}/{dept_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departments/disapproveDepartmentRequest/{user_code}/{dept_code}`

Disapprove a member’s department request (legacy verb: GET)

Sets `approval = 9`, `status = "disapproved"`, `action_by = "RPMS_Admin"`. **`message` is a number**: the handler builds the string `Request Disapproved` and then returns the integer row count instead, which is reproduced verbatim. No email is sent. Scoped to the caller’s parish, which legacy was not.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code` whose department request is being actioned. |
| `dept_code` | string | **yes** | `departments.dept_code` being actioned. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | `{"message": <rows changed>}` — an integer, not a string. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments/disapproveDepartmentRequest/{user_code}/{dept_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departments`

Paginated list of department requests in the caller’s parish

The one handler in this controller that really paginates (`paginate(20)`), so the one whose body carries Laravel’s `links` and `meta` blocks alongside `data`. **The legacy query had no tenancy condition at all** — `Department::query()->paginate(20)` returned every parish’s requests, and unlike its three sibling list methods it never consulted the token. Scoped here, so `meta.total` is the caller’s own count rather than the estate’s 628.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `page` | number | no | Page number, as read by Laravel’s paginator. Defaults to 1. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | `{data, links, meta}` for the caller’s parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `page` is not a positive integer. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/departments/save`

Create or update a department request

**201 when the body carries no `id`, 200 when it does** — a legacy quirk preserved verbatim, since `id` is never persisted and only switched the status code. All nine columns are written from the request, so an omitted `d1`/`d2`/`d3` is stored as `null`; that makes a partial save destructive and is also legacy behaviour. `status`, `approval` and `action_by` are not accepted: legacy never assigned them either, so a create takes the defaults (`pending`/`0`/`null`) and an update leaves the approval workflow alone. `parish_code` must be the caller’s own — legacy trusted the body — and an unknown `id` is a 404 rather than an insert at that primary key.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `name` | string | **yes** | Department name as typed by the requester. Free text — the live column holds 197 distinct spellings including `CHOIR`, `Choir` and `choir`. | `"Choir"` |
| `date_joined` | string | **yes** | Join date. `date_format:Y-m-d` in the legacy `FormRequest`, reproduced exactly. | `"2024-03-06"` |
| `dept_code` | string | **yes** | `department_lists.dept_code` — the catalogue entry being joined. | `"RPMSDPMTX52HVT"` |
| `user_code` | string | **yes** | `members.user_code` — the member joining the department. | `"65e8001ea42f7669370efe1c"` |
| `parish_code` | string | **yes** | Must equal the caller’s own parish claim. Legacy trusted this field, which let any authenticated caller write a department request into another parish. | `"211774"` |
| `d1` | string | no | Undocumented legacy spare column. Also the flag `sendApprovalEmailNotification` set to `1` once an approval email went out; it is null in all 642 live rows. | `""` |
| `d2` | string | no | Undocumented legacy spare column. | `""` |
| `d3` | string | no | Undocumented legacy spare column. | `""` |
| `id` | object | no | Existing `departments.id` (or a 24-character `_id`) to update. Absent means create, and flips the success status from 200 to 201 — a legacy quirk preserved verbatim (`DepartmentsController@post`, `:199`). An `id` that does not exist in the caller’s parish is a 404; legacy inserted a new row *at that primary key*, which would collide with a re-import of the MySQL id space. | `12` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "name": "Choir",
  "date_joined": "2024-03-06",
  "dept_code": "RPMSDPMTX52HVT",
  "user_code": "65e8001ea42f7669370efe1c",
  "parish_code": "211774"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "name": "Choir",
  "date_joined": "2024-03-06",
  "dept_code": "RPMSDPMTX52HVT",
  "user_code": "65e8001ea42f7669370efe1c",
  "parish_code": "211774",
  "d1": "",
  "d2": "",
  "d3": "",
  "id": 12
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Saved; the body carried an `id`. |
| `201` | Saved; the body carried no `id`. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | The `id` does not exist in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departments/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"name":"Choir","date_joined":"2024-03-06","dept_code":"RPMSDPMTX52HVT","user_code":"65e8001ea42f7669370efe1c","parish_code":"211774"}'
```

### `POST /api/v1/backend/departments/delete`

Soft-delete a department request

Answers 200 with `{"message": "Department not found: Provide the correct parameter"}` when there is no such row, which is legacy behaviour — clients branch on the text, not the status. The lookup is scoped to the caller’s parish; legacy’s `Department::find($id)` was not, so an integer id alone could delete any parish’s request.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | **yes** | Legacy `departments.id`, or a 24-character MongoDB `_id` for post-migration rows. | `1` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": 1
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` is missing. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departments/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":1}'
```

### `POST /api/v1/backend/departments/{department}/restore`

Restore a soft-deleted department request

The legacy handler read `id` from the **request body** and ignored the `{department}` path segment entirely, so the URL’s id was decorative. Both are accepted, body first, so existing clients keep working. Legacy also called `->restore()` on a possibly-null model with no tenancy condition, making an unknown id an unconditional 500; it is a 404 here, within the caller’s parish.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `department` | string | **yes** | Legacy `departments.id`, or a 24-character MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `departments.id`, or a 24-character `_id`. Takes precedence over the `{department}` path segment, which the legacy handler never read. | `1` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": 1
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | `{"no_content": true}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such row in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departments/{department}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/departments/{department}/force-delete`

Permanently delete a department request

The legacy route named `DepartmentsController@forceDelete`, a method that was never written on the controller or its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `department` | string | **yes** | Legacy `departments.id`, or a 24-character MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | `{"no_content": true}`. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such row in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/departments/{department}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/departments/{department}`

One department request by id

`DepartmentsController@form` was **never written**, on the controller or on its base class, so this route has always raised `BadMethodCallException` — a 500. Implemented as the single-record read every sibling `form` in the legacy app performs, scoped to the caller’s parish. A bare object rather than a `data` envelope, matching the single-`Resource` return those siblings used.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `department` | string | **yes** | Legacy `departments.id`, or a 24-character MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The department request. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such row in the caller’s parish. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/departments/{department}" \
  -H 'authtoken: $TOKEN'
```

---

## Employments

### `GET /api/v1/backend/employments/professionals/{scope}/{scope_code}`

Professionals at a hierarchy level, with their employment detail

Joins `employments` to `members` on `user_code` and returns every member with a non-blank `profession`. The legacy handler consulted **no** token, so `scope=national` returned every professional in the database with their phone number and email address; the caller’s own narrowest scope is now always intersected in with `$and`, and a code outside their branch returns an empty list. Soft-deleted members are excluded — the legacy raw `JOIN` included them.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | `national` \| `parish` \| `area` \| `zone` \| `province` \| `region` \| `sub-continent` \| `continent` | no | Hierarchy scope. Aliases are folded: `subcontinent`/`sub-cont` → `sub-continent`, `prov` → `province`, `country` → `national`, and `_`/space → `-`. |
| `scope_code` | string | no | Code at the requested level. Required for every scope except `national`. |

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | Scope, when not given as a path segment. |
| `scope_code` | string | no | Scope code, when not given as a path segment. |
| `type` | string | no | Legacy spelling of `scope`. |
| `type_code` | string | no | Legacy spelling of `scope_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Professionals in scope, ordered by profession then name. |
| `401` | Missing or invalid token. |
| `403` | The token carries no hierarchy claim. |
| `422` | Unknown scope (`{error: "invalid_scope", allowed_scopes: […]}`) or a missing scope code (`{error: "missing_scope_code", scope}`). Both bodies are the legacy bespoke shapes, not the Laravel validation envelope. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/employments/professionals/{scope}/{scope_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/employments/professionals/{scope}`

Professionals at a hierarchy level, with no scope code

The `{scope_code?}` half of the legacy optional-parameter route, declared separately because Express 5 no longer accepts `:param?`. A `?scope_code=` or `?type_code=` query parameter still satisfies the requirement, exactly as it did before; `scope=national` needs neither.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | `national` \| `parish` \| `area` \| `zone` \| `province` \| `region` \| `sub-continent` \| `continent` | no | Hierarchy scope. Aliases are folded: `subcontinent`/`sub-cont` → `sub-continent`, `prov` → `province`, `country` → `national`, and `_`/space → `-`. |
| `scope_code` | string | no | Code at the requested level. Required for every scope except `national`. |

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | Scope, when not given as a path segment. |
| `scope_code` | string | no | Scope code, when not given as a path segment. |
| `type` | string | no | Legacy spelling of `scope`. |
| `type_code` | string | no | Legacy spelling of `scope_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Professionals in scope. |
| `403` | The token carries no hierarchy claim. |
| `422` | Unknown scope, or a scope code is required and absent. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/employments/professionals/{scope}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/employments/professions-by-type`

Professionals by scope, read from the query string (legacy alias)

Marked "Backward-compatible legacy endpoint" in `routes/api.php:131`. Delegates to the same handler and accepts either the `scope`/`scope_code` or the older `type`/`type_code` query spellings.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | Scope, when not given as a path segment. |
| `scope_code` | string | no | Scope code, when not given as a path segment. |
| `type` | string | no | Legacy spelling of `scope`. |
| `type_code` | string | no | Legacy spelling of `scope_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Professionals in scope. |
| `403` | The token carries no hierarchy claim. |
| `422` | Unknown scope, or a scope code is required and absent. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/employments/professions-by-type" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/employments`

List every employment record in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned an `EmploymentCollection`, which is a Laravel `ResourceCollection`. Keys and key order match the MySQL column order, with `id` first and `_id` appended last.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Employment records for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/employments" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/employments/save`

Create or replace a member’s employment record

Keyed on `user_code`, one record per member. **201 when the body carries no `id`, 200 when it does** — a legacy quirk preserved verbatim, since `id` is never persisted and only switched the status code. Every column is written from the request, so an omitted optional field is stored as `null`; that is also legacy behaviour and makes a partial save destructive. `parish_code` must be the caller’s own parish: legacy trusted the body and matched on `user_code` alone, which allowed one parish to overwrite another’s record.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. The row is keyed on this — one employment record per member. | `"633a23f95e4119908a78fa9d"` |
| `parish_code` | string | **yes** | Must equal the caller’s own parish claim. Legacy trusted this field, which let any authenticated caller move an employment record into another parish. | `"211414"` |
| `employment_status` | string | **yes** | Free text — the column is a bare `varchar`, not an enum. | `"Employed"` |
| `profession` | string | **yes** | — | `"Software Engineer"` |
| `edu_qualification` | string | **yes** | — | `"BSc"` |
| `current_org` | string | no | Not in the legacy `FormRequest`, but assigned by the controller. | `"Redeemed Technologies Ltd"` |
| `org_phone_number` | string | no | — | `"02071234567"` |
| `org_address` | string | no | — | `"14 Sumner Road, Croydon"` |
| `e1` | string | no | Undocumented legacy spare column. | `""` |
| `e2` | string | no | Undocumented legacy spare column. | `""` |
| `e3` | string | no | Undocumented legacy spare column. | `""` |
| `id` | object | no | Never persisted. Its mere presence flips the success status from 201 to 200 (`EmploymentsController@post`, `:64`), so it is carried for status parity only. | `12` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "633a23f95e4119908a78fa9d",
  "parish_code": "211414",
  "employment_status": "Employed",
  "profession": "Software Engineer",
  "edu_qualification": "BSc"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "user_code": "633a23f95e4119908a78fa9d",
  "parish_code": "211414",
  "employment_status": "Employed",
  "profession": "Software Engineer",
  "edu_qualification": "BSc",
  "current_org": "Redeemed Technologies Ltd",
  "org_phone_number": "02071234567",
  "org_address": "14 Sumner Road, Croydon",
  "e1": "",
  "e2": "",
  "e3": "",
  "id": 12
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Saved; the body carried an `id`. |
| `201` | Saved; the body carried no `id`. |
| `403` | No parish claim, `parish_code` is not the caller’s own, or the record belongs to another parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/employments/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"633a23f95e4119908a78fa9d","parish_code":"211414","employment_status":"Employed","profession":"Software Engineer","edu_qualification":"BSc"}'
```

### `POST /api/v1/backend/employments/delete`

Soft-delete an employment record

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such record, which is legacy behaviour — clients branch on the text, not the status. The lookup is scoped to the caller’s parish; legacy’s `Employment::find($id)` was not, so an integer id alone could delete any parish’s record.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | **yes** | Legacy `employments.id`, or a 24-character MongoDB `_id` for post-migration rows. | `41` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": 41
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `403` | The token carries no parish claim. |
| `422` | `id` is missing. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/employments/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":41}'
```

### `POST /api/v1/backend/employments/restore`

Restore a soft-deleted employment record

Legacy validated nothing and called `->restore()` on a possibly-null model, so an unknown id was a 500. It is a 404 here, and the lookup is scoped to the caller’s parish.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | **yes** | Legacy `employments.id`, or a 24-character MongoDB `_id` for post-migration rows. | `41` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": 41
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |
| `422` | `id` is missing. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/employments/restore" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":41}'
```

### `POST /api/v1/backend/employments/{employment}/force-delete`

Permanently delete an employment record

The legacy route named an `EmploymentsController@forceDelete` method that was never written, on the controller or its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `employment` | string | **yes** | Legacy `employments.id`, or a 24-character MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/employments/{employment}/force-delete" \
  -H 'authtoken: $TOKEN'
```

---

## Events

### `GET /api/v1/backend/events`

List every event in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned an `EventCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `EventsController.php:24-25` is commented out and a bare `get()` took its place.

Soft-deleted events are excluded, matching the Eloquent global scope — 6 of the 230 live rows are trashed.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Events for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/events" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/events/save`

Create or update an event

The message is "Event **Created** Successfully" on both branches, the update included, while the status is `$request->get("id") ? 200 : 201`. So a truthy `id` answers **200** and still claims a creation; absent, `null`, `0`, `"0"` or `false` answers **201**. Clients branch on the text, so the misleading wording is a contract and is reproduced.

All eighteen columns are written from the request, so an omitted optional is stored as `null` — a save omitting `event_flyer_path` **erases a working media reference**. Preserved, because clients have relied on the resulting row shape since 2022; flagged as a product decision.

Seven of those columns are absent from `EventRequest` and therefore unvalidated: the six media fields and `id`. `notification_date` is `nullable` and so is **not** format-checked, unlike `start_date` and `end_date`; and no rule compares `end_date` against `start_date`, so an event that ends before it begins remains storable.

`parish_code` must be the caller’s own. Legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s event and *move* it in the process. A truthy `id` naming a **trashed** event is a 404 — legacy created a duplicate row at the same id instead, because `firstOrNew` carried the soft-delete scope.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `name` | string | **yes** | Event name. 189 distinct live values in **12 case-variant groups** (`let go a fishing` / `Let go a fishing`, `Birthday` / `BIRTHDAY`, `Test two` / `test two`). Never used as a query predicate by this controller — only assigned and returned — so no matcher applies to it. | `"Multicultural Sunday"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took it from the body with no tenancy check, so any authenticated caller could file an event into another parish; and `firstOrNew` matched the primary key alone, so a save could *move* another parish’s event. Both closed. | `"211774"` |
| `start_date` | string | **yes** | Stored as a `varchar`, not a date. `required\|date_format:Y-m-d`, reproduced including the calendar-validity check `DateTime::createFromFormat` performed — all 146 distinct live values already match, so nothing stored today is rejected. | `"2024-03-10"` |
| `start_time` | string | **yes** | Free-text clock time; the column is a `varchar` and the legacy rule is a bare `required`, so no format is imposed. 99 distinct live values, two of them carrying a leading zero. | `"6:30 PM"` |
| `end_date` | string | **yes** | `required\|date_format:Y-m-d`. **Not compared against `start_date`** — the legacy FormRequest has no `after_or_equal` rule, so an end date before the start date is storable and that stays true here. | `"2024-03-10"` |
| `end_time` | string | **yes** | Free-text clock time. 118 distinct live values. | `"9:01 PM"` |
| `occurrence` | string | **yes** | How often the event repeats. Exactly five live values — `Daily`, `Weekly`, `Monthly`, `Yearly`, `Once` — with **no case variants**. The legacy rule is a bare `required`, so no enum is imposed: a sixth value would be storable, as it was before. | `"Monthly"` |
| `sms` | object | no | Whether to send an SMS notification. Three live values — `Yes`, `No` and lower-case `yes` — i.e. **one case-variant group**. Nothing in this controller branches on it, so the variance is harmless here, but any future sender must compare it case-insensitively or it will skip the `yes` rows. Legacy rule is `["nullable"]`, so it is stored verbatim. | `"Yes"` |
| `notification_date` | object | no | When to notify. `nullable` in the FormRequest and therefore **not** format-checked, unlike `start_date` and `end_date` — but present with a valid `Y-m-d` value in all 230 live rows. Kept unvalidated so the endpoint does not become stricter than the API it replaces. | `"2024-03-09"` |
| `event_flyer` | object | no | Flyer URL. **Not validated** yet assigned. 30 of 230 rows have a value, all of them an `https://e-remittance-med…` URL. No URL rule is imposed, because the legacy app imposed none. | `"https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_event/banner2.jpg"` |
| `event_audio` | object | no | Audio URL. **Not validated** yet assigned. 9 of 230 rows have a value. | `"https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_event/worship.mp3"` |
| `event_video` | object | no | Video URL. **Not validated** yet assigned. 8 of 230 rows have a value. | `"https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_event/aao.mp4"` |
| `event_flyer_path` | object | no | Storage key for the flyer. **Not validated** yet assigned. | `"rpms_event/banner2.jpg"` |
| `event_audio_path` | object | no | Storage key for the audio. **Not validated** yet assigned. | `"rpms_event/worship.mp3"` |
| `event_video_path` | object | no | Storage key for the video. **Not validated** yet assigned. | `"rpms_event/aao.mp4"` |
| `e1` | object | no | Undocumented legacy spare column. Legacy rule is `["nullable"]` — a rule list with no actual constraint. **Null in all 230 live rows**, so no live data exercises it. | `null` |
| `e2` | object | no | Undocumented legacy spare column. Null in all 230 live rows. | `null` |
| `e3` | object | no | Undocumented legacy spare column. Null in all 230 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "name": "Multicultural Sunday",
  "parish_code": "211774",
  "start_date": "2024-03-10",
  "start_time": "6:30 PM",
  "end_date": "2024-03-10",
  "end_time": "9:01 PM",
  "occurrence": "Monthly"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "name": "Multicultural Sunday",
  "parish_code": "211774",
  "start_date": "2024-03-10",
  "start_time": "6:30 PM",
  "end_date": "2024-03-10",
  "end_time": "9:01 PM",
  "occurrence": "Monthly",
  "sms": "Yes",
  "notification_date": "2024-03-09",
  "event_flyer": "https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_event/banner2.jpg",
  "event_audio": "https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_event/worship.mp3",
  "event_video": "https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_event/aao.mp4",
  "event_flyer_path": "rpms_event/banner2.jpg",
  "event_audio_path": "rpms_event/worship.mp3",
  "event_video_path": "rpms_event/aao.mp4",
  "e1": {},
  "e2": {},
  "e3": {},
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no live event in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/events/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"name":"Multicultural Sunday","parish_code":"211774","start_date":"2024-03-10","start_time":"6:30 PM","end_date":"2024-03-10","end_time":"9:01 PM","occurrence":"Monthly"}'
```

### `POST /api/v1/backend/events/delete`

Soft-delete an event

Answers 200 with `{"message": "Event not found: Provide the correct parameter"}` when there is no such event, which is legacy behaviour — clients branch on the text, not the status.

The id comes from the **body**; this route has no path segment in `api.php`. The lookup is scoped to the caller’s parish; legacy’s `Event::find($id)` was not, so an integer id alone could soft-delete any parish’s event.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `events.id`, or a 24-character MongoDB `_id` for a row created since. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/events/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"5"}'
```

### `POST /api/v1/backend/events/{event}/restore`

Restore a soft-deleted event

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, which the legacy handler ignored entirely — `$request->get("id")` never consults route parameters, so the documented route only ever worked with the id in the body.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `event` | string | **yes** | Legacy `events.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `events.id`. Unvalidated, exactly as before. Wins over the `{event}` path segment, which the legacy handler ignored entirely. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such event in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/events/{event}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/events/{event}/force-delete`

Permanently delete an event

The legacy route named an `EventsController@forceDelete` method that was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `event` | string | **yes** | Legacy `events.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such event in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/events/{event}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/events/{event}/{parish}`

Read one event

**Security fix.** The legacy handler was `Event::where("id", $id)->where("parish_code", $parish_code)->get()` with the parish taken entirely from the URL — no `getUserDetails`, no token read, no comparison against the caller’s claim. Any caller could read another parish’s events by typing its code into the path. The segment is retained because the path is part of the contract, but it must now name the caller’s own parish and the query is scoped by the **token**. A mismatch is a **403**, not an empty 200, per `FOUNDATION_CONTRACT.md` §3c.

Returns a `{"data": […]}` collection of zero or one event. A non-numeric `{event}` yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `event` | string | **yes** | Legacy `events.id`, or a MongoDB `_id`. |
| `parish` | string | **yes** | Must be the caller’s own `parish_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one event. |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `{parish}` is not the caller’s own. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/events/{event}/{parish}" \
  -H 'authtoken: $TOKEN'
```

---

## Expenditures

### `GET /api/v1/backend/expenditure`

List every expenditure in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned an `ExpenditureCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `ExpendituresController.php:24-26` is commented out and a bare `get()` took its place, so the whole parish history comes back as a flat array.

`amount` is returned as the **original string**. Every money column in this database is `varchar`, 99 of the 873 live amounts carry decimals and one carries a leading zero, and the `amount_numeric` mirror that exists for aggregation is never exposed.

Note the singular path — `api.php:329` registers `expenditure` for the collection while the delete/restore routes are plural.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Expenditure records for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/expenditure" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/expenditure/viewByMonth`

List the caller’s parish expenditures for one month

`month` and `year` are both `required` and neither is format-checked, so a missing one is a 422 exactly as before.

**Deliberate divergence:** `month` matches every spelling of the named month, case-insensitively, rather than the exact bytes. The live column holds 14 spellings for 12 months and parish `211716` wrote both `Sep`/`September` and `Oct`/`October` in 2024 — so `?month=Oct` returned only part of its own October under the legacy exact match, answering 200 while doing it. This matches what `attendances` and `hf-coordinator` already do over their own `month` columns, so the three endpoints cannot disagree about what `Oct` means.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `month` | string | **yes** | Month as stored. Matched against every spelling of the named month, case-insensitively — parish `211716` wrote both `Sep`/`September` and `Oct`/`October` in 2024, so the legacy exact match returned only part of its own month. |
| `year` | string | **yes** | Four-digit year as stored. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Expenditure records for the parish-month. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `month` or `year` missing. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/expenditure/viewByMonth" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/expenditure/save`

Create or update an expenditure record

One message — "Expenditure Record Saved Successfully" — and the status is `$request->get("id") ? 200 : 201`. So a PHP-falsy `id` (`0`, `"0"`, `false`, or `""` which the global `ConvertEmptyStringsToNull` middleware folded to null) **creates** and answers **201**, while a truthy one updates and answers **200**. Reproduced verbatim.

All thirteen columns are written from the request, so an omitted optional is stored as `null`: a partial save is destructive. That is preserved — clients have relied on the resulting row shape since 2022.

`parish_code` must be the caller’s own. Legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s record and *move* it in the process. `amount` is stored as the string sent, byte for byte, with a numeric mirror populated beside it for aggregation.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `expenditure_type` | string | **yes** | Free text, not an enum. The live column holds **383 distinct values in 44 case-variant groups** (`chair`/`CHAIR`/`Chair`, `transport`/`TRANSPORT`/`Transport`) — evidence that no catalogue was ever enforced, so imposing one now would reject values the API stores today. Never used as a query predicate by this controller, only assigned and returned. | `"transport"` |
| `amount` | string | **yes** | A **string**, because the column is `varchar(191)` — there is not one DECIMAL or FLOAT column in this database and no currency field. 99 of the 873 live rows carry decimals (`50000.00`, `326850.03`) and one carries a leading zero, so the bytes are preserved exactly as sent and reads return them verbatim. Legacy rule is `required\|numeric`; sending `""` or `null` is a 422, which is what `numeric` did once `ConvertEmptyStringsToNull` had run. | `"4500"` |
| `description` | string | **yes** | The only `NOT NULL` column on the table. 610 distinct live values in **40 case-variant groups** (`GIFT`/`gift`, `Church building`/`Church Building`). | `"Uber for new member"` |
| `recipient` | string | **yes** | Who was paid. 608 distinct live values in **40 case-variant groups** (`CHURCH ADMIN`/`Church Admin`/`Church admin`), and six carry a leading zero. | `"Church Admin"` |
| `receipt_date` | string | **yes** | Stored as a `varchar`, not a date. `required\|date_format:Y-m-d` in the legacy `FormRequest`, reproduced including the calendar-validity check `DateTime::createFromFormat` performed — all 215 distinct live values already match. | `"2024-03-07"` |
| `month` | string | **yes** | Free text. The live column holds both `Sep` and `September`, and both `Oct` and `October` — in the *same* parish and year (`211716`, 2024). Stored verbatim; the `viewByMonth` filter folds the spellings when reading. | `"Mar"` |
| `year` | string | **yes** | Four-digit year as a string — the column is `varchar`, not `int`. | `"2024"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took this straight from the body with no tenancy check whatsoever, so any authenticated caller could file an expenditure into another parish — or, on the update path, move an existing record out of its own. | `"211716"` |
| `expenditure_mode` | string | **yes** | Payment method. Four values in the live data (`Cash`, `Cheque`, `Transfer`, `Card`) but the legacy rule is a bare `required`, so no enum is imposed. | `"Transfer"` |
| `reference` | string | **yes** | Payment reference. `required` despite 561 distinct live values including `--` and `1111`; 146 carry a leading zero and there are **19 case-variant groups** (`Bank`/`BANK`, `Cash`/`CASH`/`cash`). | `"REF/MN/04509832/2022"` |
| `e1` | object | no | Undocumented legacy spare column. Legacy rule is `["nullable"]` — a rule list with no actual constraint, so anything is accepted. **Null in all 873 live rows**; this field has never been written, so no live data exercises it. | `null` |
| `e2` | object | no | Undocumented legacy spare column. Null in all 873 live rows. | `null` |
| `e3` | object | no | Undocumented legacy spare column. Null in all 873 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all** — `ExpenditureRequest` declares no rule for it. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "expenditure_type": "transport",
  "amount": "4500",
  "description": "Uber for new member",
  "recipient": "Church Admin",
  "receipt_date": "2024-03-07",
  "month": "Mar",
  "year": "2024",
  "parish_code": "211716",
  "expenditure_mode": "Transfer",
  "reference": "REF/MN/04509832/2022"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "expenditure_type": "transport",
  "amount": "4500",
  "description": "Uber for new member",
  "recipient": "Church Admin",
  "receipt_date": "2024-03-07",
  "month": "Mar",
  "year": "2024",
  "parish_code": "211716",
  "expenditure_mode": "Transfer",
  "reference": "REF/MN/04509832/2022",
  "e1": {},
  "e2": {},
  "e3": {},
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no record in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/expenditure/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"expenditure_type":"transport","amount":"4500","description":"Uber for new member","recipient":"Church Admin","receipt_date":"2024-03-07","month":"Mar","year":"2024","parish_code":"211716","expenditure_mode":"Transfer","reference":"REF/MN/04509832/2022"}'
```

### `POST /api/v1/backend/expenditures/delete`

Soft-delete an expenditure record

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such record, which is legacy behaviour — clients branch on the text, not the status.

The id comes from the **body**; this route has no path segment in `api.php`, which matches the fact that `$request->get("id")` never consults route parameters in Laravel. The lookup is scoped to the caller’s parish; legacy’s `Expenditure::find($id)` was not, so an integer id alone could soft-delete any parish’s record.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `expenditures.id`, or a 24-character MongoDB `_id` for a new row. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/expenditures/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"5"}'
```

### `POST /api/v1/backend/expenditures/{expenditure}/restore`

Restore a soft-deleted expenditure record

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, which the legacy handler ignored entirely.

`deleted_at` is null in all 873 live rows, so no migrated data exercises this path — it is covered by fixtures only.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `expenditure` | string | **yes** | Legacy `expenditures.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `expenditures.id`. Unvalidated, exactly as before. Wins over the `{expenditure}` path segment, which the legacy handler ignored entirely. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/expenditures/{expenditure}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/expenditures/{expenditure}/force-delete`

Permanently delete an expenditure record

The legacy route named an `ExpendituresController@forceDelete` method that was never written, on the controller or its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `expenditure` | string | **yes** | Legacy `expenditures.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/expenditures/{expenditure}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/expenditure/{expenditure}`

Read one expenditure record from the caller’s parish

The route names `ExpendituresController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row. A non-numeric reference yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

The parish comes from the **token**, not from the URL.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `expenditure` | string | **yes** | Legacy `expenditures.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one expenditure record. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/expenditure/{expenditure}" \
  -H 'authtoken: $TOKEN'
```

---

## First Timers

### `GET /api/v1/backend/firsttimers`

List every first-timer record in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned a `FirstTimerCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `FirstTimersController.php:25-26` is commented out and a bare `get()` took its place, so the whole parish history comes back as a flat array.

**Responses**

| Status | Meaning |
|---|---|
| `200` | First-timer records for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/firsttimers" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/firsttimers/save`

Create or update a first-timer intake record

Absent or PHP-falsy `id` (`null`, `""`, `0`, `"0"`, `false`) ⇒ create, **201**, with a freshly generated `ref` of the form `FT` + nine characters from a no-confusable alphabet + the Unix timestamp in seconds. Any truthy `id` ⇒ update that record **within the caller’s parish**, **200**, leaving `ref` untouched. The message is the same either way — unlike `tithings`, the legacy handler builds one string.

`type` defaults through PHP’s `?:`, so `""`, `"0"`, `0` and `false` all store `Parish`, not just a missing key.

Every column is written from the request, so an omitted optional is stored as `null`: a partial save is destructive. `officer_user_code` and `updated_by` are the exceptions — the legacy handler never assigned them, so an update leaves whatever is stored.

When `hf_attendance_id` is sent, the referenced attendance’s `first_timer_input` is recounted. The trigger is legacy’s loose `!= null`, so integer `0` skips the recount while the string `"0"` performs it. Both the count and the attendance lookup are now scoped to the caller’s parish; legacy scoped neither, so the count included other parishes’ visitors and the update could touch another parish’s attendance row.

`parish_code` must be the caller’s own — legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s record and move it in the process.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `first_name` | string | **yes** | Visitor’s first name. | `"Adaeze"` |
| `last_name` | string | **yes** | Visitor’s last name. | `"Okoro"` |
| `gender` | string | **yes** | Free text, not an enum — the live column holds only `Male` and `Female`, but nothing constrains it. | `"Female"` |
| `service_date` | string | **yes** | Date of the service attended. A **string**, because the column is `varchar(191)`. All 295 live rows happen to be `YYYY-MM-DD`, but the legacy rule is a bare `required`, so no format is enforced and imposing one would reject payloads the API accepts today. | `"2024-03-05"` |
| `state` | string | **yes** | County/state of residence. | `"Greater London"` |
| `country` | string | **yes** | Country of residence. | `"United Kingdom"` |
| `address` | string | **yes** | Street address. | `"14 Grove Road"` |
| `landmark` | string | **yes** | Nearest landmark. | `"Opposite the library"` |
| `phone` | string | **yes** | Digits only — the legacy rule is `required\|numeric`, so a `+44` prefix or any spacing is a 422. All 295 live rows satisfy it. | `"07700900123"` |
| `email` | string | **yes** | Validated as a bare `required`, **not** as an email address — `FirstTimerRequest` declares no `email` rule. Live values include `OLAWAL@GMAIL.COM` and `JB@gmail.com`, so case is stored exactly as sent. | `"adaeze@example.com"` |
| `postcode` | string | **yes** | Postcode, free text. | `"SE1 7AB"` |
| `service_type` | string | **yes** | Which service the visitor attended. Free text: the eight live values are weekday names plus `House Fellowship`. | `"Sunday"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took this straight from the body with no tenancy check, so any authenticated caller could file a first-timer into another parish — or, on the update path, move an existing record out of its own. | `"211716"` |
| `whatsapp` | object | no | Digits only, and **not** nullable: `FirstTimerRequest` declares `numeric` with no `nullable`, so omitting the key passes while sending `null` or `""` is a 422. That is Laravel behaviour, not an addition — `ConvertEmptyStringsToNull` turned `""` into null before the `numeric` rule ran, and `is_numeric(null)` is false. | `"07700900123"` |
| `altar_call` | object | no | Whether the visitor answered the altar call. Live values are `Yes` and `No`. | `"Yes"` |
| `facebook` | object | no | Facebook handle. Null in all 295 live rows. | `null` |
| `instagram` | object | no | Instagram handle. Null in all 295 live rows. | `null` |
| `preferred_contact` | object | no | How the visitor prefers to be contacted. Set in all 295 live rows. | `"Phone"` |
| `prayer_point` | object | no | Free-text prayer request. `longtext` in MySQL, so no length limit applies. | `"Please pray for my family."` |
| `service_feedback` | object | no | Feedback on the service attended. | `"Excellent"` |
| `improvement` | object | no | What the visitor would like improved. | `"More parking"` |
| `returning` | object | no | Whether the visitor intends to return. **The live column is `returning`** — the 2022 migration calls it `return`, which is stale (`FOUNDATION_CONTRACT.md` §5). Null in all 295 live rows, so nothing has ever been recorded here. | `"Yes"` |
| `f1` | object | no | Undocumented legacy spare column. Null in all 295 live rows. | `null` |
| `f2` | object | no | Undocumented legacy spare column. Null in all 295 live rows. | `null` |
| `f3` | object | no | Undocumented legacy spare column. Null in all 295 live rows. | `null` |
| `type` | object | no | Where the visitor was received. `$request->get("type") ?: "Parish"`, so any PHP-falsy value — absent, null, `""`, `"0"`, `0`, `false` — stores `Parish`. Live values are `Parish`, `parish` and `HouseFellowship`; the column is only ever assigned, never compared, so the case variance is preserved rather than folded. | `"Parish"` |
| `followee_code` | object | no | The member being followed up, as a `members.user_code`. Null in all 295 live rows. | `null` |
| `follower_code` | object | no | The member doing the following up. Null in all 295 live rows. | `null` |
| `city` | object | no | City of residence. | `"London"` |
| `county` | object | no | County of residence. Null in all 295 live rows. | `null` |
| `date_of_followup` | object | no | When the follow-up call happened, as a string. Null in all 295 live rows — the entire follow-up half of this table has never been written to. | `null` |
| `follow_up_report` | object | no | Free-text follow-up outcome (`longtext`). Null in all 295 live rows. | `null` |
| `communication_medium` | object | no | How the follow-up was made. Null in all 295 live rows. | `null` |
| `hf_attendance_id` | object | no | Legacy `attendances.id` of the house-fellowship return this visitor was counted on. **Not validated at all** — `FirstTimerRequest` declares `f1` twice and the first declaration (`'f1' => ['hf_attendance_id']`) is overwritten by the second, so the rule intended for this field never existed. Stored into an `int(11)` column, so MySQL coerced whatever arrived; the same coercion is applied here. Sending it triggers a recount of the attendance’s `first_timer_input`. | `279` |
| `id` | object | no | Never persisted and not validated. Absent, `null`, `""`, `0`, `"0"` or `false` ⇒ create, answering **201** with a freshly generated `ref`. Any truthy value ⇒ update the record with that legacy id **within the caller’s parish**, answering **200** and leaving `ref` alone. Unlike `tithings`, the message is the same either way — `FirstTimersController@post` builds one string. | `"279"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "first_name": "Adaeze",
  "last_name": "Okoro",
  "gender": "Female",
  "service_date": "2024-03-05",
  "state": "Greater London",
  "country": "United Kingdom",
  "address": "14 Grove Road",
  "landmark": "Opposite the library",
  "phone": "07700900123",
  "email": "adaeze@example.com",
  "postcode": "SE1 7AB",
  "service_type": "Sunday",
  "parish_code": "211716"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "first_name": "Adaeze",
  "last_name": "Okoro",
  "gender": "Female",
  "service_date": "2024-03-05",
  "state": "Greater London",
  "country": "United Kingdom",
  "address": "14 Grove Road",
  "landmark": "Opposite the library",
  "phone": "07700900123",
  "email": "adaeze@example.com",
  "postcode": "SE1 7AB",
  "service_type": "Sunday",
  "parish_code": "211716",
  "whatsapp": "07700900123",
  "altar_call": "Yes",
  "facebook": {},
  "instagram": {},
  "preferred_contact": "Phone",
  "prayer_point": "Please pray for my family.",
  "service_feedback": "Excellent",
  "improvement": "More parking",
  "returning": "Yes",
  "f1": {},
  "f2": {},
  "f3": {},
  "type": "Parish",
  "followee_code": {},
  "follower_code": {},
  "city": "London",
  "county": {},
  "date_of_followup": {},
  "follow_up_report": {},
  "communication_medium": {},
  "hf_attendance_id": 279,
  "id": "279"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no record in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/firsttimers/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"first_name":"Adaeze","last_name":"Okoro","gender":"Female","service_date":"2024-03-05","state":"Greater London","country":"United Kingdom","address":"14 Grove Road","landmark":"Opposite the library","phone":"07700900123","email":"adaeze@example.com","postcode":"SE1 7AB","service_type":"Sunday","parish_code":"211716"}'
```

### `POST /api/v1/backend/firsttimers/delete`

Soft-delete a first-timer record

The id comes from the **body** and is required — this route carries no path segment, so legacy’s `id => required|numeric` is genuinely reachable and a missing id is a 422.

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such record, which is legacy behaviour — clients branch on the text, not the status. The lookup is scoped to the caller’s parish; legacy’s `FirstTimer::find($id)` was not, so an integer id alone could soft-delete any parish’s record.

The referenced attendance’s `first_timer_input` is **not** recounted, matching legacy: a stored count keeps including rows that have since been trashed.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `first_timers.id`, or a 24-character MongoDB `_id`. | `"279"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "279"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | The body `id` is missing or not numeric. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/firsttimers/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"279"}'
```

### `POST /api/v1/backend/firsttimers/{firsttimer}/restore`

Restore a soft-deleted first-timer record

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, because `$request->get('id')` never consults route parameters in Laravel — the segment has always been decorative and is honoured here as a fallback.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `firsttimer` | string | **yes** | Legacy `first_timers.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `first_timers.id`. Unvalidated, exactly as before. Wins over the `{firsttimer}` path segment, which the legacy handler ignored entirely. | `"279"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "279"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/firsttimers/{firsttimer}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/firsttimers/{firsttimer}/force-delete`

Permanently delete a first-timer record

The legacy route named a `FirstTimersController@forceDelete` method that was never written, on the controller or its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and reaching a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `firsttimer` | string | **yes** | Legacy `first_timers.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/firsttimers/{firsttimer}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/firsttimers/{firsttimer}`

Read one first-timer record from the caller’s parish

The route names `FirstTimersController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row. A non-numeric reference yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

The parish comes from the **token**, not from the URL. The three sibling `form` methods that do exist (`sermons`, `events`, `testimonies`) each took it from a second path segment and consulted no token at all, which made every one of them a reader of any parish’s records; that is not carried over.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `firsttimer` | string | **yes** | Legacy `first_timers.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one first-timer record. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/firsttimers/{firsttimer}" \
  -H 'authtoken: $TOKEN'
```

---

## Followups

### `GET /api/v1/backend/followups`

List every follow-up record in the caller’s parish

The legacy `FollowupCollection`, so the envelope is `{ "data": [...] }`. `parish_code` is matched exactly rather than case-insensitively: every live value is numeric, so there is no case for MySQL’s ci collation to have folded, and an exact match keeps the index usable.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Follow-up records for the caller’s parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followups" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/followups/assigned-members`

Every follow-up assignment in the caller’s parish, with both names resolved

Ordered by `created_at` descending. Each row carries the assignment columns plus the `followee` and `follower` member objects (five contact columns each) and the two `*_name` strings. An orphaned code — 5 of the live values match no member — resolves to `null` rather than failing the request.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Assignments for the caller’s parish. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followups/assigned-members" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/followups/hierarchy-report`

Follow-up report at a hierarchy level, within the caller’s own scope

The legacy handler never consulted the caller’s hierarchy, so `?type=national` returned every follow-up record in the database — names, phone numbers, prayer points and free-text reports across all 51 parishes — to any authenticated token. The requested scope is now intersected with the caller’s own under `$and`, so `national` means “everything you can reach” and a `type_code` outside your branch returns the empty report rather than another tenant’s data. Levels above `parish` are expanded into a parish set through `members`, because neither follow-up table carries a hierarchy column above `parish_code`.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | `national` \| `continent` \| `subContinent` \| `region` \| `province` \| `zone` \| `area` \| `parish` | **yes** | Hierarchy level to report on. `national` means the caller’s whole scope. |
| `type_code` | string | no | Code at the requested level. Required for every level except `national`, mirroring the legacy `required_unless:type,national`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The report, within the caller’s scope. |
| `403` | The token carries no hierarchy claim at all. |
| `422` | Unknown `type`, or `type_code` missing. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followups/hierarchy-report" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/followups/pending`

Assignments whose followee has no follow-up record yet

The exclusion list is compared case-insensitively. Both sides are database values, but they are written by code paths that disagree on case — `my-followup-record` upper-cased the code it stored — and SQL’s `NOT IN` folded that difference. Compared exactly, an already-followed-up member would stay pending forever. Assignments with a null `followee_code` are kept, which SQL did only when the exclusion list happened to be empty.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Pending assignments and their count. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followups/pending" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/followups/my-followees`

The follower’s view — the members the caller must follow up

Matches the caller’s own code against `follower_code`. The mirror of `my-followers`: a member can legitimately sit on either side of an assignment, so the two differ only in which column the caller is matched against. Each row reports `followed_up`, computed case-insensitively so it agrees with `followups/pending`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | The caller’s followees and their count. |
| `401` | The token carries no `user_code` or `id` claim. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followups/my-followees" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/followups/my-followers`

The followee’s view — who has been assigned to follow the caller up

Matches the caller’s own code against `followee_code`. The legacy payload carries no `parish_code` and no `followed_up` here, unlike `my-followees`; both omissions are kept.

**Responses**

| Status | Meaning |
|---|---|
| `200` | The caller’s followers and their count. |
| `401` | The token carries no `user_code` or `id` claim. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followups/my-followers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/followups/{followup}`

One follow-up record

The legacy handler was a byte-for-byte copy of `POST followups/save`: it ignored its path segment, **wrote** a row on a GET request, and answered with a save message. A GET that mutates is unsafe to retry, prefetch or cache, so this returns the record the path names, scoped to the caller’s parish. Accepts a legacy numeric id or a 24-character MongoDB `_id`.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `followup` | string | **yes** | Legacy `followups.id`, or a 24-character MongoDB `_id` for rows created after the migration (those carry no legacy id). |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The follow-up record. |
| `400` | The path segment is neither an id form. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followups/{followup}" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/followups/my-followup-record`

Log a follow-up against one of the caller’s own assigned followees

Anything else is a 403. Legacy upper-cased the incoming `followee_code` and compared it against `followup_members`, which stores lower-case hex; MySQL’s ci collation folded the difference, so a literal port would have rejected every legitimate request. The code stored on the new row is the assignment’s own rather than the upper-cased input, so the join key stays canonical.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `followee_code` | string | **yes** | `members.user_code` of the followee. Must already be assigned to the caller. | `"65e8373fa42f7669370eff3b"` |
| `type` | string | **yes** | — | `"member"` |
| `date_of_followup` | string | **yes** | `Y-m-d`. | `"2024-03-12"` |
| `communication_medium` | string | **yes** | — | `"Phone Call"` |
| `follow_up_report` | string | **yes** | — | `"Spoke on the phone; she plans to attend midweek service."` |
| `service_feedback` | string | no | — | `"The worship was uplifting"` |
| `prayer_point` | string | no | — | `"Healing for my mother"` |
| `returning` | string | no | — | `"Yes"` |
| `f1` | string | no | Free-form legacy slot. | `"y"` |
| `f2` | string | no | Free-form legacy slot. | `"n"` |
| `f3` | string | no | Free-form legacy slot. | `""` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "followee_code": "65e8373fa42f7669370eff3b",
  "type": "member",
  "date_of_followup": "2024-03-12",
  "communication_medium": "Phone Call",
  "follow_up_report": "Spoke on the phone; she plans to attend midweek service."
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "followee_code": "65e8373fa42f7669370eff3b",
  "type": "member",
  "date_of_followup": "2024-03-12",
  "communication_medium": "Phone Call",
  "follow_up_report": "Spoke on the phone; she plans to attend midweek service.",
  "service_feedback": "The worship was uplifting",
  "prayer_point": "Healing for my mother",
  "returning": "Yes",
  "f1": "y",
  "f2": "n",
  "f3": ""
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `201` | Record created; the response carries its `ref`. |
| `401` | The token carries no `user_code` or `id` claim. |
| `403` | The caller is not assigned to follow up that member. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/my-followup-record" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"followee_code":"65e8373fa42f7669370eff3b","type":"member","date_of_followup":"2024-03-12","communication_medium":"Phone Call","follow_up_report":"Spoke on the phone; she plans to attend midweek service."}'
```

### `POST /api/v1/backend/followups/save`

Create or update a follow-up record

Omit `id` to create (201); supply it to update (200). `parish_code` must fall inside the caller’s own hierarchy — legacy took it straight off the request body, which let any authenticated caller write into any parish. An unknown `id` is a 404: legacy’s `firstOrNew` plus explicit id assignment inserted a new row at a caller-chosen primary key. `ref` is issued once, on creation, and is not reissued on update.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | no | Legacy `followups.id`, or a 24-character MongoDB `_id`. Present updates and answers 200; absent creates and answers 201. | `"42"` |
| `type` | string | **yes** | — | `"first-timer"` |
| `first_name` | string | **yes** | First timer or member first name. | `"Carol"` |
| `last_name` | string | **yes** | — | `"Olaoni"` |
| `gender` | string | **yes** | — | `"Female"` |
| `altar_call` | string | no | — | `"Yes"` |
| `service_date` | string | **yes** | — | `"2024-03-05"` |
| `state` | string | **yes** | — | `"East Sussex"` |
| `country` | string | **yes** | — | `"GB"` |
| `address` | string | **yes** | — | `"12 Bridgewick Close, Lewes"` |
| `landmark` | string | **yes** | — | `"Opposite the library"` |
| `city` | string | no | — | `"Lewes"` |
| `county` | string | no | — | `"East Sussex"` |
| `phone` | string | **yes** | Legacy rule was `numeric`; relaxed to accept international formatting. | `"447805555616"` |
| `whatsapp` | string | no | — | `"447805555616"` |
| `email` | string | **yes** | Not format-checked: the legacy rule was a bare `required`, and live rows contain values that are not addresses. | `"carololaoni@example.test"` |
| `postcode` | string | **yes** | — | `"BN72EQ"` |
| `facebook` | string | no | — | `"carol.olaoni"` |
| `instagram` | string | no | — | `"@carololaoni"` |
| `preferred_contact` | string | no | — | `"Phone Call"` |
| `prayer_point` | string | no | — | `"Healing for my mother"` |
| `service_feedback` | string | no | — | `"The worship was uplifting"` |
| `improvement` | string | no | — | `"More parking"` |
| `returning` | string | no | Whether the person intends to return. The live column is `returning` — the 2022 migration calls it `return`, which never reached the database. | `"Yes"` |
| `service_type` | string | **yes** | — | `"Sunday Service"` |
| `parish_code` | string | **yes** | Parish the record belongs to. Must fall inside the caller’s own hierarchy — legacy took it from the body unchecked, which let any authenticated caller write into any parish. | `"211716"` |
| `followee_code` | string | no | — | `"65e8373fa42f7669370eff3b"` |
| `follower_code` | string | no | — | `"65e7daaea42f7669370efdcf"` |
| `date_of_followup` | string | no | — | `"2024-03-12"` |
| `follow_up_report` | string | no | — | `"Spoke on the phone; she plans to attend midweek service."` |
| `communication_medium` | string | no | — | `"Phone Call"` |
| `f1` | string | no | Free-form legacy slot. | `"y"` |
| `f2` | string | no | Free-form legacy slot. | `"n"` |
| `f3` | string | no | Free-form legacy slot. | `""` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "type": "first-timer",
  "first_name": "Carol",
  "last_name": "Olaoni",
  "gender": "Female",
  "service_date": "2024-03-05",
  "state": "East Sussex",
  "country": "GB",
  "address": "12 Bridgewick Close, Lewes",
  "landmark": "Opposite the library",
  "phone": "447805555616",
  "email": "carololaoni@example.test",
  "postcode": "BN72EQ",
  "service_type": "Sunday Service",
  "parish_code": "211716"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "42",
  "type": "first-timer",
  "first_name": "Carol",
  "last_name": "Olaoni",
  "gender": "Female",
  "altar_call": "Yes",
  "service_date": "2024-03-05",
  "state": "East Sussex",
  "country": "GB",
  "address": "12 Bridgewick Close, Lewes",
  "landmark": "Opposite the library",
  "city": "Lewes",
  "county": "East Sussex",
  "phone": "447805555616",
  "whatsapp": "447805555616",
  "email": "carololaoni@example.test",
  "postcode": "BN72EQ",
  "facebook": "carol.olaoni",
  "instagram": "@carololaoni",
  "preferred_contact": "Phone Call",
  "prayer_point": "Healing for my mother",
  "service_feedback": "The worship was uplifting",
  "improvement": "More parking",
  "returning": "Yes",
  "service_type": "Sunday Service",
  "parish_code": "211716",
  "followee_code": "65e8373fa42f7669370eff3b",
  "follower_code": "65e7daaea42f7669370efdcf",
  "date_of_followup": "2024-03-12",
  "follow_up_report": "Spoke on the phone; she plans to attend midweek service.",
  "communication_medium": "Phone Call",
  "f1": "y",
  "f2": "n",
  "f3": ""
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Record updated. |
| `201` | Record created. |
| `403` | `parish_code` is outside the caller’s scope. |
| `404` | `id` matched no record in that parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"type":"first-timer","first_name":"Carol","last_name":"Olaoni","gender":"Female","service_date":"2024-03-05","state":"East Sussex","country":"GB","address":"12 Bridgewick Close, Lewes","landmark":"Opposite the library","phone":"447805555616","email":"carololaoni@example.test","postcode":"BN72EQ","service_type":"Sunday Service","parish_code":"211716"}'
```

### `POST /api/v1/backend/followups/assignToFollowup`

Assign a follower to a followee

Omit `id` to create (201); supply it to update (200). Same two fixes as `save`: `parish_code` must be inside the caller’s scope, and an unknown `id` is a 404 rather than an insert at a caller-chosen primary key. The legacy path spelling — camelCase, unlike every sibling route — is preserved exactly.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | no | Legacy `followup_members.id`, or a 24-character MongoDB `_id`. | `"128"` |
| `followee_code` | string | **yes** | `members.user_code` of the person to be followed up. | `"65e8373fa42f7669370eff3b"` |
| `follower_code` | string | **yes** | `members.user_code` of the person who will do the following up. | `"65e7daaea42f7669370efdcf"` |
| `type` | string | **yes** | Assignment category. Live values include `member`, `Worker` and `parish`. | `"member"` |
| `parish_code` | string | **yes** | Parish the assignment belongs to. Legacy required `numeric` and that is kept, because every parish code in the live data is numeric. Must be inside the caller’s scope. | `"211716"` |
| `parish_designation` | string | no | — | `"Worker"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "followee_code": "65e8373fa42f7669370eff3b",
  "follower_code": "65e7daaea42f7669370efdcf",
  "type": "member",
  "parish_code": "211716"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "128",
  "followee_code": "65e8373fa42f7669370eff3b",
  "follower_code": "65e7daaea42f7669370efdcf",
  "type": "member",
  "parish_code": "211716",
  "parish_designation": "Worker"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Assignment updated. |
| `201` | Assignment created. |
| `403` | `parish_code` is outside the caller’s scope. |
| `404` | `id` matched no assignment in that parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/assignToFollowup" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"followee_code":"65e8373fa42f7669370eff3b","follower_code":"65e7daaea42f7669370efdcf","type":"member","parish_code":"211716"}'
```

### `POST /api/v1/backend/followups/getMembersAssignedToFollower`

The assignments held by one follower

`routes/api.php:156` has always pointed at a method that was never written, so this endpoint threw `BadMethodCallException` → 500 for its entire life. Implemented to its name: the assignments whose `follower_code` matches, within the caller’s parish.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `follower_code` | string | **yes** | `followup_members.follower_code` — the member doing the following up. | `"65e7daaea42f7669370efdcf"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "follower_code": "65e7daaea42f7669370efdcf"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Assignments held by that follower. |
| `403` | The token carries no parish claim. |
| `422` | `follower_code` is missing. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/getMembersAssignedToFollower" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"follower_code":"65e7daaea42f7669370efdcf"}'
```

### `POST /api/v1/backend/followups/getFollowerAssignedToMember`

The assignments naming one member as followee

The legacy handler filtered `follower_code` despite being named for the opposite direction. Both codes are accepted, each against its own correctly named column, so the endpoint answers its name for new callers and reproduces the legacy result exactly for old ones. Sending neither returns an empty list, matching legacy’s `WHERE follower_code = NULL`.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `followee_code` | string | no | `followup_members.followee_code` — the member being followed up. This is the code the endpoint name implies and the one new clients should send. | `"65e8373fa42f7669370eff3b"` |
| `follower_code` | string | no | Legacy field. The original handler filtered on `follower_code` here, so it is still honoured — sending it reproduces the legacy result exactly. | `"65e7daaea42f7669370efdcf"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "followee_code": "65e8373fa42f7669370eff3b",
  "follower_code": "65e7daaea42f7669370efdcf"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Matching assignments. |
| `403` | The token carries no parish claim. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/getFollowerAssignedToMember" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/followups/delete`

Soft-delete a follow-up record

The lookup is parish-scoped: legacy found the record by id alone, so any authenticated caller could soft-delete any parish’s record by guessing a small integer — ids run 1-94. A miss still answers 200 with the legacy message rather than a 404, because that body is what clients have always received for an unknown id.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `followups.id`, or a 24-character MongoDB `_id`. The legacy rule was `required\|numeric`; the hex form is additionally accepted because rows created after the migration have no legacy id and would otherwise be undeletable. | `"42"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "42"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or the legacy not-found message. |
| `400` | `id` is neither a numeric nor a 24-character id. |
| `403` | The token carries no parish claim. |
| `422` | `id` is missing. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"42"}'
```

### `POST /api/v1/backend/followups/{followup}/restore`

Restore a soft-deleted follow-up record

Legacy read the id from the request **body** while the route declares a `{followup}` segment, so the documented way to call this endpoint never worked; and it dereferenced the model unguarded, making an unknown id a 500. The path segment is now authoritative and a miss is a 404.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `followup` | string | **yes** | Legacy `followups.id`, or a 24-character MongoDB `_id` for rows created after the migration (those carry no legacy id). |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/{followup}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/followups/{followup}/force-delete`

Permanently delete a follow-up record

The legacy route named a `forceDelete` method that was never written, so this endpoint has always been a 500. Implemented as a hard delete, matching the sibling force-delete handlers. Resolves the record with soft-deleted rows included, so an already-deleted row can be purged.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `followup` | string | **yes** | Legacy `followups.id`, or a 24-character MongoDB `_id` for rows created after the migration (those carry no legacy id). |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/followups/{followup}/force-delete" \
  -H 'authtoken: $TOKEN'
```

---

## HF Coordinator

### `GET /api/v1/backend/hf-coordinator/centers`

Centres within the caller’s coordinator scope

The caller’s narrowest HF-coordinator role fixes the hierarchy level; that level is expanded into a set of parish codes via `members`, because `centers` carries no hierarchy column above `parish_code`. Ordered by centre name. Each row carries the leader’s `hf_leader_fname` and `hf_leader_lname`, both null when `hf_leader` matches no member. `?parish_code=` is intersected with the coordinator’s own parish set, never substituted for it, so a parish outside that set returns an empty list.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `per_page` | number | no | Rows per page. Clamped to 1..200; anything below 1 — including a non-numeric value, which PHP cast to 0 — falls back to 50. |
| `page` | number | no | Page number, 1-based. |
| `search` | string | no | Case-insensitive substring match across `name`, `center_code`, `parish_code` and `city`. `%` and `_` are matched literally — the legacy `LIKE` treated them as wildcards, so `search=%` returned every centre in scope. |
| `parish_code` | string | no | Narrows the list to one parish. Intersected with the coordinator’s own parish set, so a parish outside that set yields an empty list rather than another tenant’s centres. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Centres in scope, paginated. |
| `401` | Missing or invalid token. |
| `403` | The caller holds no HF-coordinator role. |
| `422` | The caller holds a coordinator role but no claim places them in the hierarchy. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/hf-coordinator/centers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/hf-coordinator/centers/{center_code}/attendances`

One centre’s attendance returns

Newest first by year, then by **month as a calendar position** — the legacy `ORDER BY month DESC` sorted month *names* alphabetically, so October preceded September. The `summary` totals are an aggregation that matches `deleted_at` explicitly, because `aggregate()` runs beneath the soft-delete middleware. `total`, `children`, `teenager` and `year` are `varchar` in the live table and are returned as strings; only the totals coerce.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `center_code` | string | **yes** | Matched case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did. |

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `per_page` | number | no | Rows per page. Clamped to 1..200; anything below 1 — including a non-numeric value, which PHP cast to 0 — falls back to 50. |
| `page` | number | no | Page number, 1-based. |
| `month` | string | no | Month name. A recognised month matches **every spelling** the column holds, case-insensitively (`Jan`, `jan`, `January`, `JANUARY`) — the live data stores 18 distinct values for 12 months. An unrecognised value falls back to a case-insensitive exact match. |
| `year` | string | no | Four-digit year. Stored as `varchar` and compared as a string. |
| `service_date_from` | string | no | Inclusive lower bound on `service_date`, compared as a string. |
| `service_date_to` | string | no | Inclusive upper bound on `service_date`, compared as a string. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Attendance rows for that centre, paginated. |
| `401` | Missing or invalid token. |
| `403` | Either the caller holds no HF-coordinator role, or the centre exists outside their scope. |
| `404` | No centre carries that `center_code`. |
| `422` | The caller holds a coordinator role but no claim places them in the hierarchy. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/hf-coordinator/centers/{center_code}/attendances" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/hf-coordinator/centers/{center_code}/visitations`

One centre’s visitation reports

Ordered like the attendance list. `avg_rating` averages `rating`, a `varchar` holding a percentage *with the sign*: the `%` is stripped and a value that is still not numeric is ignored rather than counted as zero, so one malformed rating cannot drag the average down. Null when no row in scope carries a rating, matching `AVG` over zero non-null values.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `center_code` | string | **yes** | Matched case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did. |

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `per_page` | number | no | Rows per page. Clamped to 1..200; anything below 1 — including a non-numeric value, which PHP cast to 0 — falls back to 50. |
| `page` | number | no | Page number, 1-based. |
| `month` | string | no | Month name. A recognised month matches **every spelling** the column holds, case-insensitively (`Jan`, `jan`, `January`, `JANUARY`) — the live data stores 18 distinct values for 12 months. An unrecognised value falls back to a case-insensitive exact match. |
| `year` | string | no | Four-digit year. Stored as `varchar` and compared as a string. |
| `visit_date_from` | string | no | Inclusive lower bound on `visit_date`, compared as a string. |
| `visit_date_to` | string | no | Inclusive upper bound on `visit_date`, compared as a string. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Visitation rows for that centre, paginated. |
| `401` | Missing or invalid token. |
| `403` | Either the caller holds no HF-coordinator role, or the centre exists outside their scope. |
| `404` | No centre carries that `center_code`. |
| `422` | The caller holds a coordinator role but no claim places them in the hierarchy. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/hf-coordinator/centers/{center_code}/visitations" \
  -H 'authtoken: $TOKEN'
```

---

## Incomes

### `GET /api/v1/backend/income`

List every income record in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned an `IncomeCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `IncomesController.php:23-25` is commented out and a bare `get()` took its place.

`amount` is returned as the **original string**. Every money column in this database is `varchar`, 74 of the 1,403 live amounts carry decimals and one carries a leading zero (`09808`), and the `amount_numeric` mirror that exists for aggregation is never exposed.

Note the singular path — `api.php:339` registers `income` for the collection.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Income records for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/income" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/income/viewByMonth`

List the caller’s parish income records for one month

`month` and `year` are both `required` and neither is format-checked, so a missing one is a 422 exactly as before.

**Deliberate divergence:** `month` matches every spelling of the named month, case-insensitively. The live column holds 15 spellings for 12 months and parish `211716` wrote both `Sep`/`September` and `Oct`/`October` in 2024, so `?month=Oct` returned only part of its own October under the legacy exact match. This matches what `attendances` and `hf-coordinator` already do over their own `month` columns.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `month` | string | **yes** | Month as stored. Matched against every spelling of the named month, case-insensitively — parish `211716` wrote both `Sep`/`September` and `Oct`/`October` in 2024, so the legacy exact match returned only part of its own month. |
| `year` | string | **yes** | Four-digit year as stored. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Income records for the parish-month. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `month` or `year` missing. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/income/viewByMonth" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/income/save`

Create or update an income record

One message — "Income Record Saved Successfully" — and the status is `$request->get("id") ? 200 : 201`. So a PHP-falsy `id` (`0`, `"0"`, `false`, or `""` which the global `ConvertEmptyStringsToNull` middleware folded to null) **creates** and answers **201**, while a truthy one updates and answers **200**. Reproduced verbatim.

**`amount` and `service_date` are not required**, unlike their expenditure twins: `IncomeRequest` gives them a bare `numeric` and a bare `date_format:Y-m-d`. Omitting the key stores a null; sending `""` or `null` is a 422, because the key is then *present* and Laravel runs the rule on it. That asymmetry is legacy behaviour, not an oversight to tidy.

All twelve columns are written from the request, so an omitted optional is stored as `null`: a partial save is destructive, and that is preserved. `parish_code` must be the caller’s own — legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s record and *move* it in the process.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `income_type` | string | **yes** | Free text, not an enum. The live column holds **377 distinct values in 53 case-variant groups** (`PLEDGE`/`Pledge`/`pledge`, `Offering`/`OFFERING`/`offering`) — the highest case-variance of any column in this batch, and evidence that no catalogue was ever enforced. Never used as a query predicate by this controller, only assigned and returned. | `"Offering"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took this straight from the body with no tenancy check, so any authenticated caller could file an income into another parish — or, on the update path, move an existing record out of its own. | `"211716"` |
| `service_date` | object | no | Stored as a `varchar`, not a date. **Not `required`** — the legacy rule is a bare `date_format:Y-m-d`, so omitting the key stores a null while sending `""` or `null` is a 422. All 236 distinct live values match the format. | `"2024-03-07"` |
| `amount` | object | no | A **string**, because the column is `varchar(191)` — there is not one DECIMAL or FLOAT column in this database and no currency field. 74 of the 1,403 live rows carry decimals (`333.0`, `2500000.00`, `20810.74`) and one carries a leading zero (`09808`), so the bytes are preserved exactly as sent and reads return them verbatim. **Not `required`**: the legacy rule is a bare `numeric`, so omitting the key stores a null while sending `""` or `null` is a 422. | `"2300"` |
| `month` | string | **yes** | Free text. The live column holds both `Aug` and `August`, `Sep` and `September`, `Oct` and `October` — and the last two pairs co-occur in parish `211716`’s 2024. Stored verbatim; the `viewByMonth` filter folds the spellings when reading. | `"Mar"` |
| `year` | string | **yes** | Four-digit year as a string — the column is `varchar`, not `int`. | `"2024"` |
| `income_mode` | string | **yes** | Receipt method. Five values in the live data (`Cash`, `Cheque`, `Transfer`, `Card` and the evidently-placeholder `Mode`) but the legacy rule is a bare `required`, so no enum is imposed — and imposing one would reject `Mode`, which is stored today. | `"Transfer"` |
| `description` | string | **yes** | `required`, unlike its expenditure twin’s optional spares. 727 distinct live values in **67 case-variant groups** (`LOVE`/`Love`, `GIFT`/`Gift`/`gift`) — the most of any column in this batch. | `"Bible sale"` |
| `reference` | object | no | Payment reference. Legacy rule is `["nullable"]`, so anything is accepted — note this is `required` on the expenditure side. 774 distinct live values including `--`; 237 carry a leading zero and there are **36 case-variant groups** (`Trf`/`trf`/`TRF`). | `"REF/MN/2231/2022"` |
| `i1` | object | no | Undocumented legacy spare column. Legacy rule is `["nullable"]` — a rule list with no actual constraint. **Null in all 1,403 live rows**; this field has never been written, so no live data exercises it. | `null` |
| `i2` | object | no | Undocumented legacy spare column. Null in all 1,403 live rows. | `null` |
| `i3` | object | no | Undocumented legacy spare column. Null in all 1,403 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all** — `IncomeRequest` declares no rule for it. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "income_type": "Offering",
  "parish_code": "211716",
  "month": "Mar",
  "year": "2024",
  "income_mode": "Transfer",
  "description": "Bible sale"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "income_type": "Offering",
  "parish_code": "211716",
  "service_date": "2024-03-07",
  "amount": "2300",
  "month": "Mar",
  "year": "2024",
  "income_mode": "Transfer",
  "description": "Bible sale",
  "reference": "REF/MN/2231/2022",
  "i1": {},
  "i2": {},
  "i3": {},
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no record in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/income/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"income_type":"Offering","parish_code":"211716","month":"Mar","year":"2024","income_mode":"Transfer","description":"Bible sale"}'
```

### `POST /api/v1/backend/incomes/delete`

Soft-delete an income record

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such record, which is legacy behaviour — clients branch on the text, not the status.

The id comes from the **body**; this route has no path segment in `api.php`. The lookup is scoped to the caller’s parish; legacy’s `Income::find($id)` was not, so an integer id alone could soft-delete any parish’s record.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `incomes.id`, or a 24-character MongoDB `_id` for a new row. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/incomes/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"5"}'
```

### `POST /api/v1/backend/incomes/{income}/restore`

Restore a soft-deleted income record

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, which the legacy handler ignored entirely.

`deleted_at` is null in all 1,403 live rows, so no migrated data exercises this path.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `income` | string | **yes** | Legacy `incomes.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `incomes.id`. Unvalidated, exactly as before. Wins over the `{income}` path segment, which the legacy handler ignored entirely. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/incomes/{income}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/incomes/{income}/force-delete`

Permanently delete an income record

The legacy route named an `IncomesController@forceDelete` method that was never written, on the controller or its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `income` | string | **yes** | Legacy `incomes.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/incomes/{income}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/income/{income}`

Read one income record from the caller’s parish

The route names `IncomesController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row. A non-numeric reference yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

The parish comes from the **token**, not from the URL.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `income` | string | **yes** | Legacy `incomes.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one income record. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/income/{income}" \
  -H 'authtoken: $TOKEN'
```

---

## Members

### `GET /api/v1/backend/members`

List every member of the caller’s parish

Wrapped in a `data` envelope because the legacy handler returned a `MemberCollection`. A caller whose token carries no parish claim gets **403**, not the empty list legacy returned: an empty list is indistinguishable from a parish with no members, so a misconfigured token looked like an empty parish.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Members of the caller’s parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/preview/{parish_code}`

Public preview of a parish: its members and its departments  
**🔓 Public — no token required.**

The only unauthenticated route in this module, matching legacy. The legacy handler was unreachable — two `dd()` calls halted it — so this is a reconstruction of what its surviving statements were assembling. See LEGACY_BUGS.md.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `parish_code` | string | **yes** | RCCG parish code. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Members and departments for the parish. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/preview/{parish_code}"
```

### `GET /api/v1/backend/members/getMembersByHierarchy/{type}/{type_code}`

Members at a hierarchy level, with all six profile relations

The requested scope is intersected with the caller’s own, so a code outside their part of the hierarchy returns nothing rather than another tenant’s members.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | string | **yes** | — |
| `type_code` | string | **yes** | — |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Members at the requested level. |
| `403` | The token carries no hierarchy claim. |
| `422` | Unknown hierarchy type. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/getMembersByHierarchy/{type}/{type_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/getMembersByHierarchy/{type}`

Members at a hierarchy level, falling back to the caller’s parish

The `{type_code?}` half of the legacy optional-parameter route, declared separately because Express 5 no longer accepts `:param?`.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | string | **yes** | — |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Members in the caller’s parish. |
| `422` | No type_code and no parish claim to fall back on. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/getMembersByHierarchy/{type}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/getVipMembersByHierarchy/{type}/{type_code}/{vip_status}`

VIP members at a hierarchy level, with all six profile relations

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | `national` \| `parish` \| `area` \| `zone` \| `province` \| `region` \| `subcontinent` \| `continent` \| `user_code` | **yes** | Hierarchy level. `national` means the caller’s whole scope; `user_code` looks up one member. |
| `type_code` | string | **yes** | Code at the requested level. |
| `vip_status` | string | **yes** | Matched exactly, not as a `LIKE`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | VIP members at the requested level. |
| `403` | The token carries no hierarchy claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/getVipMembersByHierarchy/{type}/{type_code}/{vip_status}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/userprofile/{user_code}`

One member with every profile relation loaded

Relations are appended after the member’s own columns, in the order the legacy `with()` call listed them: spiritual, employment, children, department, transfer, tithing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The member and their relations. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/userprofile/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/employmentprofile/{user_code}`

The member, if they have an employment record, with it attached

An array, and empty when no employment row exists — the legacy `whereHas` shape.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Zero or one member with `member_employment` set. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/employmentprofile/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/spiritualprofile/{user_code}`

The member, if they have spiritual records, with them attached

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Zero or one member with `member_spiritual` set. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/spiritualprofile/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/childrenprofile/{user_code}`

Members who name this user as father, mother or guardian

Returns rows from `members`, not from `children`: the legacy handler queried `members` for the three parent columns. `children` is a separate record of non-member dependants.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Members parented by this user code. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/childrenprofile/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/departmentprofile/{user_code}`

The member, if they have department records, with them attached

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Zero or one member with `member_department` set. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/departmentprofile/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/transferprofile/{user_code}`

The member, if they have transfer records, with them attached

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Zero or one member with `member_transferdetails` set. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/transferprofile/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/tithingProfile/{user_code}`

The member, if they have tithing records, with them attached

Tithing amounts are returned as the original strings. The `*_numeric` mirrors added for aggregation are not part of the legacy payload and are omitted.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Zero or one member with `member_tithing` set. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/tithingProfile/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/assignfollowup/{user_code}`

Add a member to the follow-up team

A GET that mutates. Legacy shape, preserved so existing clients keep working.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Member added to the follow-up team. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/assignfollowup/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/viewfollowup/{user_code}`

Whether a member is on the follow-up team

Legacy dereferenced the lookup without a null check, so an unknown user code was a 500. It is a 404 here.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | `Assigned Member` or `Member Unassigned`. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/viewfollowup/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/viewHFCenter/{user_code}`

The member’s house-fellowship centre, with that centre’s members attached

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Centres, each with a `member_center` array. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/viewHFCenter/{user_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/getTempMemberDetails/{parish_code}/{reg_code}`

Resolve an invitation link into its pending registration

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `parish_code` | string | **yes** | — |
| `reg_code` | string | **yes** | `members_temp.code` from the invitation link. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The pending `members_temp` row. |
| `403` | A member already exists with that email. |
| `404` | No pending invitation for that parish and code. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/getTempMemberDetails/{parish_code}/{reg_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/whatsapp-barcode/{parishCode}`

The stored WhatsApp barcode for a parish

`uploaded_by` is populated now. The legacy `updateOrCreate` omitted `user_code` from its payload, so the field it reported was permanently null.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `parishCode` | string | **yes** | RCCG parish code. Spelled `parishCode` on this route only — legacy path, kept. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Barcode URL and uploader. |
| `403` | That parish is outside the caller’s scope. |
| `404` | No barcode stored for that parish. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/whatsapp-barcode/{parishCode}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/HFCentreMembers/{centre_code}`

Members assigned to a house-fellowship centre

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `centre_code` | string | **yes** | House-fellowship centre code. Spelled `centre_code` in the legacy route. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Members of the centre. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/HFCentreMembers/{centre_code}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members/{member}`

One member by legacy id

Accepts the legacy integer `members.id` or, for rows created after the migration, a 24-character MongoDB `_id`.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `member` | string | **yes** | Legacy `members.id`, or a 24-character MongoDB `_id` for post-migration rows. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The member. |
| `400` | The identifier is neither an integer nor an ObjectId. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members/{member}" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/workers`

Members whose parish designation contains “worker”

Matched case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did for `LIKE`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Workers in the caller’s parish. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/workers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/followupteam`

Members on the caller parish’s follow-up team

**Responses**

| Status | Meaning |
|---|---|
| `200` | Follow-up team members. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/followupteam" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/birthdaylist`

Members whose birthday falls in the current month

Returns `{ birthdaylist, month }`, where `month` is the English month name.

**Responses**

| Status | Meaning |
|---|---|
| `200` | This month’s birthdays. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/birthdaylist" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/anniversarylist`

Members whose wedding anniversary falls in the current month

Returns `{ anniversarylist, month }`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | This month’s anniversaries. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/anniversarylist" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/getParentList`

Name and user code of every member in the caller’s scope

For parent pickers. Legacy returned every member in the database; it is scoped to the caller’s hierarchy here.

**Responses**

| Status | Meaning |
|---|---|
| `200` | `user_code`, `first_name`, `last_name` per member. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/getParentList" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/getMemberByRMIN`

Look up a member anywhere in the system by their RCCG membership code

Deliberately not tenancy-scoped: this is the cross-parish directory lookup transfers depend on. The projection is limited to the 17 columns the legacy handler selected.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `type` | `code` \| `full` | **yes** | `code` prefixes the value with `RCCG`; `full` expects the whole membership code. | `"code"` |
| `type_code` | string | **yes** | Digits only when `type=code`; the full `RCCG…` string when `type=full`. | `"1234567890"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "type": "code",
  "type_code": "1234567890"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | The member’s directory record. |
| `400` | `type` was neither `code` nor `full`. |
| `404` | No member with that code. |
| `422` | The code failed its format check. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/getMemberByRMIN" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"type":"code","type_code":"1234567890"}'
```

### `POST /api/v1/backend/members/checkDuplicateMail`

Whether an email address is already registered

System-wide by design: it guards registration, which is pre-tenancy.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `email` | string | **yes** | — | `"emeka.adeyemi@example.com"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "email": "emeka.adeyemi@example.com"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | The email is free. |
| `422` | The email is already registered. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/checkDuplicateMail" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"email":"emeka.adeyemi@example.com"}'
```

### `POST /api/v1/backend/members/changeMembershipStatus`

Set a member’s membership status

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | string | **yes** | — | `"633a23f95e4119908a78fa9d"` |
| `status` | string | **yes** | Stored verbatim — the column is a free-text `varchar(50)`, not an enum. | `"INACTIVE"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "633a23f95e4119908a78fa9d",
  "status": "INACTIVE"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Status updated. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/changeMembershipStatus" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"633a23f95e4119908a78fa9d","status":"INACTIVE"}'
```

### `POST /api/v1/backend/members/changeVipStatus`

Set a member’s VIP status

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | string | **yes** | — | `"633a23f95e4119908a78fa9d"` |
| `vip_status` | string | **yes** | — | `"Dignitary"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "633a23f95e4119908a78fa9d",
  "vip_status": "Dignitary"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | VIP status updated. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/changeVipStatus" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"633a23f95e4119908a78fa9d","vip_status":"Dignitary"}'
```

### `POST /api/v1/backend/members/save`

Create or update a member

`id` present updates and answers 200; absent creates and answers 201 — the legacy status rule. The membership code is allocated server-side. The 11-character prefix uniqueness rule on (parish_code, first_name, last_name) is enforced before the write.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy MySQL `members.id`. Present means update, absent means create — the branch `MembersController@post` takes on `isset($id)`. | `4211` |
| `title` | string | **yes** | — | `"Bro"` |
| `first_name` | string | **yes** | — | `"Emeka"` |
| `last_name` | string | **yes** | — | `"Adeyemi"` |
| `gender` | string | **yes** | — | `"Male"` |
| `marital_status` | string | **yes** | — | `"Married"` |
| `date_of_birth` | string | **yes** | Y-m-d | `"1984-03-17"` |
| `date_of_marriage` | string | no | Y-m-d | `"2011-08-06"` |
| `email` | string | no | — | `"emeka.adeyemi@example.com"` |
| `phone` | string | **yes** | — | `"07012345678"` |
| `whatsapp_phone` | string | no | — | `"07012345678"` |
| `church` | string | **yes** | — | `"RCCG Victory House"` |
| `country` | string | **yes** | — | `"United Kingdom"` |
| `country_code` | string | **yes** | Dialling code, stored without a leading +. | `"44"` |
| `state` | string | **yes** | — | `"Greater London"` |
| `city` | string | **yes** | — | `"Croydon"` |
| `address` | string | **yes** | — | `"14 Sumner Road"` |
| `parish_code` | string | **yes** | — | `"211414"` |
| `area_code` | string | **yes** | — | `"2114"` |
| `zone_code` | string | **yes** | — | `"211"` |
| `prov_code` | string | **yes** | — | `"21"` |
| `region_code` | string | **yes** | — | `"2"` |
| `subcont_code` | string | **yes** | — | `"UK"` |
| `cont_code` | string | **yes** | — | `"EU"` |
| `user_id` | string | **yes** | Becomes both `user_id` and `user_code` on create. | `"633a23f95e4119908a78fa9d"` |
| `password` | string | **yes** | Stored verbatim in `temp_password` and mailed to the member, as in legacy. | `"Chosen-Passcode-1"` |
| `passport` | string | no | — | `"rpms-images/members/passport/1712000000avatar.png"` |
| `spouse_present` | string | no | — | `"Yes"` |
| `spouse_usercode` | string | no | — | `"633a23f95e4119908a78fa9e"` |
| `child_present` | string | no | — | `"Yes"` |
| `father_usercode` | string | no | — | `"633a23f95e4119908a78fa9f"` |
| `mother_usercode` | string | no | — | `"633a23f95e4119908a78faa0"` |
| `guardian_usercode` | string | no | — | `"633a23f95e4119908a78faa1"` |
| `student` | string | no | — | `"No"` |
| `landmark` | string | no | — | `"Opposite the community hall"` |
| `county` | string | no | — | `"Surrey"` |
| `postcode` | string | no | — | `"CR0 3LG"` |
| `country_code_whatsapp` | string | no | — | `"44"` |
| `trccg_code` | string | no | — | `"RCCG1234567890"` |
| `dept_status` | string | no | — | `"Approved"` |
| `dept` | string | no | — | `"Choir"` |
| `parish_designation` | string | no | — | `"Worker"` |
| `ord_status` | string | no | — | `"Ordained"` |
| `year_last_ordained` | string | no | — | `"2019"` |
| `facebook` | string | no | Accepted as free text: Laravel’s `url` rule rejected the blank values clients actually send. | `"https://facebook.com/emeka.adeyemi"` |
| `instagram` | string | no | — | `"https://instagram.com/emeka.adeyemi"` |
| `twitter` | string | no | — | `"https://twitter.com/emeka_adeyemi"` |
| `m1` | string | no | Legacy spare column. | `null` |
| `m2` | string | no | Legacy spare column. | `null` |
| `m3` | string | no | Legacy spare column. | `null` |
| `parishPastorName` | string | **yes** | — | `"Pastor Ade Balogun"` |
| `parishName` | string | **yes** | — | `"RCCG Victory House"` |
| `parishAddress` | string | no | — | `"14 Sumner Road, Croydon"` |
| `continentName` | string | no | — | `"Europe"` |
| `subContinentName` | string | no | — | `"United Kingdom"` |
| `regionName` | string | no | — | `"Region 2"` |
| `provinceName` | string | no | — | `"Province 21"` |
| `zoneName` | string | no | — | `"Zone 211"` |
| `areaName` | string | no | — | `"Area 2114"` |
| `verificationUrl` | string | no | — | `"https://rpms.rccg.org/verify"` |
| `frontendUrl` | string | no | — | `"https://rpms.rccg.org"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "title": "Bro",
  "first_name": "Emeka",
  "last_name": "Adeyemi",
  "gender": "Male",
  "marital_status": "Married",
  "date_of_birth": "1984-03-17",
  "phone": "07012345678",
  "church": "RCCG Victory House",
  "country": "United Kingdom",
  "country_code": "44",
  "state": "Greater London",
  "city": "Croydon",
  "address": "14 Sumner Road",
  "parish_code": "211414",
  "area_code": "2114",
  "zone_code": "211",
  "prov_code": "21",
  "region_code": "2",
  "subcont_code": "UK",
  "cont_code": "EU",
  "user_id": "633a23f95e4119908a78fa9d",
  "password": "Chosen-Passcode-1",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": 4211,
  "title": "Bro",
  "first_name": "Emeka",
  "last_name": "Adeyemi",
  "gender": "Male",
  "marital_status": "Married",
  "date_of_birth": "1984-03-17",
  "date_of_marriage": "2011-08-06",
  "email": "emeka.adeyemi@example.com",
  "phone": "07012345678",
  "whatsapp_phone": "07012345678",
  "church": "RCCG Victory House",
  "country": "United Kingdom",
  "country_code": "44",
  "state": "Greater London",
  "city": "Croydon",
  "address": "14 Sumner Road",
  "parish_code": "211414",
  "area_code": "2114",
  "zone_code": "211",
  "prov_code": "21",
  "region_code": "2",
  "subcont_code": "UK",
  "cont_code": "EU",
  "user_id": "633a23f95e4119908a78fa9d",
  "password": "Chosen-Passcode-1",
  "passport": "rpms-images/members/passport/1712000000avatar.png",
  "spouse_present": "Yes",
  "spouse_usercode": "633a23f95e4119908a78fa9e",
  "child_present": "Yes",
  "father_usercode": "633a23f95e4119908a78fa9f",
  "mother_usercode": "633a23f95e4119908a78faa0",
  "guardian_usercode": "633a23f95e4119908a78faa1",
  "student": "No",
  "landmark": "Opposite the community hall",
  "county": "Surrey",
  "postcode": "CR0 3LG",
  "country_code_whatsapp": "44",
  "trccg_code": "RCCG1234567890",
  "dept_status": "Approved",
  "dept": "Choir",
  "parish_designation": "Worker",
  "ord_status": "Ordained",
  "year_last_ordained": "2019",
  "facebook": "https://facebook.com/emeka.adeyemi",
  "instagram": "https://instagram.com/emeka.adeyemi",
  "twitter": "https://twitter.com/emeka_adeyemi",
  "m1": "string",
  "m2": "string",
  "m3": "string",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House",
  "parishAddress": "14 Sumner Road, Croydon",
  "continentName": "Europe",
  "subContinentName": "United Kingdom",
  "regionName": "Region 2",
  "provinceName": "Province 21",
  "zoneName": "Zone 211",
  "areaName": "Area 2114",
  "verificationUrl": "https://rpms.rccg.org/verify",
  "frontendUrl": "https://rpms.rccg.org"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Member updated. |
| `201` | Member created. |
| `403` | The payload’s hierarchy is outside the caller’s scope. |
| `404` | `id` matched no member within the caller’s scope. |
| `409` | The 11-character name prefix is already taken. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"title":"Bro","first_name":"Emeka","last_name":"Adeyemi","gender":"Male","marital_status":"Married","date_of_birth":"1984-03-17","phone":"07012345678","church":"RCCG Victory House","country":"United Kingdom","country_code":"44","state":"Greater London","city":"Croydon","address":"14 Sumner Road","parish_code":"211414","area_code":"2114","zone_code":"211","prov_code":"21","region_code":"2","subcont_code":"UK","cont_code":"EU","user_id":"633a23f95e4119908a78fa9d","password":"Chosen-Passcode-1","parishPastorName":"Pastor Ade Balogun","parishName":"RCCG Victory House"}'
```

### `POST /api/v1/backend/members/saveTempMemberDetails`

An invited member completes their own registration

Validates the invitation before creating anything. Legacy created the member first and then dereferenced a possibly-missing invitation, orphaning the new row on failure.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy MySQL `members.id`. Present means update, absent means create — the branch `MembersController@post` takes on `isset($id)`. | `4211` |
| `title` | string | **yes** | — | `"Bro"` |
| `first_name` | string | **yes** | — | `"Emeka"` |
| `last_name` | string | **yes** | — | `"Adeyemi"` |
| `gender` | string | **yes** | — | `"Male"` |
| `marital_status` | string | **yes** | — | `"Married"` |
| `date_of_birth` | string | **yes** | Y-m-d | `"1984-03-17"` |
| `date_of_marriage` | string | no | Y-m-d | `"2011-08-06"` |
| `email` | string | no | — | `"invitee@example.com"` |
| `phone` | string | **yes** | — | `"07012345678"` |
| `whatsapp_phone` | string | no | — | `"07012345678"` |
| `church` | string | **yes** | — | `"RCCG Victory House"` |
| `country` | string | **yes** | — | `"United Kingdom"` |
| `country_code` | string | **yes** | Dialling code, stored without a leading +. | `"44"` |
| `state` | string | **yes** | — | `"Greater London"` |
| `city` | string | **yes** | — | `"Croydon"` |
| `address` | string | **yes** | — | `"14 Sumner Road"` |
| `parish_code` | string | **yes** | — | `"211414"` |
| `area_code` | string | **yes** | — | `"2114"` |
| `zone_code` | string | **yes** | — | `"211"` |
| `prov_code` | string | **yes** | — | `"21"` |
| `region_code` | string | **yes** | — | `"2"` |
| `subcont_code` | string | **yes** | — | `"UK"` |
| `cont_code` | string | **yes** | — | `"EU"` |
| `user_id` | string | **yes** | Becomes both `user_id` and `user_code` on create. | `"633a23f95e4119908a78fa9d"` |
| `password` | string | **yes** | Stored verbatim in `temp_password` and mailed to the member, as in legacy. | `"Chosen-Passcode-1"` |
| `passport` | string | no | — | `"rpms-images/members/passport/1712000000avatar.png"` |
| `spouse_present` | string | no | — | `"Yes"` |
| `spouse_usercode` | string | no | — | `"633a23f95e4119908a78fa9e"` |
| `child_present` | string | no | — | `"Yes"` |
| `father_usercode` | string | no | — | `"633a23f95e4119908a78fa9f"` |
| `mother_usercode` | string | no | — | `"633a23f95e4119908a78faa0"` |
| `guardian_usercode` | string | no | — | `"633a23f95e4119908a78faa1"` |
| `student` | string | no | — | `"No"` |
| `landmark` | string | no | — | `"Opposite the community hall"` |
| `county` | string | no | — | `"Surrey"` |
| `postcode` | string | no | — | `"CR0 3LG"` |
| `country_code_whatsapp` | string | no | — | `"44"` |
| `trccg_code` | string | no | — | `"RCCG1234567890"` |
| `dept_status` | string | no | — | `"Approved"` |
| `dept` | string | no | — | `"Choir"` |
| `parish_designation` | string | no | — | `"Worker"` |
| `ord_status` | string | no | — | `"Ordained"` |
| `year_last_ordained` | string | no | — | `"2019"` |
| `facebook` | string | no | Accepted as free text: Laravel’s `url` rule rejected the blank values clients actually send. | `"https://facebook.com/emeka.adeyemi"` |
| `instagram` | string | no | — | `"https://instagram.com/emeka.adeyemi"` |
| `twitter` | string | no | — | `"https://twitter.com/emeka_adeyemi"` |
| `m1` | string | no | Legacy spare column. | `null` |
| `m2` | string | no | Legacy spare column. | `null` |
| `m3` | string | no | Legacy spare column. | `null` |
| `parishPastorName` | string | **yes** | — | `"Pastor Ade Balogun"` |
| `parishName` | string | **yes** | — | `"RCCG Victory House"` |
| `parishAddress` | string | no | — | `"14 Sumner Road, Croydon"` |
| `continentName` | string | no | — | `"Europe"` |
| `subContinentName` | string | no | — | `"United Kingdom"` |
| `regionName` | string | no | — | `"Region 2"` |
| `provinceName` | string | no | — | `"Province 21"` |
| `zoneName` | string | no | — | `"Zone 211"` |
| `areaName` | string | no | — | `"Area 2114"` |
| `verificationUrl` | string | no | — | `"https://rpms.rccg.org/verify"` |
| `frontendUrl` | string | no | — | `"https://rpms.rccg.org"` |
| `temp_member_id` | string | **yes** | The `members_temp.code` from the invitation link. | `"9f1c1d2e-3b4a-4c5d-8e6f-7a8b9c0d1e2f"` |
| `consent_code` | string | **yes** | — | `"RCCG-DTP-A7K2M9P4QX"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "title": "Bro",
  "first_name": "Emeka",
  "last_name": "Adeyemi",
  "gender": "Male",
  "marital_status": "Married",
  "date_of_birth": "1984-03-17",
  "phone": "07012345678",
  "church": "RCCG Victory House",
  "country": "United Kingdom",
  "country_code": "44",
  "state": "Greater London",
  "city": "Croydon",
  "address": "14 Sumner Road",
  "parish_code": "211414",
  "area_code": "2114",
  "zone_code": "211",
  "prov_code": "21",
  "region_code": "2",
  "subcont_code": "UK",
  "cont_code": "EU",
  "user_id": "633a23f95e4119908a78fa9d",
  "password": "Chosen-Passcode-1",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House",
  "temp_member_id": "9f1c1d2e-3b4a-4c5d-8e6f-7a8b9c0d1e2f",
  "consent_code": "RCCG-DTP-A7K2M9P4QX"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": 4211,
  "title": "Bro",
  "first_name": "Emeka",
  "last_name": "Adeyemi",
  "gender": "Male",
  "marital_status": "Married",
  "date_of_birth": "1984-03-17",
  "date_of_marriage": "2011-08-06",
  "email": "invitee@example.com",
  "phone": "07012345678",
  "whatsapp_phone": "07012345678",
  "church": "RCCG Victory House",
  "country": "United Kingdom",
  "country_code": "44",
  "state": "Greater London",
  "city": "Croydon",
  "address": "14 Sumner Road",
  "parish_code": "211414",
  "area_code": "2114",
  "zone_code": "211",
  "prov_code": "21",
  "region_code": "2",
  "subcont_code": "UK",
  "cont_code": "EU",
  "user_id": "633a23f95e4119908a78fa9d",
  "password": "Chosen-Passcode-1",
  "passport": "rpms-images/members/passport/1712000000avatar.png",
  "spouse_present": "Yes",
  "spouse_usercode": "633a23f95e4119908a78fa9e",
  "child_present": "Yes",
  "father_usercode": "633a23f95e4119908a78fa9f",
  "mother_usercode": "633a23f95e4119908a78faa0",
  "guardian_usercode": "633a23f95e4119908a78faa1",
  "student": "No",
  "landmark": "Opposite the community hall",
  "county": "Surrey",
  "postcode": "CR0 3LG",
  "country_code_whatsapp": "44",
  "trccg_code": "RCCG1234567890",
  "dept_status": "Approved",
  "dept": "Choir",
  "parish_designation": "Worker",
  "ord_status": "Ordained",
  "year_last_ordained": "2019",
  "facebook": "https://facebook.com/emeka.adeyemi",
  "instagram": "https://instagram.com/emeka.adeyemi",
  "twitter": "https://twitter.com/emeka_adeyemi",
  "m1": "string",
  "m2": "string",
  "m3": "string",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House",
  "parishAddress": "14 Sumner Road, Croydon",
  "continentName": "Europe",
  "subContinentName": "United Kingdom",
  "regionName": "Region 2",
  "provinceName": "Province 21",
  "zoneName": "Zone 211",
  "areaName": "Area 2114",
  "verificationUrl": "https://rpms.rccg.org/verify",
  "frontendUrl": "https://rpms.rccg.org",
  "temp_member_id": "9f1c1d2e-3b4a-4c5d-8e6f-7a8b9c0d1e2f",
  "consent_code": "RCCG-DTP-A7K2M9P4QX"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `201` | Membership profile created. |
| `404` | The invitation code is not valid. |
| `409` | The 11-character name prefix is already taken. |
| `422` | Validation failed, or the email is already in use. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/saveTempMemberDetails" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"title":"Bro","first_name":"Emeka","last_name":"Adeyemi","gender":"Male","marital_status":"Married","date_of_birth":"1984-03-17","phone":"07012345678","church":"RCCG Victory House","country":"United Kingdom","country_code":"44","state":"Greater London","city":"Croydon","address":"14 Sumner Road","parish_code":"211414","area_code":"2114","zone_code":"211","prov_code":"21","region_code":"2","subcont_code":"UK","cont_code":"EU","user_id":"633a23f95e4119908a78fa9d","password":"Chosen-Passcode-1","parishPastorName":"Pastor Ade Balogun","parishName":"RCCG Victory House","temp_member_id":"9f1c1d2e-3b4a-4c5d-8e6f-7a8b9c0d1e2f","consent_code":"RCCG-DTP-A7K2M9P4QX"}'
```

### `POST /api/v1/backend/members/addTempLiterateMember`

Invite one person to complete their own registration

A `members` array in the body is rejected with 422 and a pointer to the bulk route.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `email` | string | **yes** | — | `"invitee@example.com"` |
| `first_name` | string | **yes** | — | `"Ngozi"` |
| `last_name` | string | **yes** | — | `"Okonkwo"` |
| `phone` | string | **yes** | — | `"07012345678"` |
| `phone_code` | string | no | Dialling code. | `"44"` |
| `frontendUrl` | string | **yes** | Base URL the invitation link is built from. | `"https://rpms.rccg.org"` |
| `parishPastorName` | string | **yes** | — | `"Pastor Ade Balogun"` |
| `parishName` | string | **yes** | — | `"RCCG Victory House"` |
| `user_id` | string | no | — | `"633a23f95e4119908a78fa9d"` |
| `parish_code` | string | no | — | `"211414"` |
| `area_code` | string | no | — | `"2114"` |
| `zone_code` | string | no | — | `"211"` |
| `prov_code` | string | no | Stored as `members_temp.province_code` — the column is spelled differently there. | `"21"` |
| `region_code` | string | no | — | `"2"` |
| `subcont_code` | string | no | — | `"UK"` |
| `cont_code` | string | no | — | `"EU"` |
| `continentName` | string | no | — | `"Europe"` |
| `subContinentName` | string | no | — | `"United Kingdom"` |
| `provinceName` | string | no | — | `"Province 21"` |
| `regionName` | string | no | — | `"Region 2"` |
| `zoneName` | string | no | — | `"Zone 211"` |
| `areaName` | string | no | — | `"Area 2114"` |
| `verificationUrl` | string | no | — | `"https://rpms.rccg.org/verify"` |
| `parishAddress` | string | no | — | `"14 Sumner Road, Croydon"` |
| `members` | object[] | no | Rejected with 422 on this endpoint — the legacy controller redirects bulk payloads to `members/bulkaddTempLiterateMember`. | — |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "email": "invitee@example.com",
  "first_name": "Ngozi",
  "last_name": "Okonkwo",
  "phone": "07012345678",
  "frontendUrl": "https://rpms.rccg.org",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "email": "invitee@example.com",
  "first_name": "Ngozi",
  "last_name": "Okonkwo",
  "phone": "07012345678",
  "phone_code": "44",
  "frontendUrl": "https://rpms.rccg.org",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House",
  "user_id": "633a23f95e4119908a78fa9d",
  "parish_code": "211414",
  "area_code": "2114",
  "zone_code": "211",
  "prov_code": "21",
  "region_code": "2",
  "subcont_code": "UK",
  "cont_code": "EU",
  "continentName": "Europe",
  "subContinentName": "United Kingdom",
  "provinceName": "Province 21",
  "regionName": "Region 2",
  "zoneName": "Zone 211",
  "areaName": "Area 2114",
  "verificationUrl": "https://rpms.rccg.org/verify",
  "parishAddress": "14 Sumner Road, Croydon",
  "members": []
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `201` | Invitation created. |
| `403` | The payload’s hierarchy is outside the caller’s scope. |
| `422` | Validation failed, or the email is already in use. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/addTempLiterateMember" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"email":"invitee@example.com","first_name":"Ngozi","last_name":"Okonkwo","phone":"07012345678","frontendUrl":"https://rpms.rccg.org","parishPastorName":"Pastor Ade Balogun","parishName":"RCCG Victory House"}'
```

### `POST /api/v1/backend/members/bulkaddTempLiterateMember`

Grade or send a batch of registration invitations

`action=check` grades every row’s email and writes nothing; `action=submit` sends the available ones and answers 207 when any row was skipped or failed. Spreadsheet upload is not ported — see LEGACY_BUGS.md.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `members` | object[] | **yes** | — | — |
| `action` | `check` \| `submit` | no | Defaults to `check` when `validate_only` is truthy, otherwise `submit` — the legacy default. | `"check"` |
| `validate_only` | object | no | Legacy alias that forces `action=check`. | `true` |
| `frontendUrl` | string | no | — | `"https://rpms.rccg.org"` |
| `parishPastorName` | string | no | — | `"Pastor Ade Balogun"` |
| `parishName` | string | no | — | `"RCCG Victory House"` |
| `parish_code` | string | no | — | `"211414"` |
| `area_code` | string | no | — | `"2114"` |
| `zone_code` | string | no | — | `"211"` |
| `prov_code` | string | no | — | `"21"` |
| `region_code` | string | no | — | `"2"` |
| `subcont_code` | string | no | — | `"UK"` |
| `cont_code` | string | no | — | `"EU"` |
| `phone_code` | string | no | — | `"44"` |
| `continentName` | string | no | — | `"Europe"` |
| `subContinentName` | string | no | — | `"United Kingdom"` |
| `provinceName` | string | no | — | `"Province 21"` |
| `regionName` | string | no | — | `"Region 2"` |
| `zoneName` | string | no | — | `"Zone 211"` |
| `areaName` | string | no | — | `"Area 2114"` |
| `verificationUrl` | string | no | — | `"https://rpms.rccg.org/verify"` |
| `parishAddress` | string | no | — | `"14 Sumner Road, Croydon"` |
| `user_id` | string | no | — | `"633a23f95e4119908a78fa9d"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "members": []
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "members": [],
  "action": "check",
  "validate_only": true,
  "frontendUrl": "https://rpms.rccg.org",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House",
  "parish_code": "211414",
  "area_code": "2114",
  "zone_code": "211",
  "prov_code": "21",
  "region_code": "2",
  "subcont_code": "UK",
  "cont_code": "EU",
  "phone_code": "44",
  "continentName": "Europe",
  "subContinentName": "United Kingdom",
  "provinceName": "Province 21",
  "regionName": "Region 2",
  "zoneName": "Zone 211",
  "areaName": "Area 2114",
  "verificationUrl": "https://rpms.rccg.org/verify",
  "parishAddress": "14 Sumner Road, Croydon",
  "user_id": "633a23f95e4119908a78fa9d"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Availability check completed. |
| `201` | Every row was sent. |
| `207` | Some rows were skipped or failed. |
| `422` | Invalid action, or missing shared invite fields. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/bulkaddTempLiterateMember" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"members":[]}'
```

### `POST /api/v1/backend/members/delete`

Soft-delete a member

Answers 200 with an explanatory message when the id matches nothing — legacy behaviour, preserved because clients branch on the message text rather than the status.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | **yes** | Legacy `members.id`. A 24-character MongoDB `_id` is also accepted so documents created after the migration — which have no legacy id — can be deleted too. | `4211` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": 4211
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":4211}'
```

### `POST /api/v1/backend/members/statistics`

Membership KPIs and distributions for a hierarchy scope

Supplied `params` are intersected with the caller’s own scope and cannot widen it. `profile_completeness`, which the legacy SQL aggregated, is absent from the live table and is read as 0 when missing.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `params` | object[] | no | Omit to report on the caller’s own hierarchy scope. Supplied filters are intersected with that scope — they cannot widen it. | — |

<details><summary>Full request — every accepted field</summary>

```json
{
  "params": []
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Statistics for the resolved scope. |
| `403` | The token carries no hierarchy claim. |
| `422` | An unknown field or over-long value in `params`. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/statistics" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/membersByDesignation`

Members in the caller’s parish by parish designation

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `search_term` | string | no | SQL `LIKE` pattern, `%` wildcards included. Matched case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did. | `"%worker%"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "search_term": "%worker%"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Matching members. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/membersByDesignation" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/membersByOrdination`

Members in the caller’s parish by ordination status

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `search_term` | string | no | SQL `LIKE` pattern, `%` wildcards included. Matched case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did. | `"%worker%"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "search_term": "%worker%"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Matching members. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/membersByOrdination" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/membersByVipStatus`

Members by VIP status across the caller’s whole hierarchy scope

Wider than the parish-only twin below, which is what the legacy handler intended — it read the caller’s parish and then never applied it, returning every parish in the system.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `search_term` | string | no | SQL `LIKE` pattern, `%` wildcards included. Matched case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did. | `"%worker%"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "search_term": "%worker%"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Matching members. |
| `403` | The token carries no hierarchy claim. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/membersByVipStatus" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/membersByVipStatusInParish`

Members by VIP status within the caller’s parish

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `search_term` | string | no | SQL `LIKE` pattern, `%` wildcards included. Matched case-insensitively, as MySQL’s `utf8mb4_unicode_ci` collation did. | `"%worker%"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "search_term": "%worker%"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Matching members. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/membersByVipStatusInParish" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/whatsapp-barcode/upload`

Upload the parish’s WhatsApp barcode

Stores one barcode per parish, keyed on the caller’s parish claim. Bytes are written to local storage rather than S3 in this port — see LEGACY_BUGS.md.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Barcode stored. |
| `403` | The token carries no parish claim. |
| `404` | No member exists for that parish code. |
| `422` | Missing file, wrong type, or over the size limit. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/whatsapp-barcode/upload" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/{member}/restore`

Restore a soft-deleted member

The `{member}` path segment is authoritative. Legacy read the id from the request body and ignored the segment entirely.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `member` | string | **yes** | Legacy `members.id`, or a 24-character MongoDB `_id` for post-migration rows. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/{member}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/{member}/force-delete`

Permanently delete a member

The legacy route named a `forceDelete` method that was never written, so this endpoint has always failed. Implemented to match the sibling force-delete handlers.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `member` | string | **yes** | Legacy `members.id`, or a 24-character MongoDB `_id` for post-migration rows. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/{member}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/assignHFCenter`

Assign a member to a house-fellowship centre

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | string | **yes** | — | `"633a23f95e4119908a78fa9d"` |
| `center_code` | string | **yes** | — | `"211414001"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "633a23f95e4119908a78fa9d",
  "center_code": "211414001"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Member assigned. |
| `404` | No such member within the caller’s scope. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/assignHFCenter" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"633a23f95e4119908a78fa9d","center_code":"211414001"}'
```

### `POST /api/v1/backend/sms/send`

Send an SMS

Sends the caller’s message to the caller’s number, with gateway credentials read from configuration. The legacy helper could do none of those three things — see LEGACY_BUGS.md.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `phone` | string | **yes** | — | `"07012345678"` |
| `message` | string | **yes** | — | `"Midweek service moves to 7pm from Thursday."` |
| `country_code` | string | **yes** | Dialling code, no leading +. | `"44"` |
| `id` | object | no | Legacy quirk preserved: its presence flips the success status from 201 to 200 (`MembersController@sendSMS`, ~line 2024). | `1` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "phone": "07012345678",
  "message": "Midweek service moves to 7pm from Thursday.",
  "country_code": "44"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "phone": "07012345678",
  "message": "Midweek service moves to 7pm from Thursday.",
  "country_code": "44",
  "id": 1
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Sent. Returned when the request carries an `id`. |
| `201` | Sent. |
| `400` | SMS is not configured, or the gateway refused. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/sms/send" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"phone":"07012345678","message":"Midweek service moves to 7pm from Thursday.","country_code":"44"}'
```

---

## Members Attendance Logs

### `GET /api/v1/backend/members-attendance-logs`

List attendance logs for the caller’s parish

Ordered by `clock_in` descending, each row carrying its eager-loaded `schedule` (null when the schedule has been soft-deleted). An optional `schedule_id` narrows the list; a value that does not match `^ATT-SCH-[A-Z0-9]{20}$` is **ignored rather than rejected**, which is the legacy behaviour and is safe because the parish filter still applies.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `schedule_id` | string | no | Optional filter. A value that does not match `^ATT-SCH-[A-Z0-9]{20}$` is ignored rather than rejected, matching legacy. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Attendance logs for the caller’s parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-logs" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members-attendance-logs/member`

One member’s attendance logs

Defaults to the caller’s own `user_code` claim; `?user_code=` overrides it for an admin lookup, still within the caller’s parish. `user_code` is compared **case-sensitively and is deliberately not upper-cased** — the legacy `strtoupper` was harmless only under MySQL’s case-insensitive collation and would return nothing on MongoDB.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `user_code` | string | no | Whose logs to read. Defaults to the caller’s own `user_code` claim. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | That member’s logs, newest first. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-logs/member" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members-attendance-logs/absentees`

Members of the parish with no attendance log for a schedule

The present-list is gathered by an aggregation that filters `deleted_at` explicitly, so a soft-deleted log does not mark a member present. Note two preserved legacy quirks: the result includes members whose `status` is `INACTIVE` despite the legacy comment claiming “active members”, and members with a NULL `user_code` are omitted whenever anyone at all attended, because `NULL NOT IN (…)` is NULL in SQL.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `schedule_id` | string | **yes** | Schedule to report absentees for. Must already be upper-case. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Absentees, ordered by last then first name. |
| `403` | The token carries no parish claim. |
| `404` | No such schedule in the caller’s parish. |
| `422` | `schedule_id` is missing or malformed. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-logs/absentees" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members-attendance-logs/report`

Attendance report at parish, service or member level

`?level=parish` (the default) lists every schedule with its attendee and score counts; `?level=service&schedule_id=…` returns one service with its full member list; `?level=member&user_code=…` returns one member across all services. An unrecognised `level` falls through to `parish` without erroring, as legacy did. Every member lookup is scoped to the caller’s parish — the legacy `service` and `member` levels were not, and leaked names, phones and emails from other parishes.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `level` | `parish` \| `service` \| `member` | no | One of `parish`, `service`, `member`. Anything else — including omission — is treated as `parish`, exactly as legacy did. |
| `schedule_id` | string | no | Required when `level=service`. Must already be upper-case. |
| `user_code` | string | no | Required when `level=member`. Compared case-sensitively; see §2.1. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The report for the requested level. |
| `403` | The token carries no parish claim. |
| `404` | `level=service` and no such schedule in the parish. |
| `422` | Missing `schedule_id` or `user_code` for the level. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-logs/report" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members-attendance-logs/score-report`

Service-satisfaction scores aggregated at a hierarchy level

The legacy handler consulted **no** token at all, so `type=national` aggregated every attendance log in the database and `type=parish&type_code=<any parish>` returned another parish’s attendance volumes and satisfaction scores. The caller’s own narrowest scope is now always intersected in with `$and`, so `national` means “everything you can see” and a `type_code` outside your branch returns an empty report. A token with no hierarchy claim at all gets 403 rather than the whole estate.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | `national` \| `parish` \| `area` \| `zone` \| `province` \| `region` \| `subContinent` \| `continent` | **yes** | Hierarchy level to aggregate at. |
| `type_code` | string | no | Code at the requested level. Required unless `type=national`. Upper-cased, trimmed and HTML-stripped in the service, matching legacy (`:660`). |
| `schedule_id` | string | no | Optional drill-down to a single schedule. Must already be upper-case. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Per-schedule scores plus a weighted summary. |
| `403` | The token carries no hierarchy claim at all. |
| `422` | Unknown `type`, or `type_code` missing. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-logs/score-report" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members-attendance-logs/verify-member`

Check a member and schedule before clock-in

Returns the member’s identity, the schedule with its QR URL, and whether they have already clocked in or out. `parish_code` in the body **must equal the caller’s own parish claim**: the legacy handler trusted the body, which let any authenticated caller enumerate `rccg_code`s in any parish and read names, phones, emails, passport paths, titles and genders.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `parish_code` | string | **yes** | Must equal the caller’s own parish claim; anything else is 403. | `"211549"` |
| `schedule_id` | string | **yes** | — | `"ATT-SCH-62F55000B9E444298CC7"` |
| `rccg_code` | string | **yes** | — | `"RCCG2622432007"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "parish_code": "211549",
  "schedule_id": "ATT-SCH-62F55000B9E444298CC7",
  "rccg_code": "RCCG2622432007"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Member, schedule and clock state. |
| `403` | `parish_code` is not the caller’s own parish. |
| `404` | No such member, or no such schedule, in that parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-logs/verify-member" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"parish_code":"211549","schedule_id":"ATT-SCH-62F55000B9E444298CC7","rccg_code":"RCCG2622432007"}'
```

### `POST /api/v1/backend/members-attendance-logs/clock-in`

Clock the caller in for a service

The caller’s member record is resolved by `user_id` **within their own parish** — the legacy lookup had that condition commented out, so a token could clock in another parish’s member and write a row mixing two tenancies. A physical schedule is geofenced to 200 m of its venue; a schedule with no venue coordinates fails closed with `OUT OF RANGE`, which is preserved deliberately. A second clock-in is 409, including over a soft-deleted log, which used to surface as a 500 from the UNIQUE index.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `schedule_id` | string | **yes** | Schedule to clock into. Upper-cased and trimmed before validation, matching the legacy `prepareForValidation`, so a lower-case id is accepted on this route. | `"ATT-SCH-62F55000B9E444298CC7"` |
| `timezone` | string | **yes** | IANA zone the clock-in is reported from. | `"Europe/London"` |
| `user_longitude` | number | no | Required in practice for a `physical` schedule — the geofence rejects the request without it. Stored as the string form of the parsed float, matching the `varchar(255)` column. | `-0.11805` |
| `user_latitude` | number | no | As `user_longitude`. | `51.5099` |
| `ip_address` | string | no | Optional. Omitted or blank falls back to the socket address, as the controller intended. | `"192.168.1.10"` |
| `device_id` | string | no | — | `"device-uuid-abc123"` |
| `device_fingerprint` | string | no | — | `"fp_abc123xyz"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "schedule_id": "ATT-SCH-62F55000B9E444298CC7",
  "timezone": "Europe/London"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "schedule_id": "ATT-SCH-62F55000B9E444298CC7",
  "timezone": "Europe/London",
  "user_longitude": -0.11805,
  "user_latitude": 51.5099,
  "ip_address": "192.168.1.10",
  "device_id": "device-uuid-abc123",
  "device_fingerprint": "fp_abc123xyz"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `201` | Clock-in recorded; returns `attendance_log_id`. |
| `403` | The token carries no parish claim. |
| `404` | No member for this token in this parish, or no such schedule. |
| `409` | Already clocked in for this service. |
| `422` | Service closed, out of geofence, or validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-logs/clock-in" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"schedule_id":"ATT-SCH-62F55000B9E444298CC7","timezone":"Europe/London"}'
```

### `POST /api/v1/backend/members-attendance-logs/clock-out`

Clock the caller out of a service

The log is matched on the caller’s **member `user_code`** plus their parish. Legacy compared the log’s `user_code` against the token’s `id` claim, which is what `created_by` is written from — a different claim — so this route 404’d for every caller whose token `id` did not happen to equal their `user_code`. `service_feedback` and `score` are validated here although the legacy authenticated route validated neither: an out-of-range `score` corrupts the weighted average that `score-report` serves to everyone.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `attendance_log_id` | string | **yes** | Log to close. Must already be upper-case: the legacy route validated the regex against the raw body value and never normalised it. | `"ATT-LOG-0FE162F47297448FAEC1"` |
| `service_feedback` | string | no | — | `"Uplifting service, well attended."` |
| `score` | number | no | Service satisfaction, 0 (very poor) to 5 (excellent). | `5` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "attendance_log_id": "ATT-LOG-0FE162F47297448FAEC1"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "attendance_log_id": "ATT-LOG-0FE162F47297448FAEC1",
  "service_feedback": "Uplifting service, well attended.",
  "score": 5
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Clock-out recorded. |
| `403` | The token carries no parish claim. |
| `404` | No member for this token, or no such log in the parish. |
| `409` | Already clocked out for this service. |
| `422` | Service already ended, or validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-logs/clock-out" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"attendance_log_id":"ATT-LOG-0FE162F47297448FAEC1"}'
```

### `POST /api/v1/backend/members-attendance-logs/public/clock-in`

Clock a member in without a token (QR self-service)  
**🔓 Public — no token required.**

Unauthenticated by legacy parity: identity is proven by `rccg_code` + `parish_code` in the body. **Both** are conditions on a single member query, so one parish cannot clock in another parish’s member — the legacy handler had the `parish_code` condition commented out and wrote the body’s parish onto the row regardless. An unknown `rccg_code` and a member belonging to a different parish are answered identically, from the same query, so the endpoint cannot be used to discover who is a member where. Rate-limited to 20 requests per minute per IP, tighter than the global 60.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `parish_code` | string | **yes** | Parish the caller claims. Must be the member’s own parish. | `"211549"` |
| `schedule_id` | string | **yes** | Schedule to clock into. Must already be upper-case — the legacy public route validated the regex against the raw body value, so its later `strtoupper` never had an effect. | `"ATT-SCH-62F55000B9E444298CC7"` |
| `rccg_code` | string | **yes** | Member’s RCCG code. Checked together with `parish_code`, never on its own. | `"RCCG2622432007"` |
| `timezone` | string | **yes** | — | `"Europe/London"` |
| `user_longitude` | number | no | — | `-0.11805` |
| `user_latitude` | number | no | — | `51.5099` |
| `ip_address` | string | no | — | `"192.168.1.10"` |
| `device_id` | string | no | — | `"device-uuid-abc123"` |
| `device_fingerprint` | string | no | — | `"fp_abc123xyz"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "parish_code": "211549",
  "schedule_id": "ATT-SCH-62F55000B9E444298CC7",
  "rccg_code": "RCCG2622432007",
  "timezone": "Europe/London"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "parish_code": "211549",
  "schedule_id": "ATT-SCH-62F55000B9E444298CC7",
  "rccg_code": "RCCG2622432007",
  "timezone": "Europe/London",
  "user_longitude": -0.11805,
  "user_latitude": 51.5099,
  "ip_address": "192.168.1.10",
  "device_id": "device-uuid-abc123",
  "device_fingerprint": "fp_abc123xyz"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `201` | Clock-in recorded; returns `attendance_log_id`. |
| `404` | No member with that `rccg_code` in that parish, or no such schedule. |
| `409` | Already clocked in for this service. |
| `422` | Service closed, out of geofence, or validation failed. |
| `429` | Rate limit exceeded (20 per minute per IP). |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-logs/public/clock-in" \
  -H 'Content-Type: application/json' \
  -d '{"parish_code":"211549","schedule_id":"ATT-SCH-62F55000B9E444298CC7","rccg_code":"RCCG2622432007","timezone":"Europe/London"}'
```

### `POST /api/v1/backend/members-attendance-logs/public/clock-out`

Clock a member out without a token (QR self-service)  
**🔓 Public — no token required.**

The sibling that already required both `rccg_code` and `parish_code` on the member lookup. The log is then matched on that member’s `user_code` **and** the stated parish, so a log id alone is not enough to close someone else’s attendance. Same 20-per-minute-per-IP limit as `public/clock-in`.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `attendance_log_id` | string | **yes** | Log to close. Must already be upper-case, as above. | `"ATT-LOG-0FE162F47297448FAEC1"` |
| `rccg_code` | string | **yes** | — | `"RCCG2622432007"` |
| `parish_code` | string | **yes** | — | `"211549"` |
| `service_feedback` | string | no | — | `"Uplifting service, well attended."` |
| `score` | number | no | — | `5` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "attendance_log_id": "ATT-LOG-0FE162F47297448FAEC1",
  "rccg_code": "RCCG2622432007",
  "parish_code": "211549"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "attendance_log_id": "ATT-LOG-0FE162F47297448FAEC1",
  "rccg_code": "RCCG2622432007",
  "parish_code": "211549",
  "service_feedback": "Uplifting service, well attended.",
  "score": 5
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Clock-out recorded. |
| `404` | No member with that `rccg_code` in that parish, or no such log for them. |
| `409` | Already clocked out for this service. |
| `422` | Service already ended, or validation failed. |
| `429` | Rate limit exceeded (20 per minute per IP). |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-logs/public/clock-out" \
  -H 'Content-Type: application/json' \
  -d '{"attendance_log_id":"ATT-LOG-0FE162F47297448FAEC1","rccg_code":"RCCG2622432007","parish_code":"211549"}'
```

---

## Members Attendance Schedules

### `GET /api/v1/backend/members-attendance-schedules`

List every attendance schedule in the caller’s parish

Ordered by `start_time` descending. Each row carries `qr_code_url` plus, when the schedule has attendance logs, seven aggregated score fields. Those seven keys are absent — not null — for a schedule with no logs, matching the legacy `if (!empty($stats))`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Schedules for the caller’s parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-schedules" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members-attendance-schedules/by-hierarchy`

List attendance schedules at a hierarchy level, within the caller’s own scope

The legacy handler applied no tenancy filter at all: `?type=national` returned every schedule in every parish. The requested filter is now always intersected with the caller’s own narrowest hierarchy scope, so `national` means “everything within your own scope” and a `type_code` outside the caller’s branch returns an empty list rather than another tenant’s rows.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | `national` \| `parish` \| `area` \| `zone` \| `province` \| `region` \| `subContinent` \| `continent` | **yes** | Hierarchy level to read. `national` needs no `type_code`; every other level requires one. |
| `type_code` | string | no | Code at the requested level. Upper-cased, trimmed and HTML-stripped before use, matching legacy (`:54`). Required unless `type=national`. The result is always intersected with the caller’s own scope, so a code outside their branch of the hierarchy returns nothing. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Schedules at the requested level, within scope. |
| `403` | The token carries no hierarchy claim at all. |
| `422` | Unknown `type`, or `type_code` missing. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-schedules/by-hierarchy" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members-attendance-schedules/{schedule_id}`

One attendance schedule, with its QR URL and score summary

Resolved on the business key `schedule_id`, not the row id, and scoped to the caller’s parish. The path segment is deliberately not regex-validated: legacy validated the format only on `POST …/delete`, so a malformed id answers 404 here rather than 422.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `schedule_id` | string | **yes** | Business key `members_attendance_schedules.schedule_id` — not the row id. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | The schedule. |
| `403` | The token carries no parish claim. |
| `404` | No such schedule in the caller’s parish. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-schedules/{schedule_id}" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members-attendance-schedules/save`

Create or update an attendance schedule

Omit `schedule_id` to create (201); supply it to update (200). `schedule_id`, `slug`, `qr_code` and `attendance_url` are issued once, on creation, and are deliberately not reissued on update — a QR code that has already been printed has to keep resolving. All three generated values are allocated by bounded retry, so a saturated key space is a 409 rather than a request that never returns.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `schedule_id` | string | no | Omit to create. Supplying it updates that schedule, which must belong to the caller’s parish. Upper-cased and trimmed before validation, matching legacy. | `"ATT-SCH-62F55000B9E444298CC7"` |
| `title` | string | **yes** | — | `"Sunday Service"` |
| `service_day` | string | **yes** | Free text in the live table, not an enum — `Sunday`, `Mid-Week`, anything. | `"Sunday"` |
| `service_type` | `physical` \| `online` | **yes** | Drives whether venue coordinates are required. | `"physical"` |
| `start_time` | string | **yes** | Wall-clock start, `Y-m-d H:i:s`. Read together with `timezone`. | `"2026-05-22 09:00:00"` |
| `end_time` | string | **yes** | Wall-clock end, `Y-m-d H:i:s`. Must be after `start_time`. | `"2026-05-22 11:00:00"` |
| `timezone` | string | **yes** | IANA zone the two wall-clock times are expressed in. | `"Europe/London"` |
| `year` | number | **yes** | — | `2026` |
| `month` | string | **yes** | Three-letter English abbreviation, as stored (`May`). Not validated against a list — the live column is a free `varchar(50)`. | `"May"` |
| `venue_longitude` | number | no | Required when `service_type=physical`. Stored as a string because the live column is `varchar(255)`, and reads return it verbatim. | `-0.118092` |
| `venue_latitude` | number | no | Required when `service_type=physical`. Stored as a string, as above. | `51.509865` |
| `frontend_url` | string | **yes** | Base URL of the attendance front end. `attendance_url` is derived from it as `{frontend_url}/rpms/attendance/{slug}`, and that URL is what the QR code encodes. | `"https://europe.rccgportal.org"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "title": "Sunday Service",
  "service_day": "Sunday",
  "service_type": "physical",
  "start_time": "2026-05-22 09:00:00",
  "end_time": "2026-05-22 11:00:00",
  "timezone": "Europe/London",
  "year": 2026,
  "month": "May",
  "frontend_url": "https://europe.rccgportal.org"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "schedule_id": "ATT-SCH-62F55000B9E444298CC7",
  "title": "Sunday Service",
  "service_day": "Sunday",
  "service_type": "physical",
  "start_time": "2026-05-22 09:00:00",
  "end_time": "2026-05-22 11:00:00",
  "timezone": "Europe/London",
  "year": 2026,
  "month": "May",
  "venue_longitude": -0.118092,
  "venue_latitude": 51.509865,
  "frontend_url": "https://europe.rccgportal.org"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Schedule updated. |
| `201` | Schedule created. |
| `403` | The token carries no parish claim. |
| `404` | `schedule_id` matched nothing in the caller’s parish. |
| `409` | A unique `schedule_id`, `slug` or `qr_code` could not be allocated. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-schedules/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"title":"Sunday Service","service_day":"Sunday","service_type":"physical","start_time":"2026-05-22 09:00:00","end_time":"2026-05-22 11:00:00","timezone":"Europe/London","year":2026,"month":"May","frontend_url":"https://europe.rccgportal.org"}'
```

### `POST /api/v1/backend/members-attendance-schedules/delete`

Soft-delete an attendance schedule

The id comes from the body and must already be upper-case: unlike `save`, the legacy `delete` did not run `prepareForValidation`, so it never normalised the incoming id. Preserved, because upper-casing here would accept a request legacy rejected.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `schedule_id` | string | **yes** | Schedule to soft-delete. Must already be upper-case, as in legacy. | `"ATT-SCH-62F55000B9E444298CC7"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "schedule_id": "ATT-SCH-62F55000B9E444298CC7"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Schedule soft-deleted. |
| `403` | The token carries no parish claim. |
| `404` | No such schedule in the caller’s parish. |
| `422` | `schedule_id` is missing or malformed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-schedules/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"schedule_id":"ATT-SCH-62F55000B9E444298CC7"}'
```

### `POST /api/v1/backend/members-attendance-schedules/{schedule_id}/restore`

Restore a soft-deleted attendance schedule

Legacy used `onlyTrashed()`, so a schedule that was never deleted is a 404 rather than a no-op success. Preserved.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `schedule_id` | string | **yes** | Business key `members_attendance_schedules.schedule_id` — not the row id. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Schedule restored. |
| `403` | The token carries no parish claim. |
| `404` | No soft-deleted schedule with that id in scope. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-schedules/{schedule_id}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members-attendance-schedules/{schedule_id}/force-delete`

Permanently delete an attendance schedule

Resolves the schedule with `withTrashed()`, so an already soft-deleted row can be purged. Legacy also removed the QR image from S3; no image is written by this service, so there is nothing to orphan.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `schedule_id` | string | **yes** | Business key `members_attendance_schedules.schedule_id` — not the row id. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Schedule permanently deleted. |
| `403` | The token carries no parish claim. |
| `404` | No such schedule in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members-attendance-schedules/{schedule_id}/force-delete" \
  -H 'authtoken: $TOKEN'
```

---

## RPMS Stats

### `GET /api/v1/backend/rpms/stats`

All seven dashboard metrics for the caller’s parish

Every metric is filtered by the caller’s `parish_code` claim and excludes soft-deleted rows. A token carrying no parish claim gets 422 — legacy did the same rather than falling back to system-wide totals.

**Responses**

| Status | Meaning |
|---|---|
| `200` | The seven metrics. |
| `401` | Missing or invalid token. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats`

All seven dashboard metrics across every parish

Unscoped: no `parish_code` filter is applied. Authenticated but **not** authorised — any valid token reads whole-estate totals, exactly as legacy did. See `docs/legacy-bugs/rpms-stats.md` §2.

**Responses**

| Status | Meaning |
|---|---|
| `200` | The seven metrics. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/stats/members`

Members in the caller’s parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Reported as `total_members`. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats/members" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats/members`

Members across every parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Reported as `total_members`. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats/members" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/stats/workers`

Workers in the caller’s parish

`parish_designation` containing “worker”, matched case-insensitively as the `utf8mb4_unicode_ci` column did. A case-sensitive port reports 0.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Worker count. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats/workers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats/workers`

Workers across every parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Worker count. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats/workers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/stats/children`

Children’s-church members in the caller’s parish

Counts `members` whose `church` contains “children”, case-insensitively. It is **not** a count of the `children` collection, which holds 0 rows and which legacy never consulted for this metric.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Children count. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats/children" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats/children`

Children’s-church members across every parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Children count. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats/children" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/stats/departments`

Department catalogue entries for the caller’s parish

Rows in `department_lists` — the catalogue, not the member↔department join.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Department count. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats/departments" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats/departments`

Department catalogue entries across every parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Department count. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats/departments" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/stats/hf-centers`

House-fellowship centres in the caller’s parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Reported as `hf_centers`. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats/hf-centers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats/hf-centers`

House-fellowship centres across every parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Reported as `hf_centers`. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats/hf-centers" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/stats/hf-attendance`

House-fellowship attendance records for the caller’s parish

A count of attendance **reports filed**, not of people who attended. `attendances.total` is a `varchar` headcount and is deliberately never summed — legacy counted rows.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Reported as `hf_attendance`. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats/hf-attendance" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats/hf-attendance`

House-fellowship attendance records across every parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Reported as `hf_attendance`. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats/hf-attendance" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/stats/professionals`

Employment records carrying a profession, for the caller’s parish

Excludes a null, missing or blank `profession`, as legacy did.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Professional count. |
| `422` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/stats/professionals" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/rpms/system-stats/professionals`

Employment records carrying a profession, across every parish

**Responses**

| Status | Meaning |
|---|---|
| `200` | Professional count. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/rpms/system-stats/professionals" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/members/bulk-upload`

Create or update many members from a JSON array

Takes a JSON `members` array, not a spreadsheet — the legacy handler read `$request->input('members')` and never touched PhpSpreadsheet. The spreadsheet import is `members/bulkaddTempLiterateMember`, in the members module.

Rows are processed in order and independently: one bad row fails on its own and the response reports per-row outcomes. **207 when any row failed, 201 otherwise** — clients branch on this.

A row whose `parish_code` is not the caller’s own is rejected. Legacy only *defaulted* the parish, so such a row created a member in another parish; see `docs/legacy-bugs/rpms-stats.md` §3.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `members` | object[] | **yes** | Member rows. Keys are `members` column names; unknown keys are ignored. `first_name`, `last_name` and a valid `email` are required per row. The hierarchy columns default to the caller’s own token claims when omitted. | `[{"first_name":"Emeka","last_name":"Adeyemi","email":"emeka.adeyemi@example.test","gender":"Male","phone":"07012345678","church":"Adult","parish_designation":"Worker"}]` |
| `mode` | `create` \| `upsert` | no | Rejects rows that match an existing member when set to `create`. | `"upsert"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "members": [
    {
      "first_name": "Emeka",
      "last_name": "Adeyemi",
      "email": "emeka.adeyemi@example.test",
      "gender": "Male",
      "phone": "07012345678",
      "church": "Adult",
      "parish_designation": "Worker"
    }
  ]
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "members": [
    {
      "first_name": "Emeka",
      "last_name": "Adeyemi",
      "email": "emeka.adeyemi@example.test",
      "gender": "Male",
      "phone": "07012345678",
      "church": "Adult",
      "parish_designation": "Worker"
    }
  ],
  "mode": "upsert"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `201` | Every row succeeded. |
| `207` | At least one row failed. Inspect `results[].error`. |
| `401` | Missing or invalid token. |
| `422` | The body is not a non-empty `members` array, or the token carries no parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/members/bulk-upload" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"members":[{"first_name":"Emeka","last_name":"Adeyemi","email":"emeka.adeyemi@example.test","gender":"Male","phone":"07012345678","church":"Adult","parish_designation":"Worker"}]}'
```

---

## Sermons

### `GET /api/v1/backend/sermons`

List every sermon in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned a `SermonCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `SermonsController.php:24-25` is commented out and a bare `get()` took its place.

Soft-deleted sermons are excluded, matching the Eloquent global scope. That is load-bearing on this collection specifically: **29 of the 251 live rows are trashed**, so a missing scope would inflate the estate’s listing by 13%.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Sermons for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/sermons" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/sermons/save`

Create or update a sermon

The message says "Sermon **Created** Successfully" on both branches, the update included, while the status is `$request->get("id") ? 200 : 201`. So a truthy `id` answers **200** and still claims a creation; absent, `null`, `0`, `"0"` or `false` answers **201**. Clients branch on the text, so the misleading wording is a contract and is reproduced.

All seventeen columns are written from the request, so an omitted optional is stored as `null` — and on this table that means a save omitting `sermon_audio_path` **erases a working media reference**. Preserved, because clients have relied on the resulting row shape since 2022; it is the sharpest edge in this module.

Ten of those columns are absent from `SermonRequest` and therefore unvalidated: `sermon_type`, `sermon_text`, the four media fields, `s1`–`s3` and `id`.

`parish_code` must be the caller’s own. Legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s sermon and *move* it in the process. A truthy `id` naming a **trashed** sermon is a 404 — legacy created a duplicate row at the same id instead, because `firstOrNew` carried the soft-delete scope.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `title` | string | **yes** | Sermon title. The column is `title`, **not** `sermon_title` — verified against the dump’s `CREATE TABLE`, and `sermon_title` exists in 0 of the 251 documents. 181 distinct live values in **12 case-variant groups** (`LOVE OF GOD`/`Love of God`, `Love`/`LOVE`/`love`). Never used as a query predicate by this controller, only assigned and returned. | `"The Sermon on the Mount"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. The legacy `form` endpoint took a parish straight from the URL and consulted no token at all, which made it a reader of any parish’s sermons; `post` took it from the body with no tenancy check, so any authenticated caller could file a sermon into another parish. Both are closed. | `"211716"` |
| `preacher` | string | **yes** | Who preached. 174 distinct live values, free text (`PST TOKZ`, `Hughes Bandy`). | `"Pastor Dele Ologun"` |
| `text` | string | **yes** | Scripture reference. `NOT NULL` and only `varchar(191)` despite the name — the sermon body lives in `sermon_text`. 180 distinct live values in **4 case-variant groups** (`JOHN 3:16`/`John 3:16`/`john 3:16`). | `"John 3:16"` |
| `sermon_date` | string | **yes** | Stored as a `varchar`, not a date. `required\|date_format:Y-m-d`, reproduced including the calendar-validity check `DateTime::createFromFormat` performed — all 99 distinct live values already match. | `"2024-03-03"` |
| `month` | string | **yes** | Free text. The live column holds both `Mar` and `March` **in the same parish and year** (`211778`, 2025), plus `Apr`/`April` and `Dec`/`December` elsewhere. Stored verbatim. | `"Mar"` |
| `year` | string | **yes** | Four-digit year as a string — the column is `varchar`, not `int`. | `"2024"` |
| `service_type` | string | **yes** | Which service. Eight live values, and they are **not all days of the week**: `Sunday` … `Thursday` alongside `Text-Audio`, which is plainly a `sermon_type` value written into the wrong column by a client. The legacy rule is a bare `required`, so no enum is imposed and that row stays storable. | `"Sunday"` |
| `sermon_type` | object | no | **Not validated at all** — `SermonRequest` declares no rule for it, yet `post()` assigns it. Three live values: `Text`, `Text-Video`, `Text-Audio`. | `"Text"` |
| `sermon_text` | object | no | The sermon body, `longtext`. **Not validated** yet assigned. Holds HTML in all 251 live rows (`<p>GOD IS GOOD</p>`) and carries **5 case-variant groups** of its own. Stored verbatim, with no sanitisation — the legacy app applied none, and stripping tags now would silently rewrite every existing sermon on its next save. | `"<p>God is good!</p>"` |
| `sermon_audio` | object | no | Audio URL. **Not validated** yet assigned, and it shows: 20 of 251 rows have a value, 18 of them an `https://e-remittance-med…` URL and the others the bare strings `1` and `0`. No URL rule is imposed, because one would reject those rows. | `"https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_sermons/audio.mp3"` |
| `sermon_video` | object | no | Video URL. **Not validated** yet assigned. 16 of 251 rows have a value; four carry a leading zero, one of them the bare string `0`. | `"https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_sermons/video.mp4"` |
| `sermon_audio_path` | object | no | Storage key for the audio. **Not validated** yet assigned. 18 of 251 rows have a value. | `"rpms_sermons/Text-Audio/1710000000.mp3"` |
| `sermon_video_path` | object | no | Storage key for the video. **Not validated** yet assigned. 12 of 251 rows have a value. | `"rpms_sermons/Text-Video/1710000000.mp4"` |
| `s1` | object | no | Undocumented legacy spare column. Legacy rule is `["nullable"]` — a rule list with no actual constraint. **Null in all 251 live rows**; this field has never been written, so no live data exercises it. | `null` |
| `s2` | object | no | Undocumented legacy spare column. Null in all 251 live rows. | `null` |
| `s3` | object | no | Undocumented legacy spare column. Null in all 251 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "title": "The Sermon on the Mount",
  "parish_code": "211716",
  "preacher": "Pastor Dele Ologun",
  "text": "John 3:16",
  "sermon_date": "2024-03-03",
  "month": "Mar",
  "year": "2024",
  "service_type": "Sunday"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "title": "The Sermon on the Mount",
  "parish_code": "211716",
  "preacher": "Pastor Dele Ologun",
  "text": "John 3:16",
  "sermon_date": "2024-03-03",
  "month": "Mar",
  "year": "2024",
  "service_type": "Sunday",
  "sermon_type": "Text",
  "sermon_text": "<p>God is good!</p>",
  "sermon_audio": "https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_sermons/audio.mp3",
  "sermon_video": "https://e-remittance-media.s3.eu-west-2.amazonaws.com/rpms_sermons/video.mp4",
  "sermon_audio_path": "rpms_sermons/Text-Audio/1710000000.mp3",
  "sermon_video_path": "rpms_sermons/Text-Video/1710000000.mp4",
  "s1": {},
  "s2": {},
  "s3": {},
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no live sermon in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/sermons/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"title":"The Sermon on the Mount","parish_code":"211716","preacher":"Pastor Dele Ologun","text":"John 3:16","sermon_date":"2024-03-03","month":"Mar","year":"2024","service_type":"Sunday"}'
```

### `POST /api/v1/backend/sermons/delete`

Soft-delete a sermon

Answers 200 with `{"message": "Sermon not found: Provide the correct parameter"}` when there is no such sermon, which is legacy behaviour — clients branch on the text, not the status. Note the wording differs from the expenditure and income twins, which say "Record …".

The id comes from the **body**; this route has no path segment in `api.php`. The lookup is scoped to the caller’s parish; legacy’s `Sermon::find($id)` was not, so an integer id alone could soft-delete any parish’s sermon.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `sermons.id`, or a 24-character MongoDB `_id` for a new row. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/sermons/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"5"}'
```

### `POST /api/v1/backend/sermons/{sermon}/restore`

Restore a soft-deleted sermon

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, which the legacy handler ignored entirely.

Unlike its siblings this path is exercised by migrated data: 29 live sermons are soft-deleted.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `sermon` | string | **yes** | Legacy `sermons.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `sermons.id`. Unvalidated, exactly as before. Wins over the `{sermon}` path segment, which the legacy handler ignored entirely. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such sermon in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/sermons/{sermon}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/sermons/{sermon}/force-delete`

Permanently delete a sermon

The legacy route named a `SermonsController@forceDelete` method that was never written, so this endpoint has always returned 500 — on the one collection in this batch that actually has rows needing purged. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `sermon` | string | **yes** | Legacy `sermons.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such sermon in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/sermons/{sermon}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/sermons/{sermon}/{parish}`

Read one sermon

**Security fix.** The legacy handler was `Sermon::where("id", $id)->where("parish_code", $parish_code)->get()` with the parish taken entirely from the URL — no `getUserDetails`, no token read, no comparison against the caller’s claim. Any caller could enumerate another parish’s sermons by typing its code into the path. The segment is retained because the path is part of the contract, but it must now name the caller’s own parish and the query is scoped by the **token**. A mismatch is a **403**, not an empty 200, per `FOUNDATION_CONTRACT.md` §3c.

Returns a `{"data": […]}` collection of zero or one sermon. A non-numeric `{sermon}` yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `sermon` | string | **yes** | Legacy `sermons.id`, or a MongoDB `_id`. |
| `parish` | string | **yes** | Must be the caller’s own `parish_code`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one sermon. |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `{parish}` is not the caller’s own. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/sermons/{sermon}/{parish}" \
  -H 'authtoken: $TOKEN'
```

---

## Service Feedback

### `GET /api/v1/backend/members-attendance-logs/service-feedback/statistics`

Service-feedback statistics with a hierarchy breakdown

Aggregates `members_attendance_logs.score` at the requested level and breaks it down one level further, ranked by average score then volume. The legacy handler read **no** token, so `type=national` aggregated every log in the database and any `type_code` returned another branch’s scores; the caller’s own narrowest scope is now always intersected in with `$and`, so `national` means “everything you can see”. Soft-deleted logs are excluded by an explicit `$match` — `aggregate()` runs outside the query middleware, so nothing else would exclude them and a deleted rating would inflate both the count and the average. `zoneName`, `areaName` and `parishName` are always `null`: those columns do not exist on `members`, which is why `type=province|zone|area` returned MySQL error 1054 — a 500 — before this port.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | `national` \| `continent` \| `subContinent` \| `region` \| `province` \| `zone` \| `area` \| `parish` | **yes** | Hierarchy level the report is scoped to. Intersected with the caller’s own scope. |
| `type_code` | string | no | Code at the requested level. Required unless `type=national`. Upper-cased, trimmed and HTML-stripped before use, and echoed back in that form. |
| `startDate` | string | no | Inclusive lower bound on the `clock_in` date, `Y-m-d`. |
| `endDate` | string | no | Inclusive upper bound on the `clock_in` date, `Y-m-d`. Must not precede `startDate`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Summary, score distribution and ranked breakdown. |
| `401` | Missing or invalid token. |
| `403` | The token carries no hierarchy claim at all. |
| `422` | Unknown `type`, missing `type_code`, or a malformed date. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-logs/service-feedback/statistics" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/members-attendance-logs/service-feedback/schedule-statistics`

Service-feedback statistics for one service, with every individual rating

Returns the schedule, attendance and rating totals, the score distribution and each rater’s comment joined to their name and `rccg_code`. Both the schedule and the logs are scoped to the caller’s parish — legacy scoped neither, so a `schedule_id` alone exposed another parish’s free-text feedback together with every rater’s identity. A schedule outside the caller’s parish is answered by the same 404 as one that does not exist. Note the two collections spell the parish differently: `members_attendance_schedules.parish` against `members_attendance_logs.parish_code`.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `schedule_id` | string | **yes** | Business key of the service, `ATT-SCH-` followed by 20 upper-case alphanumerics. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Schedule, summary, distribution and feedback entries. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such schedule in the caller’s parish. |
| `422` | `schedule_id` is missing or malformed. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/members-attendance-logs/service-feedback/schedule-statistics" \
  -H 'authtoken: $TOKEN'
```

---

## Spirituals

### `GET /api/v1/backend/spirituals`

The member named by the last spiritual record, with their records attached

The summary is not a typo. The legacy handler loops over every `user_code` in `spirituals` and **reassigns** `$members` on each pass instead of accumulating, so 327 of its 328 queries are discarded and the response describes only the last live row. Since `members.user_code` is uniquely indexed, that is a collection of **zero or one member**, each carrying a `member_spiritual` array — wrapped in `{"data": […]}` because `SpiritualCollection` is a Laravel `ResourceCollection`.

**Reproduced deliberately.** Returning every member in the parish who holds a spiritual record is the obvious repair, but it would expose hundreds of rows where this endpoint has returned at most one since 2022. The N+1 is not reproduced — the discarded queries have no observable effect. Flagged as a product decision.

**Security fix.** The member query carried no tenancy condition, so whichever parish owned the last spiritual row, every caller in the estate received that member’s full record. The lookup is now intersected with the caller’s own parish, which for most callers means an empty collection.

Not paginated — `Spiritual::query()` and `paginate(20)` are both commented out at `SpiritualsController.php:24,37`. Soft-deleted rows are excluded on both collections: 13 of the 328 spiritual rows are trashed, so "last" means the last live one.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one member. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/spirituals" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/spirituals/save`

Create or update a spiritual record

The message and the status come from **two different tests on the same field**, and they disagree: the message from `isset($id)`, the status from `$request->get("id") ? 200 : 201`. So `"0"`, `0` or `false` answers **201** while saying "Record **Updated** Successfully"; absent or `null` answers 201 and says "Saved"; anything else answers **200** and says "Updated". Clients branch on the text, so both halves are reproduced.

All eight columns are written from the request, so an omitted optional is stored as `null` — a save omitting `comment` **erases it**, and all 328 live rows carry one. Four of the eight (`comment`, `s1`–`s3`) carry a `["nullable"]` rule with no constraint, and `id` no rule at all.

**Security fix.** `user_code` came from the body with nothing checking it, on a table that has no `parish_code` at all, so any authenticated caller could file a record against any parish’s member. The named member must now be in the caller’s parish. A `user_code` naming no member is also a 403: with no resolvable member there is no parish to authorise against. A truthy `id` naming nothing in scope is a 404 — legacy inserted at the caller’s chosen primary key instead.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `qualification` | string | **yes** | The qualification earned. 22 distinct live values in **3 case-variant groups** (`bible college`/`Bible College`, `SOD`/`Sod`, `Workers in Training`/`Workers In Training`). Never a query predicate in this controller — only assigned and returned — so no matcher applies to it; anything that ever *filters* on it must compare case-insensitively. | `"School of Disciples"` |
| `start_date` | string | **yes** | Stored as a `varchar`, not a date. `required\|date_format:Y-m-d`, reproduced including the calendar-validity check `DateTime::createFromFormat` performed — all 280 distinct live values already match `^\d{4}-\d{2}-\d{2}$`, so nothing stored today is rejected. | `"2024-03-01"` |
| `finish_date` | string | **yes** | Also a `varchar` and also `required\|date_format:Y-m-d`. **No rule compares it against `start_date`**, so a qualification that finishes before it starts remains storable — and 282 distinct live values exist, all already `Y-m-d`. | `"2024-05-01"` |
| `user_code` | string | **yes** | The member this record belongs to — `members.user_code`, joined as a plain string because the legacy database has no foreign keys. **Must name a member in the caller’s own parish.** `spirituals` has no `parish_code` of its own, so this column is the only route to a tenancy decision; legacy performed none, and any caller could file a record against any parish’s member. 249 distinct live values, of which some are 24-character hex and some are UUIDs with hyphens, so the comparison is case-insensitive. | `"65e7daaea42f7669370efdcf"` |
| `comment` | object | no | Free-text remark. Legacy rule is `["nullable"]` — a rule list with no constraint. 202 distinct live values in **17 case-variant groups** (`Test`/`test`, `good`/`GOOD`/`Good`, `Empowering`/`empowering`). Never a predicate here, only assigned and returned. | `"Very good"` |
| `s1` | object | no | Undocumented legacy spare column. Legacy rule is `["nullable"]`. **Null in all 328 live rows** — this field has never been written, so no live data exercises it. | `null` |
| `s2` | object | no | Undocumented legacy spare column. Null in all 328 live rows. | `null` |
| `s3` | object | no | Undocumented legacy spare column. Null in all 328 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all** — `SpiritualRequest` does not mention it. It drives two different tests in the legacy handler, which disagree: the *message* comes from `isset($id)` and the *status* from `$id ? 200 : 201`. So `"0"`, `0` and `false` are "set" but falsy and answer **201** while still saying "Record **Updated** Successfully"; absent or `null` answers 201 and says "Saved"; anything else answers **200** and says "Updated". | `"328"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "qualification": "School of Disciples",
  "start_date": "2024-03-01",
  "finish_date": "2024-05-01",
  "user_code": "65e7daaea42f7669370efdcf"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "qualification": "School of Disciples",
  "start_date": "2024-03-01",
  "finish_date": "2024-05-01",
  "user_code": "65e7daaea42f7669370efdcf",
  "comment": "Very good",
  "s1": {},
  "s2": {},
  "s3": {},
  "id": "328"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (`id` absent, `null`, or PHP-falsy). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `user_code` names no member in the caller’s parish. |
| `404` | A truthy `id` that names no live record in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/spirituals/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"qualification":"School of Disciples","start_date":"2024-03-01","finish_date":"2024-05-01","user_code":"65e7daaea42f7669370efdcf"}'
```

### `POST /api/v1/backend/spirituals/delete`

Soft-delete a spiritual record

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such record, which is legacy behaviour — clients branch on the text, not the status. This controller says "Record …" where `sermons` and `events` name the domain object, and unlike its `churches`/`variables` batch-mates it does check before deleting, so it does not 500.

The id comes from the **body**; this route has no path segment in `api.php`. The lookup is now scoped through `members`; legacy’s `Spiritual::find($id)` was unscoped, so an integer id alone could soft-delete any parish’s record.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `spirituals.id`, or a 24-character MongoDB `_id` for a new row. | `"328"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "328"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/spirituals/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"328"}'
```

### `POST /api/v1/backend/spirituals/{spiritual}/restore`

Restore a soft-deleted spiritual record

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, scoped through `members`. The body `id` wins over the path segment, which the legacy handler ignored entirely — `$request->get("id")` never consults route parameters, so the documented route only ever worked with the id in the body.

Exercised by migrated data: 13 live spiritual records are soft-deleted.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `spiritual` | string | **yes** | Legacy `spirituals.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `spirituals.id`. Unvalidated, exactly as before. Wins over the `{spiritual}` path segment, which the legacy handler ignored entirely. | `"328"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "328"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/spirituals/{spiritual}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/spirituals/{spiritual}/force-delete`

Permanently delete a spiritual record

The legacy route named a `SpiritualsController@forceDelete` method that was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped through `members` and able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `spiritual` | string | **yes** | Legacy `spirituals.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/spirituals/{spiritual}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/spirituals/{spiritual}`

Read one spiritual record

The route names `SpiritualsController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one record.

Scoping is indirect, because `spirituals` has no `parish_code`: the record’s `user_code` must name a member in the caller’s parish. A record belonging to another parish — or orphaned from `members` entirely, which the absent foreign keys make possible — yields an empty collection rather than a 403, matching `expenditures.form`. A garbage `{spiritual}` also yields an empty collection, because MySQL coerced it to 0 and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `spiritual` | string | **yes** | Legacy `spirituals.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one record. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/spirituals/{spiritual}" \
  -H 'authtoken: $TOKEN'
```

---

## Tasks

### `GET /api/v1/backend/tasks`

List every task with its assigned directory entry

Returns the **whole table** in `{"data": …}`. There is no `paginate()` in the legacy handler at all — not even a commented-out one — so adding pagination would be a behaviour change.

Each element carries `task_users`: the `users_lists` row this task is assigned to, as an object, or `null` when `user_id` matches nothing. Eloquent snake-cases a loaded relation name, which is why the key is `task_users` and not `taskUsers`. A soft-deleted directory entry also yields `null`, because the eager load carried its soft-delete scope.

**`?description=` is NOT a substring search — read this before using it.** The legacy handler runs `->get()` first and only then calls `->where("description", "LIKE", "%…%")`, so that is `Illuminate\Support\Collection::where`, not the query builder. `Collection`’s `operatorForWhere()` has a `switch` whose `default:` case falls through into `case "="`, and `"LIKE"` matches no case — so the filter is a **loose equality against the literal string `"%foo%"`**. `?description=foo` therefore returns only tasks whose description is exactly the six characters `%foo%`, which in practice means an empty list. It is also **case-sensitive**, because the comparison happens in PHP rather than in MySQL and the `utf8mb4_unicode_ci` collation never applied. Sending the key at all with a blank value applies the filter as `"%%"`.

This is **reproduced, not repaired**, because it is an observable response contract: the endpoint returns everything unless the literal matches. Making it a real substring match would change what every existing caller receives, so it is raised as a product decision instead.

**One shape consequence.** The surviving rows keep their pre-filter positions — `Collection::where` is `array_filter`, which preserves keys — and PHP encodes an array as a JSON array only when its keys are `0..n-1`. So if the filter drops a row that is not at the tail, `data` comes back as an **object** keyed by the original position (`{"3": {…}}`) rather than an array. Reproduced.

**Not scoped, because there is nothing to scope by.** `tasks` has no `parish_code` and no hierarchy column, and the legacy handler applied no filter, so a token with no parish claim is served rather than refused.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `description` | object | no | **Not a substring search.** The legacy filter is a loose equality against the literal `"%" + value + "%"`, so `?description=foo` returns only tasks whose description is exactly the six characters `%foo%`. Sending it at all therefore empties the list in every realistic case. Reproduced rather than repaired — see the endpoint description. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Every task, each with its `task_users` relation. `data` is an array, or an object keyed by the pre-filter position when `description` dropped a non-trailing row. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/tasks" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/tasks/save`

Create or update a task

The body is `{"saved": true}` on both branches with **no message**, so a client can only tell a create from an update by the status, which is `$request->get("id") ? 200 : 201` — PHP truthiness, so `0`, `"0"`, `false` and a blank folded to null all CREATE and answer **201**.

**Every field is `nullable`** (`TaskRequest.php:28-30`), so an empty body is valid and stores a row of nulls. All three columns are written from the request, which makes a partial save destructive: saving without `user_id` un-assigns the task. Reproduced — adding the `required` rules a reader would expect would reject requests the legacy API accepts today.

A truthy `id` naming no **live** task is a **404**. Legacy ran `firstOrNew(["id" => $id])`, which matches on the primary key alone, then assigned `$task->id = $id`: an unknown id INSERTed at a caller-chosen primary key, and an id naming a *trashed* row collided with the live `PRIMARY KEY (id)` for a MySQL 1062 and a 500. A trashed task cannot be resurrected by saving over it.

**Authenticated but not authorised, and that is escalated rather than fixed.** Any valid token at any level may create, re-assign or delete any task in the estate. Legacy performed no authorisation and adding a role gate now would lock out whoever relies on it.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `description` | object | no | Free text describing the task. `varchar(191) DEFAULT NULL`, legacy rule `["nullable"]`, so a save may omit it and store null. This is the one column `index` can filter on — see the `description` query parameter, whose filter does **not** behave like a substring search. | `"Follow up with the new converts from Sunday service"` |
| `state` | object | no | Workflow state, stored as free text. **No enum is imposed and none ever was** — the legacy rule is `["nullable"]` — so any string is accepted. The collection is empty, so there is no set of live values to document or to constrain against. | `"open"` |
| `user_id` | object | no | The `users_lists` row this task is assigned to. Stored as `varchar(191)` while `users_lists.id` is `int(10) unsigned`, so the legacy join relied on MySQL coercing the string side to a double: `"7abc"` resolved to entry 7 and `"abc"` resolved to 0, which matches nothing and yields `task_users: null`. Unvalidated and **not checked for existence** — there are no foreign keys in this database and orphaned references are expected, so rejecting one would refuse a payload the legacy API stored. A 24-character hex value addresses a directory entry created by this API, which has no legacy id. | `"7"` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. The body is `{"saved": true}` on both branches, so the status code is the only signal of which happened. A truthy `id` naming no **live** task is a **404**. Legacy ran `firstOrNew(["id" => $id])`, which matches on the primary key alone and carries the soft-delete scope, then assigned `$task->id = $id` — so an unknown id INSERTed at a caller-chosen primary key (colliding with any re-import of the MySQL id space) and an id naming a *trashed* row INSERTed against the live `PRIMARY KEY (id)`, i.e. MySQL 1062 and a 500. | `"7"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "description": "Follow up with the new converts from Sunday service",
  "state": "open",
  "user_id": "7",
  "id": "7"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `404` | A truthy `id` that names no live task. |
| `422` | A non-scalar in a varchar field. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tasks/save" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/tasks/{task}/delete`

Soft-delete a task

Legacy validated nothing and checked nothing — `Task::find($request->get("id"))` followed straight by `->delete()` — so an absent or unknown `id` called a method on `null` and this endpoint returned an unconditional **500**. Fixed to a **404**, matching how every sibling null-pointer `delete`/`restore` in this conversion was resolved: a null-pointer 500 is not a response contract.

**The id comes from the BODY.** `$request->get()` never consults route parameters in Laravel, so the `{task}` segment was decorative and this route only ever worked with the id duplicated into the body. The segment is honoured as a fallback here so the documented route finally works, with the body winning.

On success the body is `{"no_content": true}` with a 200, which *is* preserved.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `task` | string | **yes** | Legacy `tasks.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `tasks.id`, or a 24-character MongoDB `_id`. Unvalidated, matching the legacy handler, which declared no rules for these routes at all. Wins over the `{task}` path segment. | `"7"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "7"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted. |
| `401` | Missing or invalid token. |
| `404` | No such task; legacy answered 500 here. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tasks/{task}/delete" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/tasks/{task}/restore`

Restore a soft-deleted task

Legacy validated nothing and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a **404** here. The body `id` wins over the path segment, which the legacy handler ignored entirely.

**No live data exercises this path**: `tasks` holds 0 documents and 0 soft-deleted rows, so there is nothing to restore and no historical response to be identical to. It rests entirely on synthetic fixtures.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `task` | string | **yes** | Legacy `tasks.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `tasks.id`, or a 24-character MongoDB `_id`. Unvalidated, matching the legacy handler, which declared no rules for these routes at all. Wins over the `{task}` path segment. | `"7"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "7"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `404` | No such task. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tasks/{task}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/tasks/{task}/force-delete`

Permanently delete a task

The legacy route names `TasksController@forceDelete`, which was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, able to reach a soft-deleted row. The path segment identifies the row here — there is no legacy body read to defer to, because there is no legacy method at all.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `task` | string | **yes** | Legacy `tasks.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `404` | No such task. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tasks/{task}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/tasks/{task}`

Read one task

The route names `TasksController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row, carrying `task_users` so this and `index` emit the identical key set. Unscoped, because the table has no scope. A garbage `{task}` yields an empty collection with a 200, because MySQL coerced it to 0 against an `int` primary key and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `task` | string | **yes** | Legacy `tasks.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one task. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/tasks/{task}" \
  -H 'authtoken: $TOKEN'
```

---

## Testimonies

### `GET /api/v1/backend/testimonies/pending-approvals`

Testimonies awaiting approval in the caller’s parish

The one endpoint here whose body is hand-built rather than an API Resource: `{success, total, data}`, with a thirteen-key projection per row and `created_at` renamed `submitted_at`. Oldest first. **No live row exercises this** — all 348 migrated testimonies are already `approved`, so the pending list is empty in production.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Pending testimonies, oldest first. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/testimonies/pending-approvals" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/testimonies`

List approved testimonies in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned a `TestimonyCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. `status` is matched case-insensitively, reproducing the `utf8mb4_unicode_ci` collation every legacy string comparison ran under.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Approved testimonies for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/testimonies" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/testimonies/{testimony}/{parish}`

Read one testimony from the caller’s parish

Returns a `{"data": […]}` collection of zero or one row, as the legacy `get()` did. Applies **no** status condition, so a pending or declined testimony is readable here — legacy behaviour, preserved. A non-numeric reference yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

**Security fix:** legacy took both the id *and* the parish from the URL and consulted no token at all, making this a reader of any parish’s testimonies. The parish segment must now be the caller’s own, and the query is built from the token claim rather than the segment.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `testimony` | string | **yes** | Legacy `testimonies.id`, or a MongoDB `_id`. |
| `parish` | string | **yes** | Must be the caller’s own parish code. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one testimony. |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or the parish segment is not the caller’s own. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/testimonies/{testimony}/{parish}" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/testimonies/approve`

Approve a testimony in the caller’s parish

Sets `status`, `approved_by` and `approved_at` and answers `{success, message, data:{id, status, approvedBy, approvedAt}}`. An already-approved testimony answers **422** with the bare body `{"message": "Testimony is already approved."}`. The guard is byte-exact, because the legacy check was a PHP `===` rather than a SQL comparison — a row holding `Approved` is re-approved, exactly as before.

**Security fix:** `findOrFail($id)` and the `exists:testimonies,id` rule were both unscoped, so any authenticated caller could publish any parish’s testimony, and could probe whether a given id existed anywhere in the estate. Both are now scoped to the caller’s parish.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `testimonies.id`, or a 24-character MongoDB `_id`. | `"41"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "41"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Approved. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | The testimony exists but is soft-deleted. |
| `422` | Already approved (`{message}` only), or the id is absent from the caller’s parish (Laravel’s `exists` envelope). |

```bash
curl -X POST "$BASE_URL/api/v1/backend/testimonies/approve" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"41"}'
```

### `POST /api/v1/backend/testimonies/decline`

Decline a testimony in the caller’s parish

Sets `status`, `declined_by`, `declined_at` and `decline_reason`, and answers `{success, message, data:{id, status, declinedBy, declinedAt, declineReason}}`. An already-declined testimony answers **422** with `{"message": "Testimony is already declined."}`. The approval columns are deliberately **not** cleared, so a row can carry both sets at once — legacy behaviour, and clients read them as a log.

**Security fix:** the same unscoped `findOrFail` and `exists` rule as `approve`.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `testimonies.id`, or a 24-character MongoDB `_id`. | `"41"` |
| `decline_reason` | string | no | Legacy `nullable\|string\|max:500`. | `"Please add more detail before we publish this."` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "41"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "41",
  "decline_reason": "Please add more detail before we publish this."
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Declined. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | The testimony exists but is soft-deleted. |
| `422` | Already declined, the id is absent from the caller’s parish, or `decline_reason` exceeds 500 characters. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/testimonies/decline" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"41"}'
```

### `POST /api/v1/backend/testimonies/save`

Create or update a testimony

**201 when the body carries no `id`, 200 when it does** — a legacy quirk preserved verbatim. PHP’s falsy test means `""` and the string `"0"` also mean "new". `status` is stamped `pending` on insert only, so an update leaves an approved or declined testimony in its current state. Every one of the twelve free columns is written from the request, so an omitted optional is stored as `null`; that makes a partial save destructive, which is also legacy behaviour.

**Security fixes:** `parish_code` came from the request body with no tenancy check, and `firstOrNew(["id" => …])` matched on the primary key alone — so any authenticated caller could file a testimony into another parish or overwrite one there. An unknown `id` also used to INSERT at that caller-chosen primary key; it is a 404 now.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. Stored verbatim — 326 of the 348 live rows are orphaned. | `"65e5a60fa42f7669370ef2b9"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took this straight from the body with no tenancy check, so any authenticated caller could file a testimony into another parish. | `"211774"` |
| `testifier` | string | **yes** | Name of the person testifying. | `"TOBI JOHN"` |
| `title` | string | **yes** | — | `"GOD'S FAVOUR"` |
| `details` | string | **yes** | Free text. The column is `longtext`, so no length rule applies. | `"GOD IS GOOD"` |
| `media_type` | string | no | Not in the legacy `FormRequest`, but assigned by the controller. Live values are `Text`, `text`, `Text-Audio`, `Text-Video` and `Text-Webcam_Recording` — genuinely mixed case, and unconstrained, so nothing here normalises it. | `"Text"` |
| `testimony_audio` | string | no | — | `"testimony-audio-1.mp3"` |
| `testimony_video` | string | no | — | `"testimony-video-1.mp4"` |
| `testimony_audio_path` | string | no | — | `"testimonies/audio/211774"` |
| `testimony_video_path` | string | no | — | `"testimonies/video/211774"` |
| `t1` | string | no | Undocumented legacy spare column. | `""` |
| `t2` | string | no | Undocumented legacy spare column. | `""` |
| `t3` | string | no | Undocumented legacy spare column. | `""` |
| `id` | string | no | Present ⇒ update an existing testimony and answer 200; absent, empty or `0` ⇒ create one and answer 201. PHP’s `!` treats the string `"0"` as falsy, so `id=0` has always meant "new" and still does. | `"41"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "65e5a60fa42f7669370ef2b9",
  "parish_code": "211774",
  "testifier": "TOBI JOHN",
  "title": "GOD'S FAVOUR",
  "details": "GOD IS GOOD"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "user_code": "65e5a60fa42f7669370ef2b9",
  "parish_code": "211774",
  "testifier": "TOBI JOHN",
  "title": "GOD'S FAVOUR",
  "details": "GOD IS GOOD",
  "media_type": "Text",
  "testimony_audio": "testimony-audio-1.mp3",
  "testimony_video": "testimony-video-1.mp4",
  "testimony_audio_path": "testimonies/audio/211774",
  "testimony_video_path": "testimonies/video/211774",
  "t1": "",
  "t2": "",
  "t3": "",
  "id": "41"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried an `id`. |
| `201` | Created; the body carried no `id`. |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | The `id` names no testimony in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/testimonies/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"65e5a60fa42f7669370ef2b9","parish_code":"211774","testifier":"TOBI JOHN","title":"GOD'S FAVOUR","details":"GOD IS GOOD"}'
```

### `POST /api/v1/backend/testimonies/delete`

Soft-delete a testimony

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such record — legacy behaviour, and clients branch on the text rather than the status. The lookup is scoped to the caller’s parish; legacy’s `Testimony::find($id)` was not, so an integer id alone could soft-delete any parish’s testimony.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `testimonies.id`, or a 24-character MongoDB `_id` for post-migration rows. | `"41"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "41"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` is missing or not numeric. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/testimonies/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"41"}'
```

### `POST /api/v1/backend/testimonies/{testimony}/restore`

Restore a soft-deleted testimony

Two legacy defects are addressed. The handler read `$request->get("id")`, which never sees a route parameter, so `{testimony}` was decorative and the id had to arrive in the body — the body still takes precedence, so existing callers are unaffected. And it called `->restore()` on a possibly-null model, making an unknown id an unconditional 500; it is a 404 here, scoped to the caller’s parish.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `testimony` | string | **yes** | Legacy `testimonies.id`. The legacy handler read `id` from the **body** and ignored this segment entirely; the body still wins, and this is now honoured as a fallback. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | no | Legacy `testimonies.id`. Wins over the `{testimony}` path segment, which the legacy handler ignored entirely. | `"41"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "41"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such testimony in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/testimonies/{testimony}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/testimonies/{testimony}/force-delete`

Permanently delete a testimony

The legacy route named a `TestimoniesController@forceDelete` method that was never written, on the controller or on its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `testimony` | string | **yes** | Legacy `testimonies.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such testimony in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/testimonies/{testimony}/force-delete" \
  -H 'authtoken: $TOKEN'
```

---

## Tithings

### `GET /api/v1/backend/tithings`

List every tithe record in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned a `TithingCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `TithingsController.php:24-26` is commented out and a bare `get()` took its place, so the whole parish history comes back as a flat array.

The five `amount_wk*` columns and `total` are returned as the **original strings** — every money column in this database is `varchar`, leading zeros such as `02875` and `000000` are real stored values, and the `*_numeric` mirrors that exist for aggregation are never exposed.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Tithe records for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/tithings" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/tithings/save`

Create or update a member’s monthly tithe record

**Status and message do not move together.** Absent or `null` `id` ⇒ create, **201**, "Tithe Record Saved Successfully". Truthy `id` ⇒ update, **200**, "Tithe Record Updated Successfully". Present but PHP-falsy `id` (`0`, `"0"`, `false`) ⇒ create, **201**, yet carrying the *Updated* message — the branch is `isset($id)` while the status is `$id ? 200 : 201`. Reproduced verbatim.

`total` is always recomputed as `intval(wk1)+…+intval(wk5)`; a `total` in the body is validated and then discarded. `intval` **truncates**, so a `210.50` week contributes 210 — that is why two live rows are 20p short of their own weeks, and it is preserved rather than corrected.

Every column is written from the request, so an omitted optional is stored as `null`: a partial save is destructive. `parish_code` must be the caller’s own — legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s record and move it in the process.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | string | **yes** | `members.user_code`. Stored verbatim and never used as a lookup key by this controller — all 179 distinct live values resolve to a member, but nothing enforces that. | `"65e7daaea42f7669370efdcf"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took this straight from the body with no tenancy check, so any authenticated caller could file a tithe into another parish — or, on the update path, move an existing record out of its own. | `"211716"` |
| `amount_wk1` | object | no | Week 1 offering. A **string**, because the column is `varchar(191)` — there is not one DECIMAL column in this database. `02875` and `000000` are real stored values, so the bytes are preserved exactly as sent. Omit the field to store null; sending `""` or `null` is a 422, which is what the legacy `numeric` rule did once `ConvertEmptyStringsToNull` had run. | `"800"` |
| `amount_wk2` | object | no | Week 2 offering, as a string. | `"900"` |
| `amount_wk3` | object | no | Week 3 offering, as a string. | `"700"` |
| `amount_wk4` | object | no | Week 4 offering, as a string. | `"700"` |
| `amount_wk5` | object | no | Week 5 offering, as a string. | `"500"` |
| `total` | object | no | **Validated and then discarded.** `TithingsController@post` comments out the assignment (`:53`) and recomputes the total from the five weeks (`:67`), so whatever is sent here only decides whether the request is accepted. | `"3600"` |
| `month` | string | **yes** | Free text, not an enum: the live table holds `Jan` and `January`, `Dec` and `December`, across 18 distinct spellings in 5 years. | `"Mar"` |
| `year` | string | **yes** | Four-digit year as a string — the column is `varchar`, not `int`. | `"2024"` |
| `pastor_approval` | object | no | Legacy rule is `["nullable"]` — a rule list with no actual constraint, so anything is accepted. **Null in all 236 live rows**: this flag has never once been set. | `null` |
| `t1` | object | no | Undocumented legacy spare column. Null in all 236 live rows. | `null` |
| `t2` | object | no | Undocumented legacy spare column. Null in all 236 live rows. | `null` |
| `t3` | object | no | Undocumented legacy spare column. Null in all 236 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all** — `TithingRequest` declares no rule for it. Three states matter and all three are reachable: absent or `null` ⇒ create, answering 201 with "Tithe Record Saved Successfully"; present and truthy ⇒ update, answering 200 with "Tithe Record Updated Successfully"; present but PHP-falsy (`0`, `"0"`, `false`) ⇒ create, yet still carrying the *update* message and a **201** — because the branch is `isset($id)` while the status is `$id ? 200 : 201`. A legacy quirk, reproduced verbatim. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "65e7daaea42f7669370efdcf",
  "parish_code": "211716",
  "month": "Mar",
  "year": "2024"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "user_code": "65e7daaea42f7669370efdcf",
  "parish_code": "211716",
  "amount_wk1": "800",
  "amount_wk2": "900",
  "amount_wk3": "700",
  "amount_wk4": "700",
  "amount_wk5": "500",
  "total": "3600",
  "month": "Mar",
  "year": "2024",
  "pastor_approval": {},
  "t1": {},
  "t2": {},
  "t3": {},
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no record in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tithings/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"65e7daaea42f7669370efdcf","parish_code":"211716","month":"Mar","year":"2024"}'
```

### `POST /api/v1/backend/tithings/{tithing}/delete`

Soft-delete a tithe record

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such record, which is legacy behaviour — clients branch on the text, not the status.

The `id` in the **body** wins; the `{tithing}` segment is a fallback. `$request->get('id')` never consults route parameters in Laravel, so the documented segment has always been decorative and the endpoint only ever worked with the id in the body. The lookup is scoped to the caller’s parish; legacy’s `Tithing::find($id)` was not, so an integer id alone could soft-delete any parish’s record.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `tithing` | string | **yes** | Legacy `tithings.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | no | Legacy `tithings.id`, or a 24-character MongoDB `_id` for post-migration rows. Wins over the `{tithing}` path segment. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | A body `id` that is present but not numeric. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tithings/{tithing}/delete" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/tithings/{tithing}/restore`

Restore a soft-deleted tithe record

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. As with `delete`, the body `id` wins over the path segment.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `tithing` | string | **yes** | Legacy `tithings.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `tithings.id`. Unvalidated, exactly as before. Wins over the `{tithing}` path segment, which the legacy handler ignored entirely. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tithings/{tithing}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/tithings/{tithing}/force-delete`

Permanently delete a tithe record

The legacy route named a `TithingsController@forceDelete` method that was never written, on the controller or its base class, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and reaching a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `tithing` | string | **yes** | Legacy `tithings.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such record in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/tithings/{tithing}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/tithings/{tithing}`

Read one tithe record from the caller’s parish

The route names `TithingsController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row. A non-numeric reference yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

The parish comes from the **token**, not from the URL. The three sibling `form` methods that do exist (`sermons`, `events`, `testimonies`) each took it from a second path segment and consulted no token at all, which made every one of them a reader of any parish’s records; that is not carried over.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `tithing` | string | **yes** | Legacy `tithings.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one tithe record. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/tithings/{tithing}" \
  -H 'authtoken: $TOKEN'
```

---

## Transfers

### `GET /api/v1/backend/transfers`

List transfer records for the caller’s parish, 20 per page

**Security fix — this endpoint had no tenancy filter of any kind.** The legacy handler was `Transfer::query()->paginate(20)`: no `getUserDetails`, no `where`, nothing. Any authenticated caller received page one of **every parish’s** transfer records, each row carrying a member’s `user_code`, the parish they moved between and their designation. Unlike `variables` or `churches`, `transfers` has a real `parish_code` column, so those rows belong to parishes and the omission was a tenancy breach rather than a design choice. The query is now scoped by the token’s `parish` claim — which necessarily changes `meta.total`, because a total counting other parishes’ rows is itself part of the leak.

`paginate(20)` is **live** here rather than commented out, so the body is the Laravel 6 envelope `{"data": […], "links": {…}, "meta": {…}}`. `links` is `first`/`last`/`prev`/`next`; `meta` is `current_page`/`from`/`last_page`/`path`/`per_page`/`to`/`total`, with `from`/`to` **null** rather than 0 on an empty page and `last_page` 1 rather than 0.

No filters exist: the legacy handler had no `if ($request->has(…))` blocks, so `page` is the only input. Soft-deleted rows are excluded, matching the Eloquent global scope — 18 of the 125 live rows are trashed.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `page` | object | no | 1-based page number, 20 rows per page. Anything that is not a positive integer resolves to page 1 without an error. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | One page of transfer records, with links and meta. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/transfers" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/transfers/save`

Create or update a transfer record

The message is `"Transfer Record Saved Successfully"` on both branches, and the status is `$request->get("id") ? 200 : 201` — so a truthy `id` answers **200** while absent, `null`, `0`, `"0"` or `false` answers **201**.

All eight columns are written from the request, so an omitted optional is stored as `null`: a partial save is **destructive**. On this table the blast radius is small — `t1`/`t2`/`t3` are the only optional columns and all three are null in every live row — but the behaviour is reproduced rather than quietly narrowed.

`date_from` and `date_to` are `required|date_format:Y-m-d`, including the calendar-validity check, and **no rule compares them**, so a record ending before it begins remains storable. Note the sibling `department-transfers/save` validates the same two columns with the much looser `date` rule *and* an `after_or_equal` comparison — the two controllers genuinely disagree.

`parish_code` must be the caller’s own, and the value **stored** is the token’s. Legacy took it from the body with no check at all, and `firstOrNew` matched the primary key alone, so one parish could overwrite another’s record and *move* it in the process. A truthy `id` naming no live record in the caller’s parish is a **404**; legacy INSERTed at that caller-chosen primary key instead. A truthy `id` naming a **trashed** record is also a 404 — legacy created a duplicate row at the same id, because `firstOrNew` carried the soft-delete scope.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `user_code` | object | **yes** | The member this transfer record belongs to. Legacy rule is a bare `required` — no existence check, and none is added: the legacy database has zero foreign keys, so orphaned references exist and must be tolerated. Never used as a query predicate by this controller, only assigned and returned. | `"65e7e542a42f7669370efe01"` |
| `parish_code` | object | **yes** | Must be the caller’s own parish claim. Legacy took it straight from the body with no tenancy check at all, so any authenticated caller could file a transfer record into another parish; and `firstOrNew(["id" => …])` matched the primary key alone, so a save could *move* another parish’s record. Both closed — a mismatch is a **403**. | `"211618"` |
| `date_from` | object | **yes** | Stored as a `varchar`, not a date. `required\|date_format:Y-m-d`, reproduced including the calendar-validity check `DateTime::createFromFormat` performed — so `2024-13-45` is refused where a bare mask match would have accepted it. | `"2024-03-14"` |
| `date_to` | object | **yes** | `required\|date_format:Y-m-d`. **Not compared against `date_from`** — `TransferRequest` has no `after_or_equal` rule, unlike `DepartmentTransferRequest`, so a record that ends before it begins is storable and that stays true here. | `"2024-03-15"` |
| `parish_designation` | object | **yes** | The member’s designation at the destination parish. Free text — the legacy rule is a bare `required`, so no enum is imposed even though `variables.category = parish_designation` holds the intended list. The live column has **nine case-variant groups** (`HOD PRAYER` / `HOD Prayer`, `hod` / `HOD`, `Pastor` / `PASTOR` / `pastor`); nothing in this controller compares it, so no case folding applies, but anything that ever *does* compare it must fold case or it will silently drop rows. | `"Sanctuary keeper"` |
| `t1` | object | no | Undocumented legacy spare column. The rule is `["nullable"]` — a rule list with no actual constraint — so anything scalar is stored verbatim. **Null in all 125 live rows**, so no live data exercises it. | `null` |
| `t2` | object | no | Undocumented legacy spare column. Null in all 125 live rows. | `null` |
| `t3` | object | no | Undocumented legacy spare column. Null in all 125 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all** — it appears in no rule of `TransferRequest`. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. **This is the parameter that made the write dangerous.** Legacy matched on the primary key alone and then assigned it, so a caller could name any parish’s row and overwrite it, or INSERT at a primary key of their choosing. A truthy `id` naming no live row **in the caller’s parish** is now a 404. Validation is deliberately left as loose as it was so the `errors` map, a response contract, is unchanged. | `"12"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "user_code": "65e7e542a42f7669370efe01",
  "parish_code": "211618",
  "date_from": "2024-03-14",
  "date_to": "2024-03-15",
  "parish_designation": "Sanctuary keeper"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "user_code": "65e7e542a42f7669370efe01",
  "parish_code": "211618",
  "date_from": "2024-03-14",
  "date_to": "2024-03-15",
  "parish_designation": "Sanctuary keeper",
  "t1": {},
  "t2": {},
  "t3": {},
  "id": "12"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no live transfer record in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/transfers/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"user_code":"65e7e542a42f7669370efe01","parish_code":"211618","date_from":"2024-03-14","date_to":"2024-03-15","parish_designation":"Sanctuary keeper"}'
```

### `POST /api/v1/backend/transfers/delete`

Soft-delete a transfer record

**Answers HTTP 200 — not 404 — when there is no such record**, with `{"message": "Transfer Record not found: Provide the correct parameter"}`. That odd status is a response contract: clients branch on the message text, so it is reproduced verbatim rather than "corrected". Success is `{"message": "Transfer Record Deleted"}`, also 200.

The id comes from the **body**; this route has no path segment in `api.php` — it is `transfers/delete`, not `transfers/{transfer}/delete`. The rule is `required|numeric`, widened only to also accept a 24-character `_id` so that rows this API created (which have no `legacy_id`) are deletable at all; the rejection message keeps Laravel’s `numeric` wording so the `errors` map is unchanged.

**Security fix:** `Transfer::find($request->get("id"))` had no tenancy condition, so an integer id alone soft-deleted any parish’s record. The lookup is now scoped — and an out-of-parish row produces the **same** 200-plus-not-found response as a row that does not exist, which closes the hole and preserves the contract at once. Distinguishing the two would make this endpoint an oracle for whether an id exists in some other parish.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `transfers.id`, or a 24-character MongoDB `_id` for a row created since. | `"12"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "12"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/transfers/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"12"}'
```

### `POST /api/v1/backend/transfers/{transfer}/restore`

Restore a soft-deleted transfer record

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an **unconditional 500**. It is a **404** here, and the lookup is scoped to the caller’s parish. The success body `{"no_content": true}` with a 200 is preserved.

The body `id` wins over the path segment, which the legacy handler ignored entirely — `$request->get("id")` never consults *route* parameters, so the documented route only ever worked with the id duplicated into the body. The segment is honoured as a fallback so it finally does something.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `transfer` | string | **yes** | Legacy `transfers.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `transfers.id`, or a MongoDB `_id`. Unvalidated, exactly as before. Wins over the `{transfer}` path segment, which the legacy handler ignored entirely. | `"12"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "12"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such transfer record in the caller’s parish; legacy answered 500. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/transfers/{transfer}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/transfers/{transfer}/force-delete`

Permanently delete a transfer record

The legacy route named a `TransfersController@forceDelete` method that was **never written**, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row — the shared soft-delete plugin leaves `delete*` unscoped precisely so force-delete can purge the row it exists for.

The id comes from the **path**, unlike `delete` and `restore`: there is no legacy handler to defer to, and the one surviving `forceDelete` implementation in the legacy app reads its route parameter.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `transfer` | string | **yes** | Legacy `transfers.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such transfer record in the caller’s parish; legacy answered 500. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/transfers/{transfer}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/transfers/{transfer}`

Read one transfer record

The legacy route named a `TransfersController@form` method that **does not exist** on the controller or on `Illuminate\Routing\Controller`, so this endpoint has returned **500** since 2022. There is no response contract to preserve, so it returns the `{"data": […]}` collection of zero or one row that every sibling scaffold in this codebase produces, scoped to the caller’s parish.

A non-numeric `{transfer}` yields an empty collection with a 200, because MySQL coerced it to 0 against an `int` primary key and matched nothing. A soft-deleted record also yields an empty collection.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `transfer` | string | **yes** | Legacy `transfers.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one transfer record. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/transfers/{transfer}" \
  -H 'authtoken: $TOKEN'
```

---

## Users Lists

### `GET /api/v1/backend/usersLists`

List every directory entry with the tasks assigned to it

Returns the **whole table** in `{"data": […]}`. There is no `paginate()` in the legacy handler and no filter of any kind — the handler takes a `Request` and ignores it entirely — so unlike `GET tasks` this `data` is always a JSON array.

Each element carries `assigned_task`: an **array** of the tasks assigned to that entry, `[]` when none and never `null`, because that is what a `hasMany` produces. Eloquent snake-cases a loaded relation name, which is why the key is `assigned_task` rather than `AssignedTask`. Soft-deleted tasks are absent from it, because the eager load carried their soft-delete scope.

The join is `tasks.user_id` → `users_lists.id`, and those columns have **different types**: `varchar(191)` against `int(10) unsigned`. MySQL made it work by converting the string side to a double using its leading-numeric-prefix rule, so `"7abc"` belonged to entry 7 and `"abc"` belonged to nothing. That coercion is reproduced rather than tightened — there are no foreign keys in this database and orphaned references are expected.

This table is **not** `users` and **not** `members`; it is a standalone name/email directory used only for task assignment.

**Not scoped, because there is nothing to scope by.** `users_lists` has no `parish_code` and no hierarchy column, so a token with no parish claim is served rather than refused.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Every directory entry, each with its `assigned_task` array. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/usersLists" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/usersLists/save`

Create or update a directory entry

The body is `{"saved": true}` on both branches with **no message**, so a client can only tell a create from an update by the status, which is `$request->get("id") ? 200 : 201` — PHP truthiness, so `0`, `"0"`, `false` and a blank folded to null all CREATE and answer **201**. Both columns are written from the request, so a partial save is destructive: saving without `name` nulls it.

**The legacy handler takes a plain `Request`, not `UsersListRequest`** (`:34`). Laravel resolves a FormRequest by parameter type, so `UsersListRequest` — which *is* imported at `:7` — was never instantiated and its `email => ["required"]` rule has never been applied to anything. It is dead code sitting in front of the one column MySQL declares `NOT NULL`.

The absence of validation is reproduced with **one exception**: `email` is required. Not as a tightening — legacy did not accept an omitted address either. It sent `NULL` into a `NOT NULL` column, MySQL raised 1048, and the request returned an unconditional **500** with no `errors` map to preserve. This answers **422** with exactly the message the dead FormRequest would have produced. There is still **no `email` format rule and no `unique` rule**, so `not-an-address` is accepted and so is the same address twice.

A truthy `id` naming no **live** entry is a **404**; legacy INSERTed at the caller’s chosen primary key, and against a trashed entry collided with the live `PRIMARY KEY (id)` for a 1062 and a 500.

**Authenticated but not authorised, and that is escalated rather than fixed.** Any valid token at any level may create, rename or delete any directory entry the whole estate shares.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `name` | object | no | Display name. `varchar(191) DEFAULT NULL` in the dump and **not validated at all** — the handler takes a plain `Request`, so even the dead `UsersListRequest`’s `["nullable"]` rule never ran. A save may omit it and store null, and because all assigned columns are written from the request, a partial save nulls it. | `"Adebayo Ogunlesi"` |
| `email` | object | **yes** | Contact address, and the one `NOT NULL` column in this table. **Required here where legacy returned a 500**: no validator ran, so an omitted address sent `NULL` into a `NOT NULL` column and MySQL raised 1048. This answers 422 with the message Laravel’s `required` rule would have produced. There is **no format rule and no `unique` rule**, matching legacy: `not-an-address` is accepted, and so is the same address twice. | `"adebayo@example.com"` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. The body is `{"saved": true}` on both branches. A truthy `id` naming no **live** entry is a **404**. Legacy ran `firstOrNew(["id" => $id])`, which matches on the primary key alone and carries the soft-delete scope, then assigned `$usersList->id = $id` — so an unknown id INSERTed at a caller-chosen primary key and an id naming a *trashed* entry collided with the live `PRIMARY KEY (id)`, i.e. MySQL 1062 and a 500. | `"7"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "email": "adebayo@example.com"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "name": "Adebayo Ogunlesi",
  "email": "adebayo@example.com",
  "id": "7"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `404` | A truthy `id` that names no live directory entry. |
| `422` | An omitted or blank `email`, where legacy answered 500 via MySQL 1048. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/usersLists/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"email":"adebayo@example.com"}'
```

### `POST /api/v1/backend/usersLists/{usersList}/delete`

Soft-delete a directory entry

Legacy validated nothing and checked nothing — `UsersList::find($request->get("id"))` followed straight by `->delete()` — so an absent or unknown `id` called a method on `null` and this endpoint returned an unconditional **500**. Fixed to a **404**.

**The id comes from the BODY.** `$request->get()` never consults route parameters in Laravel, so the `{usersList}` segment was decorative and this route only ever worked with the id duplicated into the body. The segment is honoured as a fallback here so the documented route finally works, with the body winning.

Note what a delete does *not* do: nothing updates `tasks.user_id` and there are no foreign keys, so every task assigned to this entry keeps pointing at it and its `task_users` simply becomes `null`. That is legacy behaviour and is reproduced.

On success the body is `{"no_content": true}` with a 200, which *is* preserved.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `usersList` | string | **yes** | Legacy `users_lists.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `users_lists.id`, or a 24-character MongoDB `_id`. Unvalidated, matching the legacy handler. Wins over the `{usersList}` path segment. | `"7"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "7"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted. |
| `401` | Missing or invalid token. |
| `404` | No such directory entry; legacy answered 500 here. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/usersLists/{usersList}/delete" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/usersLists/{usersList}/restore`

Restore a soft-deleted directory entry

**This route has always returned 500, and the defect is in the route rather than the controller.** `api.php:41` binds `"UsersListsController@redstore"` — a typo — while the controller’s method is `restore`. So `Illuminate\Routing\Controller::__call` threw `BadMethodCallException` on every request. It is the only misspelled handler name in `api.php`.

The sane path is registered here: `usersLists/{usersList}/restore`, which is what the route itself declares and what its name `api.usersList.restore` promises. Nothing in this API is served at a path containing `redstore`, and a spec pins that.

The method itself shares its siblings’ defects: no validation, id read from the body, `->restore()` on a possibly-null model. Those are a **404** here.

**No live data exercises this path**: `users_lists` holds 0 documents and 0 soft-deleted rows, so it rests entirely on synthetic fixtures.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `usersList` | string | **yes** | Legacy `users_lists.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `users_lists.id`, or a 24-character MongoDB `_id`. Unvalidated, matching the legacy handler. Wins over the `{usersList}` path segment. | `"7"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "7"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `404` | No such directory entry. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/usersLists/{usersList}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/usersLists/{usersList}/force-delete`

Permanently delete a directory entry

The legacy route names `UsersListsController@forceDelete`, which was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, able to reach a soft-deleted row. The path segment identifies the row — there is no legacy body read to defer to, because there is no legacy method at all.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `usersList` | string | **yes** | Legacy `users_lists.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `404` | No such directory entry. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/usersLists/{usersList}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/usersLists/{usersList}`

Read one directory entry

The route names `UsersListsController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row, carrying `assigned_task` so this and `index` emit the identical key set. Unscoped, because the table has no scope. A garbage `{usersList}` yields an empty collection with a 200, because MySQL coerced it to 0 against an `int` primary key and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `usersList` | string | **yes** | Legacy `users_lists.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one directory entry. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/usersLists/{usersList}" \
  -H 'authtoken: $TOKEN'
```

---

## Variables

### `GET /api/v1/backend/variables`

List the settings table, 20 per page

**The only endpoint in this conversion whose `paginate(20)` is live.** Every other index has the call commented out with a bare `get()` in its place; here it is real (`VariablesController.php:23-26`) and observable — 36 rows over two pages — so the response is the Laravel 6 paginated envelope `{"data": […], "links": {…}, "meta": {…}}` rather than a bare `{"data": […]}`. `links` is `first`/`last`/`prev`/`next`; `meta` is `current_page`/`from`/`last_page`/`path`/`per_page`/`to`/`total`. `from` and `to` are **null**, not 0, on an empty page, and `last_page` is 1 rather than 0 when nothing matches.

**Not scoped, because there is nothing to scope by.** `variables` has no `parish_code` and no hierarchy column — verified against the dump’s `CREATE TABLE` and against all 36 live documents — and the legacy handler applied no filter. Every row is shared reference data, so a token with no parish claim is served rather than refused.

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `page` | object | no | 1-based page number. Anything that is not a positive integer resolves to page 1 without an error, reproducing `Paginator::resolveCurrentPage()`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | One page of settings, with links and meta. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/variables" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/variables/getOrdinationStatusList`

Ordination statuses as a bare name/value array

Legacy returned the Eloquent collection **directly**, not through a resource, so the body is a bare JSON array with **no `data` envelope** — and because of `select("name","value")` each element carries only those two keys, with no `id`.

The `category` comparison is case-insensitive. The column has no case variants today (5 distinct values, all lower-case snake case), but the comparison is against a hard-coded literal under MySQL’s `ci` collation, so a row stored as `Ord_Status` matched there and would silently vanish from this list under a byte comparison.

Six live rows: `Brother`, `Sister`, `Deacon`, `Deaconess`, `Assistant Pastor`, `Pastor`.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A bare array of `{name, value}`. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/variables/getOrdinationStatusList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/variables/getTitleList`

Titles as a bare name/value array

Same shape as the other four lists: a bare JSON array of `{name, value}` with no envelope and no `id`. The largest of the five — 15 live rows, from `Miss` and `Mr` through `HRH`, `Prince`, `Chief` and `Architect`. Note that `Pastor`, `Deacon`, `Deaconess` and `Assistant Pastor` each appear here *and* under `ord_status`: `variables.name` is not unique and nothing prevents it.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A bare array of `{name, value}`. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/variables/getTitleList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/variables/getMaritalStatusList`

Marital statuses as a bare name/value array

A bare JSON array of `{name, value}` with no envelope and no `id`. Five live rows: `Single`, `Married`, `Divorced`, `Widow`, `Widower`. These are the values `members.marital_status` holds, so anything comparing that column must fold case.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A bare array of `{name, value}`. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/variables/getMaritalStatusList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/variables/getDepartmentalStatusList`

Departmental statuses as a bare name/value array

A bare JSON array of `{name, value}` with no envelope and no `id`. Five live rows, and the one list whose `value`s are **not** lower-cased: `leader`, `HOD`, `member`, `AHOD`, `EXCO`. Anything comparing `departments.status` or a member’s departmental status against these must be case-insensitive.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A bare array of `{name, value}`. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/variables/getDepartmentalStatusList" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/variables/getParishDesignationList`

Parish designations as a bare name/value array

A bare JSON array of `{name, value}` with no envelope and no `id`. Five live rows: `Worker`, `Minister`, `Deacon`, `Deaconess`, `Assistant Pastor`.

This is the list behind the single most-cited case-sensitivity measurement in this conversion: the stored `value` for `Worker` is `worker`, and `members.parish_designation` holds the capitalised spelling — so the legacy `LIKE '%worker%'` matched **2,732 rows** under the ci collation and matches **0** under a byte comparison.

**Responses**

| Status | Meaning |
|---|---|
| `200` | A bare array of `{name, value}`. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/variables/getParishDesignationList" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/variables/save`

Create or update a settings row

The body is `{"saved": true}` on both branches with **no message at all** — the only `save` in this batch that returns none — so a client can only tell a create from an update by the status, which is `$request->get("id") ? 200 : 201`.

**Every field is `nullable`**, so an empty body is valid and stores a row of nulls: a lookup row with no name, value or category, which no list endpoint will ever return. Reproduced, because adding the `required` rules a reader would expect would reject requests the legacy API accepted. All five columns are written from the request, so a partial save is destructive.

A truthy `id` naming no row is a 404; legacy inserted at the caller’s chosen primary key.

**Authenticated but not authorised, and that is escalated rather than fixed.** Any valid token at any level may edit reference data the whole estate shares — deleting the `pastor` row removes a title from every data-entry form in every parish. Legacy performed no authorisation and adding a role gate now would lock out whichever administrators rely on it.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `name` | object | no | Human-readable label. 29 distinct live values, and **not unique** — `Pastor`, `Deacon`, `Deaconess` and `Assistant Pastor` each appear under two different categories, and no constraint prevents it. Legacy rule is `["nullable"]`, so a save may omit it and store null. | `"Pastor"` |
| `value` | object | no | The stored code the rest of the application writes into `members.ord_status`, `members.title`, `members.marital_status` and `members.parish_designation`. 29 distinct live values, mostly lower-case but **not consistently**: `HOD`, `AHOD`, `EXCO`, `Minister` and `Barrister` are capitalised. That inconsistency is why anything comparing a member column against these values must do so case-insensitively — the `parish_designation` value is `worker` while the label is `Worker`, and it is the exact column behind the measured `LIKE '%worker%'` → 0-rows regression. | `"pastor"` |
| `category` | object | no | Which list this row belongs to. Five live values — `ord_status`, `title`, `marital_status`, `dept_status`, `parish_designation` — but **no enum is imposed**, because the legacy rule is `["nullable"]` and a sixth category would simply create a list no endpoint reads. The five read endpoints compare this column case-insensitively. | `"ord_status"` |
| `v1` | object | no | Undocumented legacy spare column. **Null in all 36 live rows** — never written, so no live data exercises it. | `null` |
| `v2` | object | no | Undocumented legacy spare column. Null in all 36 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. The body is `{"saved": true}` on both branches — this is the one `save` in the batch that returns no message. | `"6"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "name": "Pastor",
  "value": "pastor",
  "category": "ord_status",
  "v1": {},
  "v2": {},
  "id": "6"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; the body carried a truthy `id`. |
| `201` | Created (no `id`, or a PHP-falsy one). |
| `401` | Missing or invalid token. |
| `404` | A truthy `id` that names no live settings row. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/variables/save" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/variables/delete`

Soft-delete a settings row

Legacy validated nothing and checked nothing — `Variable::find($id)` followed straight by `->delete()` — so an absent or unknown `id` called a method on `null` and this endpoint returned an unconditional **500**. Its `spirituals` and `department_lists` batch-mates both declare `id => required|numeric` *and* check `isset()`, answering 200 with a message; this one and `churches` do neither. Fixed to a **404**, matching how every sibling `restore` in this conversion was fixed: a null-pointer 500 is not a response contract.

On success the body is `{"no_content": true}` with a 200, which *is* preserved.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `variables.id`, or a 24-character MongoDB `_id`. Unvalidated, matching the legacy handler, which declared no rules for this route at all. | `"6"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "6"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted. |
| `401` | Missing or invalid token. |
| `404` | No such settings row; legacy answered 500 here. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/variables/delete" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/variables/{variable}/restore`

Restore a soft-deleted settings row

Legacy validated nothing and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here. The body `id` wins over the path segment, which the legacy handler ignored entirely.

**No live data exercises this path**: `deleted_at` is null in all 36 rows, so there is nothing to restore and no historical response to be identical to. It rests entirely on synthetic fixtures.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `variable` | string | **yes** | Legacy `variables.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `variables.id`. Unvalidated. Wins over the `{variable}` path segment, which the legacy handler ignored entirely. | `"6"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "6"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `404` | No such settings row. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/variables/{variable}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/variables/{variable}/force-delete`

Permanently delete a settings row

The legacy route named a `VariablesController@forceDelete` method that was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, able to reach a soft-deleted row.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `variable` | string | **yes** | Legacy `variables.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `404` | No such settings row. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/variables/{variable}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/variables/{variable}`

Read one settings row

The route names `VariablesController@form`, which **was never written** — so this endpoint has always returned 500. Implemented as the `form` its sibling scaffolds perform: a `{"data": […]}` collection of zero or one row. Unscoped, because the table has no scope. A garbage `{variable}` yields an empty collection with a 200, because MySQL coerced it to 0 and matched nothing.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `variable` | string | **yes** | Legacy `variables.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one settings row. |
| `401` | Missing or invalid token. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/variables/{variable}" \
  -H 'authtoken: $TOKEN'
```

---

## Visitations

### `GET /api/v1/backend/visitations`

List every visitation report in the caller’s parish

Wrapped in `{"data": […]}` because the legacy handler returned a `VisitationCollection`, which is a Laravel `ResourceCollection`. Keys and key order follow the MySQL column order, with `id` first and `_id` appended last. Not paginated: the `paginate(20)` at `VisitationsController.php:25-26` is commented out and a bare `get()` took its place.

Soft-deleted reports are excluded, matching the Eloquent global scope — 4 of the 154 live rows are trashed.

**Responses**

| Status | Meaning |
|---|---|
| `200` | Visitation reports for the parish. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/visitations" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/visitations/save`

Create or update a visitation report

The message is "Visitation Record Saved Successfully" on both branches, with the status from `$request->get("id") ? 200 : 201` — a truthy `id` answers **200**, and absent, `null`, `0`, `"0"` or `false` answers **201**.

`rating` is **computed, never submitted** (`VisitationsController.php:64-87`), and stored as `"<n>%"`: 15 for `hfr1`, 10 for `manual`, 10 for `members_with_manual`, 15 for `timely_center` — each only when the value is **byte-exactly `Yes`**, because the legacy scorer is a PHP `===` and MySQL’s case-insensitive collation never applied to it — plus 25 for any `attendance_on_visitation` above zero and `intval(member_welfare)` for welfare. The live column holds lower-case `yes`, so those rows score nothing for that criterion; **3 of the 154 live rows would rate higher under a case-insensitive scorer**. Reproduced deliberately, so historical ratings stay reachable.

All eighteen request-sourced columns are written, so an omitted optional is stored as `null` — including `month` and `year`, which `VisitationRequest` never validates and which are two of the three columns forming the `unique_visitation` key.

**A duplicate `unique_visitation` key answers 200 with `{"error": "Duplicate entry …"}`, not a 409.** The legacy handler caught the `QueryException`, read `$exception->errorInfo[2]` and returned `response()->json(["error" => $message])` with no status argument, so Laravel defaulted to 200 — a client treating 2xx as success records a silent failure. Reproduced, because it is a response contract. The key compares only the first **11** characters of each column, is **not** scoped by parish, and is **not** aware of `deleted_at`, all three of which are preserved.

`parish_code` must be the caller’s own. Legacy took it from the body and matched on the primary key alone, so one parish could overwrite another’s report and *move* it in the process.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `center_code` | string | **yes** | The centre visited. **Not checked for existence** — the legacy database has zero foreign keys, and 12 of the 134 distinct codes in `visitations` name no row in `centers` (15 visitation rows in total). Rejecting an orphan would refuse data the API stores today. This is also the first member of the `unique_visitation` key, compared on its first **11** characters. | `"RPMSHFCENN6HF6H"` |
| `parish_code` | string | **yes** | Must be the caller’s own parish claim. Legacy took it from the body with no tenancy check and matched `firstOrNew` on the primary key alone, so a save could overwrite another parish’s visitation and *move* it. Both closed. | `"211774"` |
| `officer` | string | **yes** | Who performed the visit. 144 distinct live values in **3 case-variant groups** (`CGO` / `Cgo`, `taiwo` / `Taiwo`, `pst` / `PST`). Never used as a query predicate by this controller, only assigned and returned. | `"A/P FEMI"` |
| `visitor_status` | string | **yes** | The visiting officer’s standing. 94 distinct live values in **11 case-variant groups** (`CGO` / `Cgo` / `cgo`, `Pastor` / `PASTOR`, `PICP` / `picp`). Free text — the legacy rule is a bare `required`, so no enum is imposed. | `"Church Growth Officer"` |
| `visit_date` | string | **yes** | Stored as a `varchar`, not a date. `required\|date_format:Y-m-d`, reproduced including the calendar-validity check `DateTime::createFromFormat` performed — all 113 distinct live values already match, so nothing stored today is rejected. | `"2024-03-03"` |
| `manual` | string | **yes** | Whether the House Fellowship manual was in use. Worth **10** rating points, but only when the value is byte-exactly `Yes`: the legacy scorer is a PHP `===`, not a SQL comparison, so MySQL’s case-insensitive collation never applied to it. The live column holds `Yes`, `No` **and** lower-case `yes`, so those rows scored zero here. Reproduced — see the service. | `"Yes"` |
| `hfr1` | string | **yes** | House Fellowship Report 1 submitted. Worth **15** points on a byte-exact `Yes`. The column is `hfr1`, lower-case — the 2022 migration spells it `HFR1` and is stale; the dump is the authority. Live values: `Yes`, `No`, lower-case `yes`, and null in 24 rows. | `"Yes"` |
| `timely_center` | string | **yes** | Whether the centre met on time. Worth **15** points on a byte-exact `Yes`. Live values include lower-case `yes`, which scores zero. | `"Yes"` |
| `observation` | string | **yes** | Free-text observation. 111 distinct live values in **5 case-variant groups** (`Good` / `good` / `GOOD`, `na` / `NA`, `Satisfactory` / `SATISFACTORY`). Never a query predicate. | `"Word well thought"` |
| `comment` | string | **yes** | Free-text comment. 117 distinct live values in **12 case-variant groups** (`good` / `Good` / `GOOD`, `Great` / `GREAT`, `OK` / `ok` / `Ok`). Never a query predicate. | `"well atended"` |
| `members_with_manual` | string | **yes** | Whether members had the manual. Worth **10** points on a byte-exact `Yes`. Unlike its three siblings this column has **no** lower-case variant in the live data — only `Yes`, `No` and null — so the strict comparison happens to cost nothing here today. | `"Yes"` |
| `member_welfare` | string | **yes** | Welfare score, `required\|numeric\|min:0\|max:25`. Added to the rating **through PHP `intval()`**, which truncates rather than rounds: `24.9` contributes 24. Stored as the submitted `varchar`, so leading zeros survive — the live column holds `00` and `05`. | `"18"` |
| `attendance_on_visitation` | string | **yes** | Headcount at the visit, `required\|numeric`. Any value greater than zero is worth a flat **25** points — the size of the congregation is irrelevant beyond being non-zero. Stored as the submitted `varchar`; the live column holds `04`, so only the string round-trips. | `"54"` |
| `month` | object | no | Reporting month. **Written by `post()` but absent from `VisitationRequest`**, so it has never been validated — and it is the second member of the `unique_visitation` key, compared on its first 11 characters. The live column holds only three-letter abbreviations (`Jan`…`Dec`, 11 distinct), unlike `attendances.month` which mixes `Oct` and `October`. Stored verbatim: the MySQL index treats `Mar` and `March` as **different** keys, so folding spellings together here would make the uniqueness rule stricter than the database ever was. | `"Mar"` |
| `year` | object | no | Reporting year, a `varchar` not an `int`. **Also unvalidated** and also part of the `unique_visitation` key. Seven distinct live values, all four-digit. | `"2024"` |
| `v1` | object | no | Undocumented legacy spare column. Legacy rule is `["nullable"]` — a rule list with no actual constraint. **Null in all 154 live rows**, so no live data exercises it. | `null` |
| `v2` | object | no | Undocumented legacy spare column. Null in all 154 live rows. | `null` |
| `v3` | object | no | Undocumented legacy spare column. Null in all 154 live rows. | `null` |
| `id` | object | no | Never persisted, and **not validated at all**. A PHP-truthy value updates and answers **200**; absent, `null`, `0`, `"0"` or `false` creates and answers **201**, because the legacy status is `$request->get("id") ? 200 : 201`. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "center_code": "RPMSHFCENN6HF6H",
  "parish_code": "211774",
  "officer": "A/P FEMI",
  "visitor_status": "Church Growth Officer",
  "visit_date": "2024-03-03",
  "manual": "Yes",
  "hfr1": "Yes",
  "timely_center": "Yes",
  "observation": "Word well thought",
  "comment": "well atended",
  "members_with_manual": "Yes",
  "member_welfare": "18",
  "attendance_on_visitation": "54"
}
```

</details>

<details><summary>Full request — every accepted field</summary>

```json
{
  "center_code": "RPMSHFCENN6HF6H",
  "parish_code": "211774",
  "officer": "A/P FEMI",
  "visitor_status": "Church Growth Officer",
  "visit_date": "2024-03-03",
  "manual": "Yes",
  "hfr1": "Yes",
  "timely_center": "Yes",
  "observation": "Word well thought",
  "comment": "well atended",
  "members_with_manual": "Yes",
  "member_welfare": "18",
  "attendance_on_visitation": "54",
  "month": "Mar",
  "year": "2024",
  "v1": {},
  "v2": {},
  "v3": {},
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Updated; or a duplicate-key failure reported as `{"error": …}`. |
| `201` | Created (no `id`, or a PHP-falsy one). Also 200 with `{"error": …}` on a duplicate. |
| `401` | Missing or invalid token. |
| `403` | No parish claim, or `parish_code` is not the caller’s own. |
| `404` | A truthy `id` that names no live report in the caller’s parish. |
| `422` | Validation failed. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/visitations/save" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"center_code":"RPMSHFCENN6HF6H","parish_code":"211774","officer":"A/P FEMI","visitor_status":"Church Growth Officer","visit_date":"2024-03-03","manual":"Yes","hfr1":"Yes","timely_center":"Yes","observation":"Word well thought","comment":"well atended","members_with_manual":"Yes","member_welfare":"18","attendance_on_visitation":"54"}'
```

### `POST /api/v1/backend/visitations/delete`

Soft-delete a visitation report

Answers 200 with `{"message": "Record not found: Provide the correct parameter"}` when there is no such report, which is legacy behaviour — clients branch on the text, not the status. Note the wording is the generic "Record …" pair here, where `centers` and `events` name their domain.

The id comes from the **body**; this route has no path segment in `api.php`. The lookup is scoped to the caller’s parish; legacy’s `Visitation::find($id)` was not, so an integer id alone could soft-delete any parish’s report.

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | **yes** | Legacy `visitations.id`, or a 24-character MongoDB `_id` for a row created since. | `"5"` |

<details><summary>Minimal request — required fields only</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted, or reported as not found. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `422` | `id` missing, or present and unusable. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/visitations/delete" \
  -H 'authtoken: $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"id":"5"}'
```

### `POST /api/v1/backend/visitations/{visitation}/restore`

Restore a soft-deleted visitation report

Legacy validated nothing, scoped nothing, and called `->restore()` on a possibly-null model, so an unknown or absent id was an unconditional 500. It is a 404 here, and the lookup is scoped to the caller’s parish. The body `id` wins over the path segment, which the legacy handler ignored entirely — `$request->get("id")` never consults route parameters.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `visitation` | string | **yes** | Legacy `visitations.id`, or a MongoDB `_id`. |

**Request body**

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | object | no | Legacy `visitations.id`. Unvalidated, exactly as before. Wins over the `{visitation}` path segment, which the legacy handler ignored entirely. | `"5"` |

<details><summary>Full request — every accepted field</summary>

```json
{
  "id": "5"
}
```

</details>

**Responses**

| Status | Meaning |
|---|---|
| `200` | Restored. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such report in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/visitations/{visitation}/restore" \
  -H 'authtoken: $TOKEN'
```

### `POST /api/v1/backend/visitations/{visitation}/force-delete`

Permanently delete a visitation report

The legacy route named a `VisitationsController@forceDelete` method that was never written, so this endpoint has always returned 500. Implemented as the hard delete every sibling `forceDelete` in the legacy app performs, scoped to the caller’s parish and able to reach a soft-deleted row.

Purging matters more on this table than on most: a soft-deleted report still occupies the `unique_visitation` key, because a MySQL UNIQUE index knows nothing about `deleted_at`. Until it is purged, the same centre/month/year cannot be re-reported.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `visitation` | string | **yes** | Legacy `visitations.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | Deleted permanently. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |
| `404` | No such report in the caller’s parish. |

```bash
curl -X POST "$BASE_URL/api/v1/backend/visitations/{visitation}/force-delete" \
  -H 'authtoken: $TOKEN'
```

### `GET /api/v1/backend/visitations/{visitation}`

Read one visitation report

**The legacy `VisitationsController@form` method does not exist.** The route at `api.php:196` names it, but it is absent from the controller and from its base class, so `Illuminate\Routing\Controller::__call` threw `BadMethodCallException` and this endpoint has always returned 500. There is therefore no historical response to be byte-identical to.

Implemented on the shape every sibling `form` used — a `{"data": […]}` collection of zero or one row. Unlike the `sermons` and `events` `form` methods, which *were* written and took their parish from the URL, this route has only an id segment, so the parish can come only from the token: the cross-tenant hole is not expressible here. A non-numeric id yields an empty collection with a 200, matching MySQL’s int coercion.

**Path parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `visitation` | string | **yes** | Legacy `visitations.id`, or a MongoDB `_id`. |

**Responses**

| Status | Meaning |
|---|---|
| `200` | A collection of zero or one visitation report. |
| `401` | Missing or invalid token. |
| `403` | The token carries no parish claim. |

```bash
curl -X GET "$BASE_URL/api/v1/backend/visitations/{visitation}" \
  -H 'authtoken: $TOKEN'
```

---

## health

### `GET /api/health`

Service health, including a MongoDB ping  
**🔓 Public — no token required.**

**Responses**

| Status | Meaning |
|---|---|
| `200` | All checks passed The Health Check is successful |
| `503` | The Health Check is not successful |

```bash
curl -X GET "$BASE_URL/api/health"
```

### `GET /api/v1/health`

Service health, including a MongoDB ping  
**🔓 Public — no token required.**

**Responses**

| Status | Meaning |
|---|---|
| `200` | All checks passed The Health Check is successful |
| `503` | The Health Check is not successful |

```bash
curl -X GET "$BASE_URL/api/v1/health"
```

---

## What this document does not contain

Per-endpoint **sample response bodies**. The controllers document what each status *means* but not the body it returns, and inventing plausible-looking JSON would be worse than omitting it — a reader cannot tell a real example from a fabricated one. Use the envelope table above, and Swagger UI at `/api/docs` for live responses against your own data.

Path parameters are documented with the names the **legacy routes** used, so `{member}`, `{sermon}` and so on. Each accepts either the legacy integer id or a 24-character MongoDB `_id`, because rows created by this API have no legacy id.

