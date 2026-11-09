# Parish import — API for the wizard

The endpoints behind the guided parish import and the region/province setup
forms, for the team building the wizard. The administrator-facing explanation
of the same feature is [PARISH_IMPORT_GUIDE.md](PARISH_IMPORT_GUIDE.md); the
wizard's copy should follow it.

Every payload below was captured from a run against a local database, not
written by hand. Codes are from that run.

---

## Read this first

- **Replace the existing bulk upload.** `src/content/common/directory/BulkParishUpload/`
  in `remittance-frontend-v2` parses the file in the browser and posts one
  `POST /v1/parishDirectory` per row — no hierarchy rules, no HQ rules, no
  history. Retire it in favour of this.
- **Validate never writes.** `POST import/validate` stores a batch and returns a
  preview. The directory changes only on `confirm`.
- **Confirm returns `202` and runs in the background.** Poll the batch.
- **Errors carry `code` and often `detail`.** Branch on `code`, never on
  `message`. `detail.resendWith` is the exact body to retry with.
- **Access:** sub-continent administrators and above (`PARISH_IMPORT_ROLES`,
  default `sub-cont-admin,cont-admin`, plus super-admin and `ELEVATED_ROLES`).
  Anyone else gets `403 IMPORT_NOT_PERMITTED` — hide the wizard for them.

All paths are under **`/v1/parishDirectory`**, bearer token as usual.

| Method | Path | Wizard step |
|---|---|---|
| GET | `import/scope` | 1 — choose region and province |
| GET | `import/template` | 2 — download template |
| GET | `import/template/columns` | 2 — render the column guide |
| POST | `import/validate` | 3–4 — upload, get the preview |
| GET | `import/batches/:batchId` | 4, 6, 7 — preview, progress, result |
| GET | `import/batches/:batchId/rows` | 4 — paged rows, filter by status |
| POST | `import/batches/:batchId/confirm` | 5 — confirm |
| POST | `import/batches/:batchId/cancel` | — abandon |
| GET | `import/batches/:batchId/download/:kind` | 7 — downloads |
| POST | `import/batches/:batchId/undo` | history — undo |
| GET | `import/batches` | history |
| POST | `hierarchy/provinces` | setup form |
| POST | `hierarchy/regions` | setup form |

---

## The batch status machine

```
                 upload
                   │
        ┌──────────┴──────────┐
   VALIDATED              READY_FOR_IMPORT ──cancel──► CANCELLED
 (nothing valid)                │  └─(7 days)──────► EXPIRED
                         confirm│
                                ▼
                           PROCESSING ──(stalled 2 min)──┐
                     ┌──────────┼──────────┐             │ confirm
                COMPLETED  COMPLETED_   FAILED ──────────┘ again
                     │     WITH_ERRORS     (resumes, never writes twice)
                     └──────┬─────┘
                       undo │
                            ▼
                        UNDOING ──► UNDONE
```

| Status | Show |
|---|---|
| `VALIDATED` | The reasons; offer *Download rejected rows*. Confirm disabled. |
| `READY_FOR_IMPORT` | The tree and counts; Confirm enabled. |
| `PROCESSING` | Progress from `counts.processed` / `counts.pending`. Poll every 2–3 s. |
| `COMPLETED` | Success; offer *Download what was imported*. |
| `COMPLETED_WITH_ERRORS` | As above, plus *Download rejected rows*. |
| `FAILED` | `batch.error`; offer **Resume** (= confirm again). |
| `UNDOING` / `UNDONE` | Progress / done. If `UNDOING` carries `error`, offer the undo again. |

Every batch response carries `next` — one sentence saying what to do next.
Showing it as-is is a reasonable default.

---

## Step 1 — `GET import/scope`

A sub-continent administrator gets their regions and provinces straight away:

```json
{
  "level": "region",
  "subContinent": { "code": "CNT03SUBCNT01", "name": "WEST AFRICA", "continentCode": "CNT03", "continentName": "AFRICA" },
  "regions": [
    { "code": "R07", "name": "REGION 7",
      "provinces": [ { "code": "LA47", "name": "LAGOS PROVINCE 47", "parishes": 1 } ] }
  ]
}
```

A continent administrator or the national desk first gets
`{ "level": "sub-continent", "subContinents": [ { code, name, continentCode, continentName, parishes } ] }`;
call again with `?subContinentCode=…` to drill in.

