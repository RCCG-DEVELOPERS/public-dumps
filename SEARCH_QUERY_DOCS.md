# Search — choosing the right endpoint

Three ways to search the parish directory and the user directory. Each takes the same body and returns the
same envelope. The only difference is how each value is compared, and that
difference decides whether the database can use an index.

## Read this first

Nothing that exists has changed. `/v1/users/search`,
`/v1/parishdirectory/search`, `/searchAndFilter` and
`/searchAndFilterWithPastorData` behave exactly as they always have, down to the
`columnLogic` they already accept. **No frontend change is required by this
release.** The eight endpoints below are additions.

They exist because of a real outage. On 18 September the production primary
spent ten minutes at 99.7% CPU with zero read tickets free, and the API returned
about 2,970 timeouts. Two filters from that morning's slow-query log:

```js
{ parishCode: { $regex: "975462", $options: "i" } }   // 52,721 index keys → 1 row
{ province:   { $regex: "BE02",   $options: "i" } }   // 58,057 documents  → 1 row
```

`parishCode` **is** indexed, and it made no difference. An unanchored
case-insensitive regex cannot seek into an index, so MongoDB walked the whole
thing. The user query had no usable index at all and scanned the collection.

---

## Contents

- [The three comparisons](#the-three-comparisons)
- [Which one to use](#which-one-to-use)
- [Endpoints](#endpoints)
- [Request body](#request-body)
- [Worked examples](#worked-examples)
- [Behaviour you should know about](#behaviour-you-should-know-about)
- [Migrating a screen](#migrating-a-screen)
- [Still on the regex path](#still-on-the-regex-path)
- [A note on indexes](#a-note-on-indexes)

---

## The three comparisons

| Endpoint family | Sends | Uses an index |
|---|---|---|
| `/search…` | `{ $regex: "value", $options: "i" }` | **No** |
| `/query…` | `{ $eq: "value" }` | Yes |
| `/queryFlex…` | `{ $regex: "^value" }` | Yes |

`/queryFlex` needs **both** properties to stay fast. The `^` gives the index a
starting point; leaving off the `i` flag is what keeps it there. Put the `i`
back and it is exactly as slow as `/search`.

---

## Which one to use

**Use `/query` when you know the whole value.** Codes and identifiers: on
parishes `parishCode`, `provinceCode`, `regionCode`, `areaCode`, `zoneCode`,
`status`; on users `username`, `email`, `province`, `region`, `parish`,
`status`. This is most screens, most of the time, and it is the fastest.
A username or email lookup benefits most, since both are unique-indexed.

**Use `/queryFlex` for "starts with".** Name lookups and type-ahead, where the
user has typed the first few characters of `parishName`, `firstName` or
`username`.

**Stay on `/search` for "contains" and for case-insensitive matching.** Finding
"HOUSE" in the middle of a parish name genuinely needs the slow shape. Keep it
for that, and know it is expensive.

---

## Endpoints

All are `POST`, all take a bearer token, all return `201`.

**Parish directory**, under `/v1/parishdirectory`:

| Exact | Prefix | Replaces |
|---|---|---|
| `/query` | `/queryFlex` | `/search` |
| `/queryAndFilter` | `/queryFlexAndFilter` | `/searchAndFilter` |
| `/queryAndFilterWithPastorData` | `/queryFlexAndFilterWithPastorData` | `/searchAndFilterWithPastorData` |

**Users**, under `/v1/users`:

| Exact | Prefix | Replaces |
|---|---|---|
| `/query` | `/queryFlex` | `/search` |

The `AndFilter` variants accept `requestedFields` as a query parameter, exactly
as before. The `WithPastorData` variants add the `pic` block and accept
`hasPic=true|false`, exactly as before. Users has no filter or pastor variant,
because `/v1/users/search` never had one.

---

## Request body

Unchanged from `/search`:

```json
{
  "orAnd": "and",
  "params": [
    { "columnName": "provinceCode", "columnValue": "PR0042" },
    { "columnName": "status",       "columnValue": "1" }
  ]
}
```

| Field | Meaning |
|---|---|
| `orAnd` | `"and"` or `"or"` — how the parameters are joined. Honoured by all six. |
| `params[].columnName` | The field to match. `field` is also accepted. |
| `params[].columnValue` | The value. `value` is also accepted. |

> **`columnLogic` is ignored by the new endpoints.** The comparison belongs to
> the endpoint you called. Sending `columnLogic: "LIKE"` to `/query` still gives
> you an exact match, because a caller who wanted LIKE would have called
> `/search`. Only `/search` reads it.

---

## Worked examples

**Find one parish by its code.** The query from the outage, done properly:

```bash
curl -X POST https://<host>/v1/parishdirectory/query \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"orAnd":"and","params":[{"columnName":"parishCode","columnValue":"975462"}]}'
```

**Every active parish in a province, with only the fields you render:**

```bash
curl -X POST "https://<host>/v1/parishdirectory/queryAndFilter?requestedFields=parishCode%20parishName%20status" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"orAnd":"and","params":[
        {"columnName":"provinceCode","columnValue":"PR0042"},
        {"columnName":"status","columnValue":"1"}]}'
```

**Type-ahead on a parish name:**

```bash
curl -X POST https://<host>/v1/parishdirectory/queryFlex \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"orAnd":"and","params":[{"columnName":"parishName","columnValue":"RCCG HOUSE"}]}'
```

**Parishes in a province with no pastor in charge:**

```bash
curl -X POST "https://<host>/v1/parishdirectory/queryAndFilterWithPastorData?hasPic=false" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"orAnd":"and","params":[{"columnName":"provinceCode","columnValue":"PR0042"}]}'
```

**Find one user by username:**

```bash
curl -X POST https://<host>/v1/users/query \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"orAnd":"and","params":[{"columnName":"username","columnValue":"grace.okonkwo"}]}'
```

**Everyone in a province.** The query that scanned 58,057 documents, done
properly:

```bash
curl -X POST https://<host>/v1/users/query \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"orAnd":"and","params":[{"columnName":"province","columnValue":"BE02"}]}'
```

**Type-ahead on a user's first name:**

```bash
curl -X POST https://<host>/v1/users/queryFlex \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"orAnd":"and","params":[{"columnName":"firstName","columnValue":"Grac"}]}'
```

The response is the standard envelope, the same for all eight endpoints:

```json
{ "totalCount": 240, "records": [ … ], "pageNo": 0, "pageSize": 20 }
```

---

## Behaviour you should know about

**`/queryFlex` is case-sensitive.** `"rccg"` will not match `"RCCG HOUSE OF
PRAYER"`. Send the value in the case it is stored. This is not an oversight: it
is the property that keeps the index usable. If you need case-insensitive
matching, use `/search`.

**`/query` matches the whole value.** Half a parish code returns nothing. That
is the contract, not a bug.

**Regex characters in `/queryFlex` values are escaped.** Searching for
`"RCCG (HQ)"` matches the literal text, and a value starting `.*` cannot defeat
the anchor.

**An empty value in `/queryFlex` becomes an exact match on the empty string**
rather than `^`, which would match every document and scan the collection.

**`hasPic` still filters the returned page, not the query.** The appointment
lives in a different collection, so `totalCount` counts parishes matching the
search before the pastor filter, and a filtered page can come back short. The
response says so in `picFilterNote`.

---

## Migrating a screen

Change the path. Nothing else.

```diff
- POST /v1/parishdirectory/searchAndFilter
+ POST /v1/parishdirectory/queryAndFilter
```

Then check two things. That the values you send are complete rather than
partial, since `/query` will not match a fragment. And, if you moved to
`/queryFlex`, that the case matches what is stored.

If a screen genuinely needs "contains" or case-insensitive matching, leave it on
`/search`. It is slow, but it is correct for that job, and it is not going away.

---

## Still on the regex path

Parishes and users are done. These still build every parameter as an unanchored
case-insensitive regex, and each would take the same small change:

`/v1/roles/search`, `/v1/activityLogs/search`, `/v1/permissions/search`,
`/v1/provincedirectory/search`, `/v1/regiondirectory/search`,
`/v1/subcontinentdirectory/search`, `/v1/continentdirectory/search`,
`/v1/userProfiles/search`, `/v1/geofencingExemptions/search`,
`/v1/appIcons/search`, `/v1/birthdayLogs/search`, `/v1/birthdayTemplates/search`,
`/v1/parishdirectoryMonthly/search`.

None of them appeared in the slow-query log for the outage, which is why they
were left rather than changed speculatively. The builders in
`src/utils/SearchHelper.ts` are generic, so adding a pair to any of them is the
same edit made here.

---

## A note on indexes

These endpoints make an index *usable*. They do not create one.

`parishCode`, `username` and `email` are already indexed, so `/query` against
those is fast today. `users.province` had no usable index during the incident,
so `/query` on it is much cheaper than the regex but still not a seek until an
index exists.

Adding that index is a separate, deliberate change. On this codebase it has to
be built out of band rather than declared on the schema, because a
schema-declared index here silently never builds.
