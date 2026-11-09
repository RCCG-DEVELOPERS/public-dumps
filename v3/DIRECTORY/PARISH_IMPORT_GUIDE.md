# Importing parishes from a spreadsheet

Set up a province's zones, areas and parishes in one go — by filling in a
spreadsheet — instead of creating parishes one at a time.

---

## Read this first

- **Nothing is saved when you upload.** Uploading only *checks* your file and
  shows you what would happen. Parishes are created only when you press
  **Confirm**, after you have seen the preview.
- **You import into one province at a time.** You choose the province first;
  the file never mentions it.
- **Every zone needs a zone HQ, and every area needs an area HQ.** This is the
  rule people trip over most — see [Headquarters](#headquarters-the-rule-that-matters-most).
- **Rows with problems are not imported.** The good rows can go ahead; you
  download the rejected ones, fix them, and upload that file again.
- **You can undo an import** — as long as nothing has started to depend on the
  new parishes yet. See [Undoing an import](#undoing-an-import).

**Who can use this (for now):** sub-continent administrators and continent
administrators, within their own sub-continent or continent; and the national
support and ICT teams anywhere. Province and region administrators cannot
import yet.

---

## Contents

1. [The steps](#the-steps)
2. [Filling in the template](#filling-in-the-template)
3. [Headquarters: the rule that matters most](#headquarters-the-rule-that-matters-most)
4. [Codes](#codes)
5. [Reading the preview](#reading-the-preview)
6. [When rows are rejected](#when-rows-are-rejected)
7. [After the import](#after-the-import)
8. [Undoing an import](#undoing-an-import)
9. [Creating a new province or region](#creating-a-new-province-or-region)
10. [Every message, and what to do](#every-message-and-what-to-do)
11. [Questions](#questions)

---

## The steps

| Step | What you do | What happens |
|---|---|---|
| 1. Choose where | Pick the **region**, then the **province**. | Only provinces you are allowed to import into are listed. |
| 2. Get the template | Download the CSV template. Tick *include examples* if you want to see filled-in rows. | Example rows say **EXAMPLE** in every name — delete them before uploading. |
| 3. Fill it in | One row per parish. See [Filling in the template](#filling-in-the-template). | — |
| 4. Save as CSV | In Excel: **File → Save As → CSV UTF-8 (Comma delimited)**. | An `.xlsx` file is refused, with this same instruction. |
| 5. Upload | Upload the CSV. | Every row is checked against the province as it is today. **Nothing is saved.** |
| 6. Read the preview | Check the tree: is each parish going into the right zone and area? Are the HQs right? | See [Reading the preview](#reading-the-preview). |
| 7. Fix or carry on | If rows were rejected, you can import the good rows now and fix the rest later. | See [When rows are rejected](#when-rows-are-rejected). |
| 8. Confirm | Press **Confirm**. | The parishes are created. A large file takes a minute or two; the page shows progress. |
| 9. Download the result | Download **what was imported**. | The real codes every new zone, area and parish was given. |

---

## Filling in the template

One row is one parish.

| Column | Required? | What to put |
|---|---|---|
| `zone_name` | **Yes** | The zone the parish is in. If a zone with this name already exists in the province, the parish joins it. If not, a **new zone** is created. |
| `zone_code` | No | Only needed when **two zones in the province have the same name** — put the code of the one you mean. Leave blank for a new zone. |
| `area_name` | **Yes** | The area the parish is in. Matched **within its zone** — "Area 1" in two different zones is two different areas. A new name creates a **new area**. |
| `area_code` | No | Only needed when two areas in the same zone share a name. Leave blank for a new area. |
| `parish_name` | **Yes** | The parish's official name. |
| `zone_hq` | See below | **Yes** if this parish is the zone's headquarters, otherwise **No**. Blank means No. |
| `area_hq` | See below | **Yes** if this parish is the area's headquarters, otherwise **No**. Blank means No. |
| `parish_alias` | No | A name people commonly use for the parish. |
| `address`, `city`, `state`, `postal_code`, `landmark` | No | The parish's location. |
| `country` | No | Leave blank to use the province's country. |
| `latitude`, `longitude` | No | Decimal numbers, e.g. `6.5244` and `3.3792`. |
| `parish_email`, `parish_phone` | No | — |

**Things that do not matter** (so you don't have to worry about them): capital
letters and extra spaces in names — `victory  zone` matches `Victory Zone`.
Column headings can be in any case.

**Things the file must not have:** a province column (you chose the province
already), parish codes (they are generated), or province/region HQ columns
(those are set when a province or region is created — see
[Creating a new province or region](#creating-a-new-province-or-region)).

### Example

| zone_name | area_name | parish_name | zone_hq | area_hq |
|---|---|---|---|---|
| Victory Zone | Grace Area | Mercy Parish | No | No |
| Victory Zone | Peace Area | Peace Parish | No | Yes |
| Hope Zone | Hope Area 1 | Hope Parish | **Yes** | **Yes** |
| Hope Zone | Hope Area 1 | Joy Parish | No | No |
| Hope Zone | Hope Area 2 | Faith Parish | No | **Yes** |

Here *Victory Zone* and *Grace Area* already exist, so Mercy Parish joins them.
*Peace Area* is new, inside the existing Victory Zone, with Peace Parish as its
HQ. *Hope Zone* is new, with two new areas: Hope Parish heads the zone **and**
Hope Area 1; Faith Parish heads Hope Area 2.

---

## Headquarters: the rule that matters most

> **Every zone has exactly one zone HQ. Every area has exactly one area HQ.**
> A zone HQ is automatically the area HQ of its own area.
> A zone can have many areas — each with its own area HQ.

In practice:

- **A new zone** — one of its parishes must say **Yes** under `zone_hq`. Exactly one.
- **A new area** — one of its parishes must say **Yes** under `area_hq`.
  Exactly one. If the zone HQ is in this area, that already counts: the zone HQ
  *is* the area HQ, so don't mark a second parish in that area.
- **An existing zone or area that already has an HQ** — leave its HQ columns
  as **No**. An import never moves an HQ. To change one, use *HQ assignments*.
- **An existing zone or area with no HQ** — you may name one in the file. The
  preview says so ("assigns HQ to …").

If you put **Yes** for zone HQ and **No** for area HQ on the same row, the area
HQ is set to Yes for you, and the preview tells you.

**If a new zone's HQ row is rejected** (say, a mistyped email), the whole zone
is held back, because a zone cannot exist without its HQ. Fix that one row and
the rest of the zone goes through with it. The same applies to a new area.

---

## Codes

Codes are generated for you for every new **zone, area and parish**.

Codes for a new **province or region** are chosen by you (see
[Creating a new province or region](#creating-a-new-province-or-region)), and
must follow one rule:

> **Letters and digits only, in capitals. No spaces, no hyphens, nothing else.**
> For example: `UKREG01`, `LA99`.

Small letters are changed to capitals for you. Anything else is refused, with a
suggestion — `LA 99-B` is refused with *"Did you mean LA99B?"* — rather than
changed silently, because quietly removing characters could make your new code
the same as one that already exists.

---

## Reading the preview

The preview shows the province as a tree — **zones**, then their **areas**,
then the **parishes** in each — so you check the *shape* before the numbers:

```
LAGOS PROVINCE 47
├── Hope Zone                     NEW   HQ: Hope Parish
│   ├── Hope Area 1               NEW   HQ: Hope Parish
│   │   ├── Hope Parish           ✓ will be created
│   │   └── Joy Parish            ✓ will be created
│   └── Hope Area 2               NEW   HQ: Faith Parish
│       └── Faith Parish          ✓ will be created
└── Victory Zone   ZN8000027650   existing   HQ: Province HQ Parish
    ├── Grace Area AR8000000211   existing
    │   ├── Mercy Parish          ✓ will be created
    │   └── Mercy  Parish         ✗ duplicate of row 2
    └── Peace Area                NEW   HQ: Peace Parish
        └── Peace Parish          ✓ will be created
```

Check in particular:

- Are new zones and areas really new — or has a typo created a "new" zone that
  should have been an existing one?
- Is every HQ the parish you expected?
- The counts: how many will be created, how many were rejected, and why.

Each row has one of these statuses:

| Status | Meaning |
|---|---|
| **Valid** | Will be created when you confirm. |
| **Already exists** | A parish of that name is already in that area. Skipped — not an error. |
| **Duplicate** | The same parish appears twice in your file. The first copy is kept. |
| **Invalid** | Something must be fixed first. The reason is shown on the row. |

A **warning** (for example *"a parish with this name already exists elsewhere in
the province"*) does not stop a row; it asks you to double-check.

---

## When rows are rejected

You do **not** have to fix everything before importing.

1. Confirm — only the **Valid** rows are created.
2. Download **Rejected rows**. It is your own file, with only the rows that did
   not go in, and a column explaining why.
3. Fix those rows **in that same file**. Leave the extra `import_status` and
   `import_errors` columns in — they are ignored when you upload.
4. Upload it again and confirm.

Uploading a row that was already imported is safe: it shows as **Already
exists** and is skipped. So if you are unsure, uploading the whole file again
does no harm.

---

## After the import

- **Download "what was imported"** — the codes every new zone, area and parish
  was given, read back from the directory, so it is what is really there.
- **Import history** lists every import in your sub-continent: who did it, when,
  into which province, how many parishes, and what happened. Each one keeps four
  downloads: the **original file**, the **full report** (every row and what
  happened to it), the **rejected rows**, and **what was imported**.

If an import is interrupted (for example the system restarts), open it in the
history and press **Confirm** again. It carries on from where it stopped and
never creates the same parish twice.

---

## Undoing an import

An import can be undone — removing exactly the parishes it created, and
nothing else. Press **Undo** and you first get a preview of what would be
removed.

An undo is **refused** if any of the new parishes has, since the import:

- gained **members**,
- been given a **pastor in charge**, a **principal officer** or a **secondary role**,
- been **edited**, **moved**, or had its **HQ changed**,
- been **deleted** —

or if someone has added a parish of their own to a zone or area the import
created. The preview lists every reason. It is all or nothing: once people
depend on a parish, removing it is a decision for a person, not a button.

Every parish an undo removes is kept in the **deletions log**, so it can still
be restored from there.

---

## Creating a new province or region

A province or region comes into existence with its **headquarters parish** —
there is no such thing as an empty province.

**New province** (under an existing region). For each province give:

- its **name** and **code** (letters and digits, e.g. `LA99`);
- its HQ parish: the **parish name**, and the names of its first **zone** and
  first **area**, plus an address if you have it.

The HQ parish becomes the HQ of the province, its first zone and its first area
all at once. Zone, area and parish codes are generated.

**New region** (under your sub-continent). For each region give its name and
code, its **first province** (name and code), and its HQ parish as above. The
HQ parish heads the region, the province, the zone and the area.

You can create up to 50 at a time. Everything is checked first — names already
used, codes already taken, codes with spaces — and if anything is wrong,
**nothing** is created and every problem is listed. A preview is shown before
anything is saved. Each setup appears in the import history and can be undone
the same way.

Then import the rest of the province's parishes with the spreadsheet.

---

## Every message, and what to do

### Problems with the file

| Message (code) | What to do |
|---|---|
| This is an Excel workbook, not a CSV (`NOT_CSV`) | Save as **CSV UTF-8 (Comma delimited)** and upload that. |
| The file is not saved as UTF-8 (`NOT_UTF8`) | Same — choose **CSV UTF-8** specifically. |
| The file is … KB; the limit is 5 MB (`FILE_TOO_LARGE`) | Split it into smaller files. |
| The file has … rows; the limit is 5000 (`TOO_MANY_ROWS`) | Split it into smaller files. |
| The required column '…' is missing (`MISSING_COLUMN`) | Start from the template; don't rename `zone_name`, `area_name` or `parish_name`. |
| Column '…' is not part of the template (`UNKNOWN_COLUMN`) | Remove or rename it. The message lists the allowed columns. |
| Column 'phq' / 'parish_code' / 'province_code' (`REFUSED_COLUMN`) | Remove it — the message says why. |
| Columns '…' and '…' both mean … (`DUPLICATE_COLUMN`) | Keep one of them. |

### Problems with a row

| Message (code) | What to do |
|---|---|
| Zone name / area name / parish name is required (`REQUIRED_FIELD`) | Fill it in. |
| Zone HQ must be Yes or No (`INVALID_YES_NO`) | Use **Yes** or **No**. |
| … zones in this province are called '…' (`AMBIGUOUS_ZONE_NAME`) | Put the right zone's code in `zone_code` (the message lists them). |
| … areas in zone '…' are called '…' (`AMBIGUOUS_AREA_NAME`) | Put the right area's code in `area_code`. |
| No zone with code … exists in this province (`ZONE_NOT_IN_PROVINCE`) | Check the code, or leave `zone_code` blank to create a new zone. |
| Area … is not in zone '…' (`AREA_NOT_IN_ZONE`) | The message says which zone it is really in. |
| Zone … is also in use under province … (`ZONE_UNDER_OTHER_PROVINCE`) | Only while `HIERARCHY_UNIQUE_PARENT_ENFORCED` is on. The zone is split across provinces; have it repaired in Directory integrity first. |
| Area … is also in use under zone … (`AREA_UNDER_OTHER_ZONE`) | Only while `HIERARCHY_UNIQUE_PARENT_ENFORCED` is on. The area is split across zones; have it repaired in Directory integrity first. |
| The zone code may contain only letters and digits (`INVALID_CODE_FORMAT`) | Use the suggested code. |
| New zone '…' has no zone HQ (`NEW_ZONE_NEEDS_HQ`) | Set `zone_hq` to Yes on exactly one of its parishes. |
| New area '…' has no area HQ (`NEW_AREA_NEEDS_HQ`) | Set `area_hq` to Yes on exactly one of its parishes. |
| New zone '…' has 2 zone HQ rows (`MULTIPLE_ZONE_HQ`) | Keep **Yes** on one only. |
| New area '…' has 2 area HQ rows (`MULTIPLE_AREA_HQ`) | Keep **Yes** on one only — remember the zone HQ already counts for its own area. |
| Zone / area '…' already has an HQ (`HQ_ALREADY_HELD`) | Set the HQ column to **No**. To move an HQ, use HQ assignments. |
| … cannot be created: the row marked as its HQ was rejected (`HQ_ROW_REJECTED`) | Fix the HQ row (its own message says how); this row then goes through too. |
| The same parish is already on row … (`DUPLICATE_IN_FILE`) | Delete the extra copy. |
| … is not an email address (`INVALID_EMAIL`) | Correct or clear it. |
| latitude must be a number between -90 and 90 (`INVALID_COORDINATE`) | Correct or clear it. |
| … is … characters; the limit is … (`FIELD_TOO_LONG`) | Shorten it. |

### Warnings (the row still goes in)

| Warning | Meaning |
|---|---|
| Area HQ has been set to Yes (`HQ_CASCADED`) | A zone HQ is always its area's HQ. |
| Already in this area; skipped (`ALREADY_EXISTS`) | Nothing to do. If it says *deleted*, restore that parish instead. |
| A parish with this name already exists in this province (`POSSIBLE_DUPLICATE`) | Check it is not the same church. |
| Zone … is called '…' in the directory (`ZONE_NAME_DIFFERS`) | The directory's name is used. |
| New zone will be named '…', as first written (`NAME_SPELLING`) | Spell it the same way throughout. |
| Repeated spaces … single spaces (`NAME_TIDIED`) | Nothing to do. |

### When confirming

| Message (code) | What to do |
|---|---|
| The directory has changed since this preview (`PREVIEW_STALE`) | Someone changed the province after you uploaded. The preview has been refreshed — check it and confirm again. |
| Another import into … is running (`PROVINCE_BUSY`) | Wait for it to finish, then confirm. |
| This preview is more than seven days old (`BATCH_EXPIRED`) | Upload the file again. |
| The preview you confirmed showed … rows (`PREVIEW_COUNT_MISMATCH`) | Reload the preview and confirm what it shows. |

---

## Questions

**I uploaded and nothing appeared in the directory.**
That is expected — uploading only checks. Press **Confirm**.

**Can I import into several provinces at once?**
No — one province per file. That keeps every row's zone and area unambiguous.

**A zone name exists twice in my province. What do I do?**
Put the code of the one you mean in `zone_code`. The message lists both codes.

**I made a mistake after importing.**
If nothing depends on the new parishes yet, **Undo** the import, fix the file,
and import again. If it is just one parish, edit it in the directory instead.

**Will importing the same file twice create duplicates?**
No. Parishes already in the same area are shown as *Already exists* and
skipped.

**Can I move an HQ with an import?**
No. Imports never change an existing HQ. Use HQ assignments.