---

## Step 2 — template

- `GET import/template` → `text/csv` attachment, headers only.
  `?examples=true` adds four example rows whose names start with `EXAMPLE`.
- `GET import/template/columns` → JSON: `columns[]` (`key`, `label`, `required`,
  `aliases`, `description`, `example`, `maxLength`) and `rules` (plain-language
  strings: province, matching, headquarters[], codes, duplicates, limits). Render
  both beside the download.

HQ columns take **Yes/No** (blank = No). `1/0`, `y/n` and `true/false` are also
accepted. Headers are matched case-insensitively; `zhq`/`ahq` work as aliases.

---

## Steps 3–4 — `POST import/validate`

`multipart/form-data`: **`file`** (the CSV) and **`provinceCode`**. Returns
`201`. Limits: `.csv`, 5 MB, 5,000 rows, UTF-8.

Response (trimmed — `rows.records` normally holds the first 50):

```json
{
  "batch": {
    "batchId": "IMP-20261108-889602",
    "kind": "CSV_IMPORT",
    "status": "READY_FOR_IMPORT",
    "region": { "code": "R07", "name": "REGION 7" },
    "province": { "code": "LA47", "name": "LAGOS PROVINCE 47" },
    "subContinent": { "code": "CNT03SUBCNT01", "name": "WEST AFRICA" },
    "filename": "lagos-47.csv",
    "fileBytes": 413,
    "counts": { "imported": 0, "failed": 0, "total": 7, "valid": 5, "exists": 0, "invalid": 1, "duplicate": 1, "withWarnings": 1 },
    "plan": {
      "newZones": [ { "zoneId": "new:hope zone", "name": "Hope Zone", "rows": 3 } ],
      "newAreas": [
        { "areaId": "code:ZN8000027650|new:peace area", "zoneName": "Victory Zone", "name": "Peace Area", "rows": 1 },
        { "areaId": "new:hope zone|new:hope area 1", "zoneName": "Hope Zone", "name": "Hope Area 1", "rows": 2 },
        { "areaId": "new:hope zone|new:hope area 2", "zoneName": "Hope Zone", "name": "Hope Area 2", "rows": 1 }
      ],
      "newParishes": 5,
      "hqAssignments": [
        { "level": "area", "unitName": "Peace Area", "newUnit": true, "parishName": "Peace Parish", "rowNumber": 3 },
        { "level": "zone", "unitName": "Hope Zone", "newUnit": true, "parishName": "Hope Parish", "rowNumber": 4 },
        { "level": "area", "unitName": "Hope Area 1", "newUnit": true, "parishName": "Hope Parish", "rowNumber": 4 },
        { "level": "area", "unitName": "Hope Area 2", "newUnit": true, "parishName": "Faith Parish", "rowNumber": 6 }
      ]
    },
    "fileErrors": [],
    "createdBy": "sco.westafrica",
    "createdAt": "2026-11-08T05:38:17.549Z",
    "expiresAt": "2026-11-15T05:38:17.549Z"
  },
  "tree": { "province": { "code": "LA47", "name": "LAGOS PROVINCE 47", "region": { "code": "R07", "name": "REGION 7" } },
            "zones": [ "…see below…" ], "unplaced": [] },
  "rows": { "totalCount": 7, "records": [ "…see Rows…" ], "pageNo": 1, "pageSize": 50 },
  "next": "5 parish(es) are ready. Check the tree, then confirm with expectedValidRows: 5. Rejected rows will not be imported; download them to fix later.",
  "confirmWith": { "expectedValidRows": 5 }
}
```

### The tree — render this as the preview

```json
{
  "zoneId": "code:ZN8000027650", "code": "ZN8000027650", "name": "Victory Zone", "isNew": false,
  "existingHq": { "parishCode": "211549", "parishName": "Province HQ Parish" },
  "plannedHq": null,
  "counts": { "valid": 2, "exists": 0, "invalid": 0, "duplicate": 1 },
  "areas": [
    { "areaId": "code:ZN8000027650|code:AR8000000211", "code": "AR8000000211", "name": "Grace Area", "isNew": false,
      "existingHq": null, "plannedHq": null,
      "counts": { "valid": 1, "exists": 0, "invalid": 0, "duplicate": 1 },
      "parishes": [
        { "rowNumber": 2, "parishName": "Mercy Parish", "status": "VALID", "zoneHq": false, "areaHq": false, "errors": [], "warnings": [] },
        { "rowNumber": 8, "parishName": "Mercy Parish", "status": "DUPLICATE", "zoneHq": false, "areaHq": false,
          "errors": ["The same parish is already on row 2 (same zone, area and name)."], "warnings": [] }
      ] }
  ]
}
```

