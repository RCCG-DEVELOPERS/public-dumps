# App Icons — API Reference

The application menu, served from the Directory in eight languages.

- [What changed and why](#what-changed-and-why)
- [Translation model](#translation-model)
- [Endpoints](#endpoints)
- [Link and image rules](#link-and-image-rules)
- [Error codes](#error-codes)
- [Frontend guidance](#frontend-guidance)

---

## What changed and why

The menu was a **171-row CSV compiled into the frontend bundle**
(`public/app_icons.csv` → `src/utils/appIconsFallback.js`). Adding an
application, reordering the menu, or changing who sees what needed a release.

It is now a collection the Directory serves. **The deployed frontend needs no
change**: the path and the legacy `snake_case` response fields are preserved
exactly, and the CSV stays as the offline fallback.

---

## Translation model

**Every icon carries all eight labels.** Switching language is instant, needs no
refetch, and works against a cached payload offline.

Embedded rather than resolved from the frontend catalog because that catalog is
built by **static extraction from `t()` calls** — it cannot see values that only
exist in a database. A catalog approach would need a release every time an
administrator added an icon, which is the thing this feature exists to stop.

| Field | Meaning |
|---|---|
| `translations` | `{ en, fr, es, pt, de, ja, sw, zh }` — the labels |
| `translationCode` | a stable id (`ICT_APP_NAME`) that survives a label being rewritten |
| `machineTranslated` | locales whose text is **unreviewed machine output** |
| `label` | the label for the requested language, already resolved |
| `defaultName` | the English fallback of last resort |

Resolution is **requested locale → `en` → `defaultName`**, so a missing
translation never renders an empty menu item.

> **`machineTranslated` matters.** Seeding fills seven languages at once from a
> machine-translation file. Without this list there is no way to tell an approved
> Japanese label from a guessed one. Editing a locale through the translations
> endpoint marks it reviewed and removes it from the list.

---

## Endpoints

Mount `/v1/app-icons`, bearer token required.

| Route | Auth |
|---|---|
| `GET /me` | any authenticated user |
| everything else | **super-admin** |

### GET /v1/app-icons/me

The user/caller's menu — scoped to their continent/sub-continent, filtered by role,
sorted by `displayOrder`.

```
GET /v1/app-icons/me
GET /v1/app-icons/me?lang=zh
```

```jsonc
{
  "status": "success",
  "data": [
    {
      "id": "65a1…", "code": "ICT", "slug": "ict",
      "name": "Information Technology",
      "defaultName": "Information Technology",
      "translationCode": "ICT_APP_NAME",
      "label": "信息技术",                    // resolved for ?lang=zh
      "translations": { "zh": "信息技术", "en": "Information Technology" },
      "machineTranslated": ["ja", "sw", "zh"],
      "iconKey": "default_app",
      "path": "/static/images/menu/ict.png",
      "menuLink": "/v2/ict",  "menu_link": "/v2/ict",
      "displayOrder": 10,     "sort_order": 10,
      "parentId": null,       "parent_id": null,
      "level": 1, "type": 1,
      "isActive": true,       "active": 1,
      "scope": "GLOBAL"
    }
  ]
}
```

**No `lang`** returns all eight translations — the default, because a language
switch then costs nothing. **With `lang`** the payload narrows to that locale
**plus English**, so a client that falls back has something to fall back to.

`scope` is `GLOBAL`, `CONTINENT` or `SUBCONTINENT`, showing why the icon reached
this user.

### Administration

```
GET    /v1/app-icons                     list
GET    /v1/app-icons/:id                 one
POST   /v1/app-icons                     create
PATCH  /v1/app-icons/:id                 update
DELETE /v1/app-icons/:id                 soft delete (isActive:false + deletedAt)
PATCH  /v1/app-icons/reorder             bulk displayOrder
PUT    /v1/app-icons/:id/translations    merge labels
POST   /v1/app-icons/:id/continents      assign scope
DELETE /v1/app-icons/:id/continents/:continentId
POST   /v1/app-icons/:id/subcontinents
DELETE /v1/app-icons/:id/subcontinents/:subcontinentId
```

The old verb-in-path aliases `/create`, `/update/:id`, `/delete/:id` are **gone** —
compatibility shims that doubled the surface, called by nothing.

### PUT /v1/app-icons/:id/translations

```json
{ "translations": { "fr": "Technologies de l'information", "zh": "信息技术" } }
```

**Merges, never replaces** — editing French cannot silently drop Japanese.

Supplying text marks that locale **reviewed** and removes it from
`machineTranslated`; a person editing a label *is* the review. Pass
`"reviewed": false` to keep it flagged, which is what the seed does.

### Writable fields

Only these. Anything else in the body is ignored:

```
code slug name defaultName translationCode translations iconKey path menuLink
description level parentId type displayOrder isGlobal continents subcontinents
roles isActive
```

> **`deletedAt` is not writable.** It was accepted by the imported update schema,
> so a caller could resurrect a soft-deleted icon or tombstone one without the
> `isActive: false` that `remove()` pairs with it. Only `DELETE` sets it.

---

## Link and image rules

`menuLink` and `path` accept **two shapes and nothing else**:

| Shape | Example |
|---|---|
| Site-relative path | `/v2/finance/reports?item=total-tithe` |
| https on an RCCG host | `https://training.rccg.org/sso/login/…` |

Allowed hosts: `rccg.org`,  `rccgportal.org`, `rccgworld.org`, `trccg.org`,
`e-remittance-media.s3.amazonaws.com`, and their subdomains.

**The security property is the leading single slash**, not a character denylist.
A value starting `/` followed by a non-slash cannot be a scheme, so
`javascript:` and `data:` are unreachable, and `//evil.tld` is excluded. Query
strings, fragments and `:param` routes are therefore all fine — and all three
appear in the real data.

The host is matched between `://` and the first `/`, so
`https://evil.tld/?x=rccg.org` cannot pass by mentioning an allowed name, and the
suffix check is dot-delimited so `notrccg.org` does not either.

> This rule was rewritten after a dry run. The first version allowed only
> `[A-Za-z0-9-._~/]` and rejected **13 of the 14** working links in the seed
> CSV — nine query strings, one `:directoryId` route, four external apps —
> which would have made those menu items silently non-navigable.

`parentId` requires a **24-hex string**, not the repo's shared
`customJoi.objectId()`. That delegates to `Types.ObjectId.isValid`, which returns
**`true` for the number `126`** and then mints `0000007e71f6791f1b996dba` — a
reference to nothing. The CSV carries `parent_id: 126`, so this was a live path
to silent corruption.

---


### About the translations

They were written directly, not produced by a translation service — no
translation dependency exists in either package.json, and a directory service is
the wrong place to add a runtime one.

Proper nouns and acronyms are deliberately untranslated: `RPMS`, `CSR`, `HGS`,
`LGaF`, `Open Heavens`, `Go-A-Fishing`, `Vision 2032`, `Top 100`.

Domain vocabulary follows what the church means rather than the dictionary:
parish is `Gemeinde` / `paróquia` / `parokia` / `堂会`, with `教区` reserved for
province; `HF` is expanded to house fellowship, since the initials carry nothing
outside English.

**All 1,197 are flagged `machineTranslated`.** They have not been read by a native
speaker of any of the seven languages. `GET /v1/app-icons?unreviewed=true` is the
review queue; editing a label through `PUT /:id/translations` clears its flag.

---

## Error codes

| Status | When |
|---|---|
| `400` | validation — bad link, unknown locale, non-hex `parentId`, missing `translationCode` |
| `403` | not super-admin (or not authenticated, for `/me`) |
| `404` | no such icon |
| `409` | duplicate `code` or `slug` |

---

## Frontend guidance

1. **Read `label`** for the current language — the fallback chain is already
   applied server-side. `translations` is there for instant switching without a
   refetch.
2. **Do not fetch per language.** The default response carries all eight; only
   pass `?lang=` if you specifically want a smaller payload.
3. **`translationCode` is the stable id.** Pin anything that must survive a label
   being rewritten to it, not to `name`.
4. **Show `machineTranslated` in the admin UI.** A label in that list has not
   been read by a person.
5. **The legacy fields are still there** — `menu_link`, `parent_id`,
   `sort_order`, `active: 1|0`. New code should use the camelCase ones; the
   snake_case pair exists so the deployed bundle keeps working.
6. **The CSV fallback still applies** when the API is unreachable, and is now
   the offline path rather than the source of truth.