- `code: null` with `isNew: true` → show **NEW** ("code assigned on import").
  After the import, `GET import/batches/:batchId` returns the same tree with
  the issued codes filled in.
- `existingHq` is the unit's current HQ; `plannedHq` is what this file makes
  its HQ.
- `tree.unplaced` holds rows that could not be put in a zone at all (an
  ambiguous zone name, a code not in the province). Show them under their own
  heading.
- Zones and areas arrive sorted by name.

### File-level problems

If the file itself is wrong, no row is planned: `status` is `VALIDATED`,
`counts` are all zero, and `fileErrors` explains:

```json
"fileErrors": [
  { "code": "REFUSED_COLUMN", "field": "phq",
    "message": "Column 'phq': Province, region and higher HQs are set when a province or region is created, not by an import." },
  { "code": "MISSING_COLUMN", "field": "parish_name", "message": "The required column 'parish_name' is missing." }
]
```

Upload-level refusals come back as HTTP errors before any batch is stored:
`400 FILE_REQUIRED`, `400 NOT_CSV` (including an `.xlsx`), `400 NOT_UTF8`,
`400 FILE_TOO_LARGE`, `400 PROVINCE_REQUIRED`, `404 PROVINCE_NOT_FOUND`,
`409 PROVINCE_INCONSISTENT`, `403 IMPORT_NOT_PERMITTED`, `403 OUTSIDE_YOUR_UNIT`.

---

## Rows — `GET import/batches/:batchId/rows?status=INVALID,DUPLICATE&pageNo=1&pageSize=50`

Paged, the usual `{ totalCount, records, pageNo, pageSize }`. One record:

```json
{
  "rowNumber": 7,
  "status": "INVALID",
  "cells": [ { "header": "zone_name", "value": "Light Zone" }, { "header": "area_name", "value": "Light Area" },
             { "header": "parish_name", "value": "Light Parish" }, { "header": "zone_hq", "value": "No" },
             { "header": "area_hq", "value": "No" }, { "header": "address", "value": "" }, { "header": "city", "value": "Epe" } ],
  "values": { "zone_name": "Light Zone", "area_name": "Light Area", "parish_name": "Light Parish", "zone_hq": "No", "area_hq": "No", "address": "", "city": "Epe" },
  "errors": [ { "code": "NEW_ZONE_NEEDS_HQ", "field": "zone_hq",
                "message": "New zone 'Light Zone' has no zone HQ. Set Zone HQ to Yes on exactly one of its parishes." } ],
  "warnings": [],
  "resolved": { "zoneId": "new:light zone", "zoneCode": "", "zoneName": "Light Zone", "isNewZone": true,
                "areaId": "new:light zone|new:light area", "areaCode": "", "areaName": "Light Area", "isNewArea": true,
                "askedZoneHq": false, "askedAreaHq": false, "flags": { "zhq": "0", "ahq": "0" },
                "fields": { "city": "Epe", "country": "Nigeria" }, "parishName": "Light Parish" },
  "outcome": {}
}
```

`rowNumber` is the spreadsheet row (the header is row 1). `cells` are exactly
what was uploaded; `errors[].field` names the column to highlight. After the
import, `outcome` carries `parishCode`, `zoneCode`, `areaCode` and `parishId`.

Statuses: `VALID`, `EXISTS`, `INVALID`, `DUPLICATE`, then `IMPORTED`, `FAILED`
or `UNDONE`. Every error and warning code is listed with its fix in
[the guide](PARISH_IMPORT_GUIDE.md#every-message-and-what-to-do).

---

## Step 5 — `POST import/batches/:batchId/confirm`

```json
{ "expectedValidRows": 5 }
```

Send `confirmWith.expectedValidRows` from the preview. Returns **`202`** with
the batch in `PROCESSING`; then poll `GET import/batches/:batchId`.

| Response | Meaning | Do |
|---|---|---|
| `409 PREVIEW_COUNT_MISMATCH` | The number sent is not the preview's. `detail.resendWith` holds the right body. | Reload the preview. |
| `409 PREVIEW_STALE` | The directory changed since the upload; the batch has been **re-validated**. `detail.batch` is the refreshed summary, `detail.resendWith` the new body (null if nothing is importable now). | Refetch the batch and show the new preview. |
| `409 PROVINCE_BUSY` | Another import into the province is running. | Ask the user to wait. |
| `409 ALREADY_PROCESSING` | Already confirmed (double click, second tab). | Just poll. |
| `409 BATCH_EXPIRED` | Preview older than 7 days. | Upload again. |
| `409 NOT_CONFIRMABLE` | Wrong status; `message` includes the next step. | — |
| `503 IMPORT_NOT_READY` | Required indexes are missing on the server. `detail.missing` lists them. | An operator must run `scripts/createParishImportIndexes.ts`. |

```json
{ "status": 409, "name": "HttpError", "code": "PREVIEW_COUNT_MISMATCH",
  "message": "The preview you confirmed showed 99 importable row(s); this batch has 5. Reload the preview and confirm what it shows.",
  "detail": { "resendWith": { "expectedValidRows": 5 } } }
```

**Resuming:** a batch in `FAILED`, or `PROCESSING` with no heartbeat for two
minutes, is resumed by sending the same confirm. Nothing already written is
written again.

### Progress and result — `GET import/batches/:batchId`

```json
{
  "batch": { "batchId": "IMP-20261108-889602", "status": "COMPLETED",
             "counts": { "imported": 5, "failed": 0, "total": 7, "valid": 5, "exists": 0, "invalid": 1,
                         "duplicate": 1, "withWarnings": 1, "pending": 0, "processed": 5 },
             "confirmedBy": "sco.westafrica", "startedAt": "…", "finishedAt": "…", "…": "…" },
  "tree": { "…": "as at validation, with issued zone and area codes filled in" },
  "next": "Imported. Download what was written with the `imported` download.",
  "confirmWith": null
}
```

While `PROCESSING`, `counts.processed` and `counts.pending` advance every 25 rows.

---

## Step 7 — downloads — `GET import/batches/:batchId/download/:kind`

All `text/csv; charset=utf-8` attachments with a byte-order mark (so Excel reads
names correctly). Cells that would be spreadsheet formulas are prefixed with `'`.

| `kind` | Contents |
|---|---|
| `original` | The file exactly as uploaded. (`404 NO_FILE` for a setup batch.) |
| `report` | Every row: `row, import_status, zone_name, zone_code, area_name, area_code, parish_name, parish_code, zone_hq, area_hq, import_errors, import_warnings`. |
| `rejected` | The uploaded columns plus `import_status, import_errors`, only rows not imported. **Re-uploadable as is** — the two extra columns are ignored. |
| `imported` | What is actually in the directory, read back: `row, in_directory, parish_code, parish_name, zone_code, zone_name, area_code, area_name, zone_hq, area_hq, province_code, province_name, region_code, region_name, address, city, state, country, parish_id`. |

`imported` from the run above:

```csv
"row","in_directory","parish_code","parish_name","zone_code","zone_name","area_code","area_name","zone_hq","area_hq",…
"3","Yes","547768","Peace Parish","ZN8000027650","Victory Zone","AR4982027903","Peace Area","No","Yes",…
"4","Yes","901761","Hope Parish","ZN6476339626","Hope Zone","AR1348491177","Hope Area 1","Yes","Yes",…
"6","Yes","801977","Faith Parish","ZN6476339626","Hope Zone","AR3527436522","Hope Area 2","No","Yes",…
```

`rejected` from the same run:

```csv
"zone_name","area_name","parish_name","zone_hq","area_hq","address","city","import_status","import_errors"
"Light Zone","Light Area","Light Parish","No","No","","Epe","INVALID","New zone 'Light Zone' has no zone HQ. Set Zone HQ to Yes on exactly one of its parishes."
"Victory Zone","Grace Area","Mercy  Parish","No","No","","Ikeja","DUPLICATE","The same parish is already on row 2 (same zone, area and name)."
```

After an undo, `imported` still lists every row with `in_directory` set to
`No — removed by undo`.

---

## History — `GET import/batches?status=&provinceCode=&kind=&pageNo=&pageSize=`

Paged `{ totalCount, records, pageNo, pageSize }`, newest first; each record is
a batch summary (no tree). Scoped to the caller: a sub-continent administrator
sees their sub-continent's batches, a continent administrator their continent's,
the national desk everything. A batch outside the caller's scope answers
`404 BATCH_NOT_FOUND`, the same as one that does not exist.

`POST import/batches/:batchId/cancel` abandons a `READY_FOR_IMPORT` or
`VALIDATED` batch (`409 NOT_CANCELLABLE` otherwise).

---

## Undo — `POST import/batches/:batchId/undo`

```json
{ "dryRun": true }
```

Dry run is the default. Response:

```json
{ "wouldRemove": 5, "alreadyGone": [], "blockers": [], "dryRun": true,
  "next": "Send the same request with dryRun: false to remove these 5 parish(es)." }
```

`blockers[]` items are `{ rowNumber, parishCode, code, message }` with codes
`HAS_MEMBERS`, `HAS_PIC`, `HAS_OFFICER`, `HAS_ROLE_GRANT`, `EDITED_SINCE`,
`MOVED_SINCE`, `HQ_CHANGED_SINCE`, `DELETED_SINCE`, `UNIT_IN_USE`. Show them as a
list; the undo button stays disabled while any remain.

`{ "dryRun": false }` → `202`, status `UNDOING`; poll until `UNDONE`. With
blockers it answers `409 UNDO_BLOCKED` with `detail.blockers`. Only
`COMPLETED`/`COMPLETED_WITH_ERRORS` batches can be undone (`409 NOT_UNDOABLE`).

---

## Setup forms

### `POST hierarchy/provinces`

```json
{
  "regionCode": "R07",
  "dryRun": true,
  "provinces": [
    { "provinceName": "Lagos Province 99", "provinceCode": "la99",
      "hqParish": { "parishName": "Jesus House", "zoneName": "Zone 1", "areaName": "Area 1", "city": "Ikeja" } }
  ]
}
```

`hqParish` also takes `parishAlias, address, state, country, postalCode,
landmark, latitude, longitude, parishEmail, parishPhone`. Up to 50 provinces.
Dry run (`200`):

```json
{
  "dryRun": true,
  "region": { "code": "R07", "name": "REGION 7" },
  "wouldCreate": [ {
    "item": 1,
    "province": { "code": "LA99", "name": "Lagos Province 99" },
    "zone": { "name": "Zone 1", "code": "generated on create" },
    "area": { "name": "Area 1", "code": "generated on create" },
    "hqParish": { "name": "Jesus House", "code": "generated on create", "headquartersOf": ["province", "zone", "area"] }
  } ],
  "next": "Send the same request with dryRun: false to create 1 province(s)."
}
```

With `"dryRun": false` → `201` with `batch` (a `HIERARCHY_SETUP` batch, in the
same history, undoable the same way) and `created[]`, each item carrying the
real codes for province, zone, area and HQ parish — or `failed: true` and
`error` for an item that could not be written.

### `POST hierarchy/regions`

```json
{
  "subContinentCode": "CNT03SUBCNT01",
  "dryRun": true,
  "regions": [ {
    "regionName": "Region 99", "regionCode": "r99",
    "firstProvince": { "provinceName": "Region 99 Province 1", "provinceCode": "r99p1" },
    "hqParish": { "parishName": "…", "zoneName": "…", "areaName": "…" }
  } ]
}
```

The HQ parish heads the region, the province, the zone and the area.

### Validation errors

Everything is checked before anything is written; any problem returns
`400 SETUP_INVALID` with every problem in `detail.errors`, and nothing is
created:

```json
{ "status": 400, "code": "SETUP_INVALID",
  "message": "1 problem(s) — nothing was created. Item 1: The province code 'LA 99-B' may contain only letters and digits — no spaces, hyphens or other characters. Did you mean LA99B?",
  "detail": { "errors": [ { "index": 0, "field": "provinceCode", "code": "INVALID_CODE_FORMAT",
    "message": "The province code 'LA 99-B' may contain only letters and digits — no spaces, hyphens or other characters. Did you mean LA99B?" } ] } }
```

Error codes: `INVALID_CODE_FORMAT` (show the suggestion), `CODE_TAKEN`,
`NAME_TAKEN`, `DUPLICATE_IN_REQUEST`. `index` is zero-based into the array you
sent; `field` names the input to highlight. Codes are upper-cased for you, so
show the typed value back in capitals.
