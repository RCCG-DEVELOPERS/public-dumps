# Parish hierarchy — conflicts, and how to resolve them

Generated from the live directory on 2026-09-11 by `GET /v1/hq-assignments/integrity`. Read-only; producing it changed nothing.

`parishDirectory` holds 53,049 rows, of which 53,008 are places and 41 are departments.

## The six headquarters flags

One per geographic level. `parish` is **not** among them — it marks a row as a
real parish rather than a department, and takes no part in any headship.

| Level | Flag | Units | No HQ | More than one HQ |
|---|---|---:|---:|---:|
| continent | `chq` | 17 | 3 | 7 |
| sub-continent | `schq` | 18 | 3 | 6 |
| region | `rhq` | 124 | 2 | 37 |
| province | `phq` | 709 | 11 | 16 |
| zone | `zhq` | 7,918 | 43 | 19 |
| area | `ahq` | 17,616 | 65 | 45 |
| **total** | | | **127** | **130** |

| Other faults | Count |
|---|---:|
| Headquarters missing a flag beneath it | 629 across 15 combinations |
| Flag values no query can match | 3 rows |
| Parishes not marked `parish="1"` | 81 |
| Departments wrongly marked `parish="1"` | 41 |
| Codes living under two different parents | 340 |
| Duplicate `parishCode` | 1 |
| Indexes on any headquarters column | **0** |

---

## 1. Units with more than one headquarters — 130

Two parishes in one unit both carry that unit's flag. Exactly one may.

**Resolve each with:**

```http
POST /v1/hq-assignments/resolve-conflict
{ "flag": "<flag>", "unitCode": "<unit>", "keep": "<parishCode>",
  "clear": ["<every other parishCode listed below>"], "dryRun": true }
```

Every current holder must appear in `keep` or `clear`, or the call is refused.

### continent — 7 of 17 (`chq`)

**CNT01** — CONTINENT 1  ·  14,855 members, 2 claim `chq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `671966` | GRACELAND PARISH | CNTHQ01 | ZN0000002804 | AR0000002804 |
| `666250` | GRACE COURT | CNTHQ01 | UNCATEGORIZED | UNCATEGORIZED |

**CNT02** — CONTINENT 2  ·  6,544 members, 13 claim `chq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `731789` | RCCG CENTRAL PARISH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `781327` | RCCG SOLUTION POINT | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `965500` | VICTORY LAND PARISH  | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `732277` | DIVINE VICTORY | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `897249` | THE LORD'S ARK | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `601171` | RCCG LIVING WORD | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `548635` | PENIEL PLACE | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `739170` | THE CATALYST CHURCH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `612119` | FAITH HUB INTERNET CHURCH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `710870` | RCCG LIVING SEED, THE CHOSEN GENERATION | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `955480` | RCCG CHAPEL OF LIGHT | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `600444` | RCCG TABERNACLE OF MERCY | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `962161` | LIVING SEED CHURCH -THE EMERGING CENTRE | CNTHQ02 | ZN0000002508 | AR0000002508 |

**CNT03** — CONTINENT 3  ·  17,775 members, 3 claim `chq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `863259` | DOMINION CATHEDRAL | CNTHQ03 | ZN0000000009 | AR0000000009 |
| `617361` | CLOUD OF GLORY | CNTHQ03 | ZN0000000153 | AR0000000153 |
| `706150` | CHOSEN GENERATION | CNTHQ03 | ZN0000000009 | AR0000004765 |

**CNT04** — CONTINENT 4  ·  434 members, 2 claim `chq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `652177` | CONTINENT 4 TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071826 | AR0000071826 |
| `698110` | CONTINENT 4 SOUTHERN AFRICA TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071827 | AR0000071827 |

**CNT12** — CONTINENT 12  ·  9,249 members, 6 claim `chq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `568256` | GLORY HOUSE (COVENANT SANCTUARY) | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `927918` | PRECIOUS PEOPLE | CNTHQ12 | ZN0000008068 | AR0000008068 |
| `669907` | RCCG, GLORY PALACE | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `583402` | RCCG, GLORY CLOUD PARISH | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `936442` | RCCG, HOPE OF GLORY PARISH | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `579169` | RCCG LIGHT GENERATIONS, EXPRESSION CHURCH (CONT 3.ANNEX PARISH) | CNTHQ12 | UNCATEGORIZED | UNCATEGORIZED |

**CONTINENTEMERITUS** — CONTINENT EMERITUS  ·  2 members, 2 claim `chq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `774630` | ROYAL GATE | CONTINENTEMERITUS01 | ZN0000021355 | AR0000021355 |
| `761310` | DIVINE ASSEMBLY | CONTINENTEMERITUS01 | ZN0000028609 | AR0000028609 |

**EC0009** — Europe Continent 9  ·  1,182 members, 3 claim `chq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211603` | Breakthrough Sanctuary | ECPNLD0002 | ECZNLD0003 | ECANLD0007 |
| `211001` | Victory House, London | UKRGPR01 | ZN44400188 | AR444077351 |
| `213143` | JESUS CENTRE DUBLIN | RG0A | ZNIRE0000025 | ARIRE00000051 |

### sub-continent — 6 of 18 (`schq`)

**CNT01SUBCNT01** — CONTINENT 1 SUBCONTINENT 1  ·  14,857 members, 6 claim `schq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `671966` | GRACELAND PARISH | CNTHQ01 | ZN0000002804 | AR0000002804 |
| `666250` | GRACE COURT | CNTHQ01 | UNCATEGORIZED | UNCATEGORIZED |
| `881662` | JESUS GRACE PAVILLION | RG35 | ZN0000002639 | AR0000002639 |
| `882561` | JESUS THE LIGHT | RG35 | ZN0000002639 | AR0000002639 |
| `888549` | JESUS POWER ARENA | RG35 | ZN0000002639 | AR0000002639 |
| `882424` | JESUS SANCTUARY | RG35 | ZN0000002639 | AR0000002639 |

**CNT02SUBCNT01** — CONTINENT 2 SUBCONTINENT 1  ·  6,544 members, 13 claim `schq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `731789` | RCCG CENTRAL PARISH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `781327` | RCCG SOLUTION POINT | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `965500` | VICTORY LAND PARISH  | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `732277` | DIVINE VICTORY | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `897249` | THE LORD'S ARK | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `601171` | RCCG LIVING WORD | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `548635` | PENIEL PLACE | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `739170` | THE CATALYST CHURCH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `612119` | FAITH HUB INTERNET CHURCH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `710870` | RCCG LIVING SEED, THE CHOSEN GENERATION | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `955480` | RCCG CHAPEL OF LIGHT | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `600444` | RCCG TABERNACLE OF MERCY | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `962161` | LIVING SEED CHURCH -THE EMERGING CENTRE | CNTHQ02 | ZN0000002508 | AR0000002508 |

**CNT03SUBCNT01** — CONTINENT 3 SUBCONTINENT 1  ·  17,782 members, 3 claim `schq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `863259` | DOMINION CATHEDRAL | CNTHQ03 | ZN0000000009 | AR0000000009 |
| `617361` | CLOUD OF GLORY | CNTHQ03 | ZN0000000153 | AR0000000153 |
| `706150` | CHOSEN GENERATION | CNTHQ03 | ZN0000000009 | AR0000004765 |

**CNT04SUBCNT01** — CONTINENT 4 SUBCONTINENT 1  ·  434 members, 2 claim `schq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `652177` | CONTINENT 4 TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071826 | AR0000071826 |
| `698110` | CONTINENT 4 SOUTHERN AFRICA TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071827 | AR0000071827 |

**CNT12SUBCNT01** — CONTINENT 12 SUBCONTINENT 1  ·  9,251 members, 6 claim `schq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `568256` | GLORY HOUSE (COVENANT SANCTUARY) | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `927918` | PRECIOUS PEOPLE | CNTHQ12 | ZN0000008068 | AR0000008068 |
| `669907` | RCCG, GLORY PALACE | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `583402` | RCCG, GLORY CLOUD PARISH | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `936442` | RCCG, HOPE OF GLORY PARISH | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `579169` | RCCG LIGHT GENERATIONS, EXPRESSION CHURCH (CONT 3.ANNEX PARISH) | CNTHQ12 | UNCATEGORIZED | UNCATEGORIZED |

**EMERITUSSUBCNT01** — CONTINENT EMERITUS SUBCONTINENT  ·  2 members, 2 claim `schq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `774630` | ROYAL GATE | CONTINENTEMERITUS01 | ZN0000021355 | AR0000021355 |
| `761310` | DIVINE ASSEMBLY | CONTINENTEMERITUS01 | ZN0000028609 | AR0000028609 |

### region — 37 of 124 (`rhq`)

**CNT01** — CONTINENT 1  ·  2 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `671966` | GRACELAND PARISH | CNTHQ01 | ZN0000002804 | AR0000002804 |
| `666250` | GRACE COURT | CNTHQ01 | UNCATEGORIZED | UNCATEGORIZED |

**CNT02** — CONTINENT 2  ·  13 members, 13 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `731789` | RCCG CENTRAL PARISH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `781327` | RCCG SOLUTION POINT | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `965500` | VICTORY LAND PARISH  | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `732277` | DIVINE VICTORY | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `897249` | THE LORD'S ARK | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `601171` | RCCG LIVING WORD | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `548635` | PENIEL PLACE | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `739170` | THE CATALYST CHURCH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `612119` | FAITH HUB INTERNET CHURCH | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `710870` | RCCG LIVING SEED, THE CHOSEN GENERATION | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `955480` | RCCG CHAPEL OF LIGHT | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `600444` | RCCG TABERNACLE OF MERCY | CNTHQ02 | ZN0000002508 | AR0000002508 |
| `962161` | LIVING SEED CHURCH -THE EMERGING CENTRE | CNTHQ02 | ZN0000002508 | AR0000002508 |

**CNT03** — CONTINENT 3  ·  3 members, 3 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `863259` | DOMINION CATHEDRAL | CNTHQ03 | ZN0000000009 | AR0000000009 |
| `617361` | CLOUD OF GLORY | CNTHQ03 | ZN0000000153 | AR0000000153 |
| `706150` | CHOSEN GENERATION | CNTHQ03 | ZN0000000009 | AR0000004765 |

**CNT04** — CONTINENT 4  ·  2 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `652177` | CONTINENT 4 TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071826 | AR0000071826 |
| `698110` | CONTINENT 4 SOUTHERN AFRICA TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071827 | AR0000071827 |

**CNT12** — CONTINENT 12  ·  6 members, 6 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `568256` | GLORY HOUSE (COVENANT SANCTUARY) | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `927918` | PRECIOUS PEOPLE | CNTHQ12 | ZN0000008068 | AR0000008068 |
| `669907` | RCCG, GLORY PALACE | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `583402` | RCCG, GLORY CLOUD PARISH | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `936442` | RCCG, HOPE OF GLORY PARISH | CNTHQ12 | ZN0000002815 | AR0000002815 |
| `579169` | RCCG LIGHT GENERATIONS, EXPRESSION CHURCH (CONT 3.ANNEX PARISH) | CNTHQ12 | UNCATEGORIZED | UNCATEGORIZED |

**CONTINENTEMERITUSR01** — CONTINENT EMERITUS REGION  ·  2 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `774630` | ROYAL GATE | CONTINENTEMERITUS01 | ZN0000021355 | AR0000021355 |
| `761310` | DIVINE ASSEMBLY | CONTINENTEMERITUS01 | ZN0000028609 | AR0000028609 |

**EAREG01** — EAST AFRICA REGION 1  ·  99 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `937764` | EAST AFRICA TEMP HQ PARISH (MUST BE CHANGED) | EA01 | ZN0000071828 | AR0000071828 |
| `534329` | JESUS HOUSE | RGEA01 | UNCATEGORIZED | AR0000069430 |

**R01** — REGION 1  ·  1,026 members, 4 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `721693` | THRONE OF GRACE | RG01 | ZN0000051237 | AR0000051237 |
| `599781` | RCCG GRACE HUB | RG01 | ZN0000051237 | AR0000051237 |
| `717721` | SANCTUARY OF GRACE | RG01 | ZN0000051237 | AR0000051237 |
| `213146` | JESUS TABERNACLE DUBLIN | R01 | ZNIRE0000006 | ARIRE00000011 |

**R02** — REGION 2  ·  766 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `706853` | THE MASTER'S PLACE | RG02 | ZN0000002609 | AR0000002609 |
| `213174` | OPEN HEAVEN DUBLIN | R02 | ZNIRE0000011 | ARIRE00000020 |

**R04** — REGION 4  ·  802 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `938456` | THE MASTERS COURT | RG04 | ZN0000038905 | AR0000038905 |
| `545255` | MOUNT OF JOY | RG04 | ZN0000038905 | AR0000038905 |

**R05** — REGION 5  ·  1,060 members, 6 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `754212` | REGIONAL HQTRS PARISH CHAPEL OF BLESSING | RG05 | ZN0000002803 | AR0000002803 |
| `968750` | REVIVAL CENTRE | RG05 | ZN0000002803 | AR0000002803 |
| `788639` | CHAPEL OF GLORY | RG05 | ZN0000002803 | AR0000002803 |
| `605987` | CHAPEL OF GLADNESS | RG05 | ZN0000002803 | AR0000002803 |
| `923820` | LIFE WAY PARISH | RG05 | ZN0000002803 | AR0000002803 |
| `214049` | Jesus Household Milan | P10 | ZNMLD0000006 | ARMLD0000008 |

**R06** — REGION 6  ·  910 members, 3 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `993046` | JESUS EMBASSY  REGION 6 HEADQUARTER PARISH | RG06 | ZN0000029562 | AR0000029562 |
| `375139` | HIS GRACE  | P02 | ZN0000941462 | AR0000989205 |
| `214000` | New Life Assembly,  Antwerpen | P01 | ZNMLD0000001 | ARMLD0000001 |

**R07** — REGION 7  ·  476 members, 10 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `973863` | VICTORY ASSEMBLY | RG07 | ZN0000002805 | AR0000002805 |
| `755739` | TRIUMPHANT ASSEMBLY | RG07 | ZN0000002805 | AR0000002805 |
| `824033` | GLORY OF GOD TABERNACLE | RG07 | ZN0000002805 | AR0000002805 |
| `847042` | OPEN HEAVENS | RG07 | ZN0000002805 | AR0000002805 |
| `601732` | COMPLETE RESTORATION | RG07 | ZN0000002805 | AR0000002805 |
| `968672` | FAITH TABERNACLE | RG07 | ZN0000002805 | AR0000002805 |
| `791833` | PERFECT PEACE | RG07 | ZN0000002805 | AR0000002805 |
| `743679` | RESTORATION PARISH | RG07 | ZN0000002805 | AR0000002805 |
| `976187` | VICTORY TABERNACLE | RG07 | ZN0000002805 | AR0000002805 |
| `937925` | MOUNT ZION | RG07 | ZN0000002805 | AR0000002805 |

**R09** — REGION 9  ·  184 members, 3 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `843711` | ROYAL PALACE PARISH | RG09 | ZN0000002807 | AR0000002807 |
| `879498` | ROYAL DIADEM  | RG09 | ZN0000002807 | AR0000002807 |
| `768744` | ROYAL CHAMPIONS | RG09 | ZN0000002807 | AR0000002807 |

**R11** — REGION 11  ·  712 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `622504` | RCCG RESURRECTION PARISH | RG11 | ZN0000049472 | AR0000049472 |
| `989459` | RESURRECTION PARISH ANNEX 4 | RG11 | ZN0000049472 | AR0000049472 |

**R14** — REGION 14  ·  583 members, 6 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `673829` | SPIRIT AND LIFE PARISH | RG14 | ZN0000019386 | AR0000019386 |
| `695980` | ABUNDANCE MEGA | RG14 | ZN0000022134 | AR0000022134 |
| `790925` | NEW BEGINNING PARISH | RG14 | ZN0000022134 | AR0000069320 |
| `581509` | VICTORY CHAPEL | RG14 | ZN0000022134 | AR0000069320 |
| `886577` | POWER ARENA | RG14 | ZN0000022134 | AR0000069320 |
| `978131` | LEGACY PARISH | RG14 | ZN0000022134 | AR0000069320 |

**R15** — REGION 15  ·  684 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `726314` | REPOSITIONING CENTER | RG15 | ZN0000015330 | AR0000015330 |
| `775488` | HOUSE OF GLORY | RG15 | ZN0000015330 | AR0000015330 |

**R16** — REGION 16  ·  165 members, 14 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `536231` | REDEEMERS HOUSE | RG16 | ZN0000002811 | AR0000002811 |
| `819365` | HIGH FLYERS (EAGLE'S WING) | RG16 | ZN0000002811 | AR0000064786 |
| `651083` | HOUSE OF GLORY | RG16 | ZN0000002811 | AR0000002811 |
| `951668` | ALHERI PARISH | RG16 | ZN0000002811 | AR0000002811 |
| `571719` | FADAR YABO | RG16 | ZN0000002811 | AR0000002811 |
| `577100` | AMG FELLOWSHIP | RG16 | ZN0000002811 | AR0000064786 |
| `710658` | CELEBRATION HOUSE | RG16 | ZN0000002811 | AR0000064786 |
| `595779` | JUBILEE HOUSE | RG16 | ZN0000002811 | AR0000064786 |
| `760315` | CHERITH PARISH | RG16 | ZN0000002811 | AR0000064786 |
| `758022` | PATHFINDERS | RG16 | ZN0000002811 | AR0000002811 |
| `654614` | VESSELS OF GOLD | RG16 | ZN0000002811 | AR0000002811 |
| `627450` | FLOOD GATES OF HEAVEN | RG16 | ZN0000002811 | AR0000002811 |
| `580735` | PROSPERITY | RG16 | ZN0000002811 | AR0000002811 |
| `782664` | RCCG GLAD TIDINGS | RG16 | ZN0000002811 | AR0000002811 |

**R17** — REGION 17  ·  237 members, 7 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `893444` | ALPHA AND OMEGA PARISH | RG17 | ZN0000002812 | AR0000002812 |
| `902038` | CHRIST CHURCH | RG17 | ZN0000002812 | AR0000002812 |
| `609681` | RESURRECTION POWER SANCTUARY | RG17 | ZN0000002812 | AR0000002812 |
| `864868` | RCCG  GINDAN YABO (HOUSE OF WORSHIP) | RG17 | ZN0000002812 | AR0000002812 |
| `627932` | RCCG  GINDAN JINKAI (HOUSE OF MERCY) | RG17 | ZN0000002812 | AR0000002812 |
| `638588` | RCCG  GINDAN YANCI (HOUSE OF FREEDOM) | RG17 | ZN0000002812 | AR0000002812 |
| `804515` | RCCG  NAZARA (HOUSE OF VICTORY) | RG17 | ZN0000002812 | AR0000002812 |

**R20** — REGION 20  ·  661 members, 4 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `907359` | REVELATION PARISH | RG20 | ZN0000059330 | AR0000059330 |
| `708761` | R.C.C.G TOWER OF DAVID | RG20 | ZN0000047330 | AR0000047330 |
| `852513` | CITY OF DAVID SANCTUARY | RG20 | ZN0000048581 | AR0000048581 |
| `692669` | GRACE ARENA | RG20 | ZN0000059330 | AR0000059330 |

**R23** — REGION 23  ·  1,305 members, 3 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `966721` | BEAUTIFUL GATE MEGA PARISH | RG23 | ZN0000002817 | AR0000002817 |
| `733191` | TABERNACLE OF GLORY | RG23 | ZN0000002817 | AR0000002817 |
| `626834` | RCCG, SEAT OF MERCY | RG23 | ZN0000002817 | AR0000002817 |

**R26** — REGION 26  ·  664 members, 3 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `649501` | REGIONAL HQTS  (THE OVERCOMERS PLACE) | RG26 | ZN0000005990 | AR0000005990 |
| `844719` | RCCG THE GLORY HOUSE | RG26 | ZN0000005990 | AR0000005990 |
| `836874` | RCCG GATE OF HEAVEN PARISH | RG26 | ZN0000005990 | AR0000005990 |

**R29** — REGION 29  ·  479 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `565049` | FLOURISHING BRANCH | RG29 | ZN0000009728 | AR0000009728 |
| `650251` | MARANATHA CHAPEL | RG29 | ZN0000009728 | AR0000009728 |

**R31** — REGION 31  ·  821 members, 5 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `883665` | Overcomers Assembly | RG31 | N/A | N/A |
| `763702` | OVERCOMERS PARISH | RG31 | ZN0000000288 | AR0000000288 |
| `674016` | HOUSE OF PRAYER | RG31 | ZN0000023300 | AR0000023300 |
| `916277` | RCCG OVERCOMERS SANCTUARY | RG31 | ZN0000000288 | AR0000000288 |
| `993797` | PALACE OF FAVOUR | RG31 | ZN0000023300 | AR0000023300 |

**R34** — REGION 34  ·  880 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `922320` | UNITY MODEL PARISH | RG34 | ZN0000014820 | AR0000014820 |
| `971411` | ROSE OF SHARON MEGA | RG34 | ZN0000057145 | AR0000057145 |

**R35** — REGION 35  ·  837 members, 8 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `637808` | JESUS HOUSE | RG35 | ZN0000002639 | AR0000002639 |
| `670430` | HIGHFLYERS ASSEMBLY | RG35 | ZN0000002639 | AR0000002639 |
| `938202` | RCCG  SALVATION ARENA | RG35 | ZN0000002639 | AR0000002639 |
| `615095` | JESUS EMBASSY | RG35 | ZN0000002639 | AR0000002639 |
| `917105` | JESUS GLORIOUS ASSEMBLY | RG35 | ZN0000002639 | AR0000002639 |
| `743409` | RCCG JESUS WAY | RG35 | ZN0000002639 | AR0000002639 |
| `587533` | JESUS VINEYARD PARISH | RG35 | ZN0000002639 | AR0000002639 |
| `860386` | RCCG REDEEMERS UNIVERSITY CHAPEL OF POWER | RG35 | ZN0000071185 | AR0000071185 |

**R36** — REGION 36  ·  1,030 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `932539` | DESTINY SANCTUARY | RG36 | ZN0000025714 | AR0000025714 |
| `954853` | SHABACH HILL | RG36 | ZN0000025714 | AR0000025714 |

**R37** — REGION 37  ·  1,115 members, 3 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `682660` | PALACE OF GLORY | RG37 | ZN0000046993 | AR0000046993 |
| `955176` | RCCG PALACE OF HOPE | RG37 | ZN0000046993 | AR0000046993 |
| `889585` | PRAISE PALACE | RG37 | ZN0000046993 | AR0000046993 |

**R46** — REGION 46  ·  804 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `606013` | SURE MERCIES OF DAVID | RG46 | ZN0000009699 | AR0000009699 |
| `980901` | GREEN PASTURES PARISH | RG46 | ZN0000013402 | AR0000013402 |

**R55** — REGION 55  ·  968 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `822143` | HOUSE OF GRACE (REGIONAL  HEADQUARTERS) | RG55 | ZN0000050266 | AR0000050266 |
| `763794` | HOUSE OF HOPE | RG55 | ZN0000050266 | AR0000050266 |

**R58** — REGION 58  ·  683 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `737793` | SHILOH ARENA | RG58 | ZN0000029811 | AR0000029811 |
| `998932` | RCCG THE SPRING | RG58 | ZN0000029811 | AR0000029811 |

**R63** — REGION 63  ·  573 members, 3 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `999341` | RCCG GPA REGIONAL HQTRS | RG63 | ZN0000002522 | AR0000002522 |
| `578595` | GOD'S PASTURES PARISH | RG63 | ZN0000002522 | AR0000002522 |
| `766606` | GOOD SHEPHERD PASTURE - GSP | RG63 | ZN0000002522 | AR0000002522 |

**R65** — REGION 65  ·  782 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `737258` | FCT PROVINCE 18 HQRTS  (JESUS EMBASSY) | RG65 | ZN0000028949 | AR0000028949 |
| `907036` | FAITH EMBASSY | RG65 | ZN0000028949 | AR0000028949 |

**RCITYREG01** — REDEMPTION CITY REGION REGION  ·  83 members, 4 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `892861` | HOUSE OF FAVOUR | REDEMPTIONCITYRG01 | ZN0000048588 | AR0000048588 |
| `569878` | RCCG HOUSE OF VICTORY PARISH | REDEMPTIONCITYRG01 | ZN0000048588 | AR0000048588 |
| `713807` | DIVINE FAVOUR PARISH | REDEMPTIONCITYRG01 | ZN0000048588 | AR0000048588 |
| `879554` | HOUSE OF PEACE | REDEMPTIONCITYRG01 | ZN0000048588 | AR0000048588 |

**RWC01** — WEST COAST REGION 1  ·  132 members, 7 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `614309` | RCCG WORD ALIVE | RGWC01 | ZN0000066967 | AR0000066967 |
| `790776` | GLORY ASSEMBLY | RGWC01 | ZN0000066967 | AR0000066967 |
| `962933` | EAGLES COURFT | RGWC01 | ZN0000066967 | AR0000066967 |
| `553232` | HOPE OF GLORY ASSEMBLY | RGWC01 | ZN0000066967 | AR0000066967 |
| `881108` | HOUSE OF DOMINION | RGWC01 | ZN0000066967 | AR0000066967 |
| `886406` | BREAKTHROUGH SANCTUARY | RGWC01 | ZN0000066964 | AR0000066964 |
| `855539` | FATHERS HOUSE | RGWC01 | ZN0000066967 | AR0000066967 |

**RWC03** — WEST COAST REGION 3  ·  104 members, 6 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `944549` | MERCY SEAT CATHEDRAL | RGWC03 | ZN0000070364 | AR0000070364 |
| `907523` | MERCY TABERNACLE | RGWC03 | ZN0000070364 | AR0000072192 |
| `681900` | HOUSE OF GLORY | RGWC03 | ZN0000070364 | AR0000070364 |
| `573675` | LIFE GATE ASSEMBLY | RGWC03 | ZN0000070364 | AR0000072192 |
| `579288` | ISAAC GENERATION | RGWC03 | ZN0000070364 | AR0000070364 |
| `762618` | RESTORATION HOUSE | RGWC03 | ZN0000070364 | AR0000070364 |

**RWC04** — WEST COAST REGION 4  ·  288 members, 2 claim `rhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `638770` | RCCG SALVATION CENTER | RGWC04 | ZN0000067270 | AR0000067270 |
| `745909` | ROYAL ASSEMBLY PARISH  | RGWC04 | ZN0000067270 | AR0000067270 |

### province — 16 of 709 (`phq`)

**CNTHQ04** — CONTINENT 4 HQ  ·  2 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `652177` | CONTINENT 4 TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071826 | AR0000071826 |
| `698110` | CONTINENT 4 SOUTHERN AFRICA TEMP HQ PARISH (MUST BE CHANGED) | CNTHQ04 | ZN0000071827 | AR0000071827 |

**DE20** — DELTA PROVINCE 20  ·  81 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `740910` | CHAMPIONS CATHEDRAL | DE20 | ZN0000012249 | AR0000012249 |
| `817394` | DLT20 HQRTS | DE20 | ZN0000065557 | AR0000065557 |

**DE24** — DELTA PROVINCE 24  ·  137 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `799192` | CORNERSTONE MEGA PARISH (KWALE) | DE24 | ZN0000013213 | AR0000013213 |
| `610706` | AMAZING GRACE | DE24 | ZN0000015612 | AR0000060312 |

**ECPAUT0001** — AUT Province 1  ·  20 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211562` | RCCG Hosanna Zion Plovdiv | ECPAUT0001 | ECZBGR0001 | ECABGR0001 |
| `211549` | Chapel of His Glory | ECPAUT0001 | ECZAUT0001 | ECAAUT0001 |

**ICTHQ1** — ICT HQ  ·  61 members, 3 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `888890` | NO NAME PARISH | ICTHQ1 | ZN3050092737 | AR46601659006 |
| `373745` | Glory | ICTHQ1 | ZN133979 | AR1000002815 |
| `632101` | RCCG GLOBAL PARISH | ICTHQ1 | ZN100012691 | AR1000002815 |

**KY01** — KENYA PROVINCE 1  ·  44 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `805470` | MARANATHA INTER. CHRISTIAN CENTRE (ZONE) | KY01 | ZN0000069433 | AR0000069433 |
| `970688` | POTTERS HOUSE (ZONE) | KY01 | ZN0000069432 | AR0000069432 |

**LA114** — LAGOS PROVINCE 114  ·  95 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `734862` | THE    LORD   FAVOUR   PARISH | LA114 | ZN0000027531 | AR0000006673 |
| `726068` | BEAUTIFUL GATE | LA114 | ZN0000014713 | AR0000014713 |

**ME01** — MIDDLE EAST PROVINCE 1  ·  18 members, 3 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `710119` | GARDEN OF PEACE | ME01 | ZN0000066396 | AR0000066396 |
| `995289` | REHBOTH ASSEMBLY | ME01 | ZN0000066397 | AR0000066397 |
| `874747` | VICTORY HOUSE | ME01 | ZN0000066398 | AR0000066398 |

**ME02** — MIDDLE EAST PROVINCE 2  ·  20 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `921392` | POWER AND PRAISE CHAPEL | ME02 | ZN0000066399 | AR0000066399 |
| `652030` | HOUSE OF TRUE VINE | ME02 | ZN0000066399 | AR0000066399 |

**ME03** — MIDDLE EAST PROVINCE 3  ·  13 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `797384` | GRACE SANCTUARY PARISH | ME03 | ZN0000066401 | AR0000066401 |
| `897278` | TRINITY PALACE AJMAN | ME03 | ZN0000066411 | AR0000066411 |

**P01** — PROVINCE 1  ·  41 members, 3 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `213141` | INSPIRATION HOUSE CORK | P01 | ZNIRE0000001 | ARIRE0000001 |
| `214050` | House of Victory Novara | P01 | ZNMLD0000007 | ARMLD0000009 |
| `214000` | New Life Assembly,  Antwerpen | P01 | ZNMLD0000001 | ARMLD0000001 |

**P02** — PROVINCE 2  ·  38 members, 5 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `213150` | KING'S COURT PARISH DAMASTOWN DUBLIN | P02 | ZNIRE0000012 | ARIRE00000021 |
| `371665` | New Life Assembly  | P02 | ZN0000241469 | AR0000979204 |
| `214017` | House of Peace, Belgrade | P02 | ZNMLD0000003 | ARMLD0000005 |
| `214055` | Mercy Court Trento | P02 | ZNMLD0000008 | ARMLD0000011 |
| `375139` | HIS GRACE  | P02 | ZN0000941462 | AR0000989205 |

**P03** — PROVINCE 3  ·  33 members, 3 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `213172` | OAISIS ARENA WATERFORD  | P03 | ZNIRE0000007 | ARIRE00000012 |
| `214030` | Six Wings Ministry | P03 | ZNMLD0000004 | ARMLD0000006 |
| `214059` | Bread of Life Ravenna | P03 | ZNMLD0000010 | ARMLD0000012 |

**P04** — PROVINCE 4  ·  34 members, 3 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `213165` | MIRACLE LAND DUNDALK | P04 | ZNIRE0000017 | ARIRE00000035 |
| `214036` | Peace House | P04 | ZNMLD0000005 | ARMLD0000007 |
| `214064` | City of Victory Roma | P04 | ZNMLD0000012 | ARMLD0000014 |

**P05** — PROVINCE 5  ·  17 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `213110` | CHAPEL OF LIGHT LONGFORD | P05 | ZNIRE0000022 | ARIRE00000043 |
| `214053` | House of Prayer Mantova | P05 | ZNMLD0000013 | ARMLD0000016 |

**RGWC01** — WEST COAST REGION 1 HQTRS  ·  7 members, 2 claim `phq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `553232` | HOPE OF GLORY ASSEMBLY | RGWC01 | ZN0000066967 | AR0000066967 |
| `886406` | BREAKTHROUGH SANCTUARY | RGWC01 | ZN0000066964 | AR0000066964 |

### zone — 19 of 7,918 (`zhq`)

**ECZNLD0001** — ZONE 1  ·  6 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211591` | Embassy of God | ECPNLD0001 | ECZNLD0001 | ECANLD0001 |
| `211592` | Amazing Grace Parish | ECPNLD0001 | ECZNLD0001 |  ECANLD0001 |

**ECZNLD0002** — ZONE 2  ·  6 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211600` | Amazing Grace Sanctuary | ECPNLD0001 | ECZNLD0002 | ECANLD0004 |
| `211598` | Amazing Grace Centre  | ECPNLD0001 | ECZNLD0002 | ECANLD0003 |

**ECZNLD0003** — ZONE 1  ·  7 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211603` | Breakthrough Sanctuary | ECPNLD0002 | ECZNLD0003 | ECANLD0007 |
| `211605` | Tabanacle of David  | ECPNLD0002 | ECZNLD0003 | ECANLD0006 |

**ZN0000002015** — SHALOM CHAPEL  ·  11 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `743532` | SHALOM CHAPEL | LA93 | ZN0000002015 | AR0000002015 |
| `761911` | ALL SUFFICIENT GOD PARISHÂ | LA93 | ZN0000002015 | AR0000059831 |

**ZN0000009159** — FULLNESS OF JOY  ·  9 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `981226` | LIVING TREASURE | KO07 | ZN0000009159 | AR34727417784 |
| `548619` | FULFILLMENT PARISH | KO07 | ZN0000009159 | AR0000009159 |

**ZN0000011617** — TOWER OF PRAISE  ·  4 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `864995` | TOWER OF PRAISE | OY11 | ZN0000011617 | AR0000011617 |
| `933197` | GRACE TARBERNACLE PARISH | OY11 | ZN0000011617 | AR0000056009 |

**ZN0000015599** — LIVING BREAD  ·  14 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `971726` | COMFORTER | DE14 | ZN0000015599 | AR0000023248 |
| `964778` | LIVING BREAD | DE14 | ZN0000015599 | AR0000015599 |

**ZN0000024139** — BREAKTHROUGH  ·  11 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `867953` | OPEN DOOR | LA52 | ZN0000024139 | AR0000024827 |
| `967939` | BREAKTHROUGH | LA52 | ZN0000024139 | AR0000024139 |

**ZN0000025199** — DAYSPRING (ZONE 5)  ·  11 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `832314` | LIVING WATERS PARISH | LA114 | ZN0000025199 | AR0000059704 |
| `607632` | DAYSPRING PARISH | LA114 | ZN0000025199 | AR0000025199 |

**ZN0000027332** — WATERED GARDEN  ·  17 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `843718` | GOSHEN MODEL PARISH | DE14 | ZN0000027332 | AR0000015486 |
| `602758` | WATERED GARDEN | DE14 | ZN0000027332 | AR0000027332 |

**ZN0000030966** — TABERNACLE OF FAITH (ZONE 11)  ·  11 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `672245` | TABERNACLE OF FAITH | LA113 | ZN0000030966 | AR0000030966 |
| `877524` | DAYSPRING MODEL PARISH | LA113 | ZN0000030966 | AR0000030962 |

**ZN0000032362** — NEW WINE  ·  10 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `872659` | NEW WINE PARISH | OY21 | ZN0000032362 | AR0000032362 |
| `769275` | KING OF GLORY | OY21 | ZN0000032362 | AR0000032673 |

**ZN0000046697** — LATTER HOUSE  ·  7 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `959731` | LATTER HOUSE | OG29 | ZN0000046697 | AR0000046697 |
| `696410` | MARVELOUS GRACE | OG29 | ZN0000046697 | AR0000047027 |

**ZN0000063970** — Bethel (Zone 10)  ·  9 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `930677` | HOUSE OF VICTORY | LA45 | ZN0000063970 | AR0000033490 |
| `997488` | Bethel Parish | LA45 | ZN0000063970 | AR0000063970 |

**ZN205085** — RCCG NEW LIFE POWER ASSEMBLY ZONE  ·  5 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `377868` | RCCG NEW LIFE POWER ASSEMBLY ZONE | FC12 | ZN205085 | AR0000051636 |
| `376603` | RCCG NEW LIFE POWER ASSEMBLY | FC12 | ZN205085 | AR61591231 |

**ZN44400029** — Covenant Restoration Assembly, Perry Barr  ·  10 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211151` | Covenant Restoration Assembly, Selly Oak | UKR02PR01 | ZN44400029 | AR444077062 |
| `211195` | Covenant Restoration Assembly, Perry Barr | UKR02PR01 | ZN44400029 | AR444077061 |

**ZN44400118** — Living Water Parish, Stoke-on-Trent  ·  2 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211829` | Living Water Parish, Stoke-on-Trent | UKRGPR11 | ZN44400118 | AR444077212 |
| `211897` | Living Water Parish, Stoke-on-Trent | UKR11PR01 | ZN44400118 | AR444077212 |

**ZN7598660480** — ABUNDANT LIFE (AREA 18)  ·  3 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `977878` | FAVOUR (DISTINCT AREA 5) | LA79 | ZN7598660480 | AR0000044898 |
| `565886` | ABUNDANT LIFE (UNIQUE ZONE 4, DISTINCT AREA 4) | LA79 | ZN7598660480 | AR0000036320 |

**ZN807549** — LIVING HOPE SANCTUARY  ·  2 members, 2 claim `zhq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `955848` | LIVING HOPE SANCTUARY | LA129 | ZN807549 | AR0000066065 |
| `376647` | LIVING HOPE SANCTUARY | LA129 | ZN807549 | AR0000024861 |

### area — 45 of 17,616 (`ahq`)

**AR0000005593** — MOUNT ZION  ·  7 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `699715` | MOUNT ZION PARISH | LA54 | ZN100023833 | AR0000005593 |
| `683102` | TREE OF MERCY | LA54 | ZN100023833 | AR0000005593 |

**AR0000009155** — GREAT JOY ASSEMBLY (AREA 63)  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `836885` | GREAT JOY ASSEMBLY | OY08 | ZN0000009155 | AR0000009155 |
| `535388` | VICTORY PARISH | OY08 | ZN0000009155 | AR0000009155 |

**AR0000009919** — UNITY SANTUARY  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `553079` | UNITY SANTUARY | OY06 | ZN0000015952 | AR0000009919 |
| `813341` | CHAPEL OF BREAKTHROUGH | OY06 | ZN0000015952 | AR0000009919 |

**AR0000012583** — OVERFLOW  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `778522` | OVERFLOW | OY08 | ZN0000012583 | AR0000012583 |
| `379303` | ANOINTED PARISH | OY08 | ZN0000012583 | AR0000012583 |

**AR0000018541** — THRONE OF FAVOUR (AREA 23)  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `655718` | ARISE AND SHINE PARISH | LA75 | ZN0000038615 | AR0000018541 |
| `625516` | THRONE OF FAVOUR | LA75 | ZN0000038615 | AR0000018541 |

**AR0000020582** — COVENANT OF LIFE (AREA 27)  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `948152` | GRACE MODEL | ED11 | ZN0000020582 | AR0000020582 |
| `922575` | COVENANT OF LIFE | ED11 | ZN0000020582 | AR0000020582 |

**AR0000028072** — REDEMPTION LIGHT (AREA 16)  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `772394` | REDEMPTION LIGHT | ED14 | ZN0000028072 | AR0000028072 |
| `939671` | GREAT SHEPHERD | ED14 | ZN0000048455 | AR0000028072 |

**AR0000038395** — NEXT LEVEL  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `662126` | SANCTUARY OF HOPE PARISH | LA75 | ZN100095429 | AR0000038395 |
| `660861` | NEXT LEVEL | LA75 | ZN0000037091 | AR0000038395 |

**AR0000044200** — LIVING WATER  ·  6 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `794308` | PALACE OF FIRE AREA | NA04 | ZN0000044174 | AR0000044200 |
| `729234` | LIVING WATER | NA04 | ZN0000044174 | AR0000044200 |

**AR0000047680** — POTTERS PLACE (AREA 002)  ·  5 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `847668` | POTTERS PLACE | LA46 | ZN7821160022 | AR0000047680 |
| `801773` | PRAISE SANCTUARY | LA46 | ZN7821160022 | AR0000047680 |

**AR0000048045** — WISDOM GATE (AREA 007)  ·  5 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `930907` | WISDOM GATE | LA46 | ZN0000047745 | AR0000048045 |
| `751237` | HIGH PRAISES ASSEMBLY | LA46 | ZN0000047745 | AR0000048045 |

**AR0000048187** — TRINITY CHAPEL (AREA 018)  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `968523` | WIINNERS PARISH | LA46 | ZN0000048095 | AR0000048187 |
| `594328` | TRINITY CHAPEL | LA46 | ZN0000048095 | AR0000048187 |

**AR0000051338** — PRINCE OF PEACE (AREA 25)  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `718216` | PRINCE OF PEACE | LA23 | ZN0000051338 | AR0000051338 |
| `927166` | SOLUTION CENTER | LA23 | ZN0000051338 | AR0000051338 |

**AR0000059831** — THRONE OF MERCY  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `737230` | THRONE OF MERCY | LA93 | ZN0000002015 | AR0000059831 |
| `761911` | ALL SUFFICIENT GOD PARISHÂ | LA93 | ZN0000002015 | AR0000059831 |

**AR0000061271** — FOUNTAIN OF JOY  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `895151` | FOUNTAIN OF JOY | DE22 | ZN0000061271 | AR0000061271 |
| `548937` | THE CONSUMING FIRE | DE22 | ZN0000061271 | AR0000061271 |

**AR0000062133** — RCCG RESURRECTION  ANNEX 2 (AREA 14)  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `680279` | RCCG RESURRECTION PARISH ANNEX 2 | LA114 | ZN0000025196 | AR0000062133 |
| `909510` | RCCG RESURRECTION PARISH ANNEX 3 | LA114 | ZN0000025196 | AR0000062133 |

**AR0000979204** — ICT NEW AREA 1  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `378291` | THE NEW | P02 | ZN0000541449 | AR0000979204 |
| `371665` | New Life Assembly  | P02 | ZN0000241469 | AR0000979204 |

**AR16733444228** — HIS COMPASSION  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `849031` | SANCTUARY OF JOY | FC12 | ZN0000013959 | AR16733444228 |
| `761660` | HIS COMPASSION | FC12 | ZN0000013959 | AR16733444228 |

**AR371496** — Area 1  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `370932` |  Higher | PR372447 | ZN373616 | AR371496 |
| `371337` |  Life Assembly  | PR372447 | ZN373616 | AR371496 |

**AR444077043** — City of God, Glasgow  ·  8 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211380` | Light House Parish, Clydebank | UKR04PR02 | ZN44400046 | AR444077043 |
| `211376` | City of God, Glasgow | UKR04PR02 | ZN44400046 | AR444077043 |

**AR444077176** — Jesus House, Torry  ·  8 members, 4 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211414` | Jesus House, Torry | UKR04PR01 | ZN44400096 | AR444077176 |
| `212096` | Jesus is Lord Assembly, Gillingham | UKR07PR02 | ZN44400061 | AR444077176 |
| `211768` | Jesus House, London | UKRGPR10 | ZN44400094 | AR444077176 |
| `211827` | Jesus House, London | UKRGPR11 | ZN44400095 | AR444077176 |

**AR444077209** — Living Praise, Sunderland  ·  5 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211120` | Living Praise, Sunderland | UKR01PR04 | ZN44400024 | AR444077209 |
| `211119` | Covenant of Peace, Sidcup | UKR01PR04 | ZN44400024 | AR444077209 |

**AR444077211** — Living Water Parish, South Wimbledon  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `212053` | Living Water Parish, South Wimbledon | UKRGPR07 | ZN44400117 | AR444077211 |
| `211860` | Living Water Parish, South Wimbledon | UKR11PR02 | ZN44400116 | AR444077211 |

**AR444077212** — Living Water Parish, Stoke-on-Trent  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211829` | Living Water Parish, Stoke-on-Trent | UKRGPR11 | ZN44400118 | AR444077212 |
| `211897` | Living Water Parish, Stoke-on-Trent | UKR11PR01 | ZN44400118 | AR444077212 |

**AR444077229** — New Hope Sanctuary, Eltham  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211313` | New Hope Sanctuary, Eltham | UKR03PR02 | ZN44400191 | AR444077229 |
| `211314` | House of Praise, Kidbrooke | UKR03PR02 | ZN44400191 | AR444077229 |

**AR444077244** — Open Heavens Christian Centre, Edgware  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211767` | Open Heavens Christian Centre, Edgware | UKRGPR10 | ZN44400135 | AR444077244 |
| `211814` | Open Heavens Christian Centre, Edgware | UKR10PR02 | ZN44400103 | AR444077244 |

**AR444077254** — Overcomers House, Weston-Super-Mare  ·  9 members, 3 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211428` | Fountain of Grace, Telford | UKR05PR01 | ZN44400141 | AR444077254 |
| `211427` | Overcomers House, Weston-Super-Mare | UKR05PR01 | ZN44400141 | AR444077254 |
| `211489` | Overcomers House, Torquay | UKR05PR02 | ZN44400057 | AR444077254 |

**AR444077269** — Redemption Parish, Leyton  ·  5 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211995` | Redemption Parish, Leyton | UKR06PR02 | ZN44400136 | AR444077269 |
| `211498` | Redemption Parish, Leyton | UKRGPR06 | ZN44400148 | AR444077269 |

**AR444077285** — Royal Connections, London  ·  12 members, 4 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211165` | Royal Connections, London | UKR02PR03 | ZN44400110 | AR444077285 |
| `211164` | Royal Connections, London | UKR02PR03 | ZN44400110 | AR444077285 |
| `211193` | Royal Connections, London | UKRGPR02 | ZN44400157 | AR444077285 |
| `211530` | Rock of Redemption, Chadwell Heath | UKR06PR01 | ZN44400075 | AR444077285 |

**AR444077310** — The Alpha Court, Erith  ·  5 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211329` | The Alpha Court, Erith | UKR03PR03 | ZN44400196 | AR444077310 |
| `211330` | Great High Place, Welling | UKR03PR03 | ZN44400196 | AR444077310 |

**AR46601659006** — NO NAME PARISH  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `888890` | NO NAME PARISH | ICTHQ1 | ZN3050092737 | AR46601659006 |
| `370454` | ICT AREA | ICTHQ1 | ZN3050092737 | AR46601659006 |

**AR61164244** — CITY OF REFUGE AREA  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `810127` | CITY OF REFUGE | DE14 | ZN100081709 | AR61164244 |
| `375380` | CITY OF REFUGE AREA | DE14 | ZN100081709 | AR61164244 |

**AR61217364** — EAGLES ASSEMBLY AREA  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `684972` | EAGLES ASSEMBLY | DE14 | ZN100029077 | AR61217364 |
| `370103` | EAGLES ASSEMBLY AREA | DE14 | ZN100029077 | AR61217364 |

**AR61293122** — MARANTHA AREA  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `875686` | MARANATHA PARISH | DE14 | ZN0000015599 | AR61293122 |
| `374088` | MARANTHA AREA | DE14 | ZN0000015599 | AR61293122 |

**AR61344252** — REDEMPTION HALL AREA  ·  5 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `963903` | REDEMPTION HALL | DE14 | ZN4477725910 | AR61344252 |
| `379105` | REDEMPTION HALL AREA | DE14 | ZN100022688 | AR61344252 |

**AR61453508** — PILLAR OF FIRE AREA  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `616757` | PILLAR OF FIRE | DE14 | ZN100055515 | AR61453508 |
| `371860` | PILLAR OF FIRE AREA | DE14 | ZN100055515 | AR61453508 |

**AR61481264** — MY FATHER'S HOUSE  ·  5 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `841530` | LIVING HOPE AREA | LA103 | ZN268044 | AR61481264 |
| `370046` | MY FATHER'S HOUSE | LA103 | ZN268044 | AR61481264 |

**AR61544002** — Mount Camel  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `378494` | Victory House | ICTHQ1 | ZN1000002815 | AR61544002 |
| `378343` | Mount Camel | ICTHQ1 | ZN1000002815 | AR61544002 |

**AR61640830** — ELOHIM AREA  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `982179` | ELOHIM | DE14 | ZN0000015489 | AR61640830 |
| `370418` | ELOHIM AREA | DE14 | ZN0000015489 | AR61640830 |

**AR61645051** — CHRIST THE WAY AREA  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `919962` | CHRIST THE WAY | DE14 | ZN100055515 | AR61645051 |
| `378683` | CHRIST THE WAY AREA | DE14 | ZN100055515 | AR61645051 |

**AR61853682** — SOLUTION CENTRE MODEL  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `779736` | SOLUTION | DE14 | ZN0000015599 | AR61853682 |
| `493589` | SOLUTION CENTRE MODEL | DE14 | ZN0000015599 | AR61853682 |

**AR80357781788** — RESTORATION HOUSE  ·  2 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `746734` | RESTORATION HOUSE | LA117 | ZN5180583484 | AR80357781788 |
| `377859` | HOUSE OF VICTORY | LA117 | ZN5180583484 | AR80357781788 |

**ARIRE0000007** — AREA 14  ·  4 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `213191` | TRINITY CHAPEL GORT | P01 | ZNIRE0000004 | ARIRE0000007 |
| `213108` | BETHEL PARISH GALWAY ZONE | P01 | ZNIRE0000004 | ARIRE0000007 |

**ARMLD0000001** — Area 1  ·  6 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `214000` | New Life Assembly,  Antwerpen | P01 | ZNMLD0000001 | ARMLD0000001 |
| `214001` | Winners Assembly | P01 | ZNMLD0000001 | ARMLD0000001 |

**ECAFRA0006** — Area 6  ·  3 members, 2 claim `ahq`

| parishCode | parishName | province | zone | area |
|---|---|---|---|---|
| `211583` | Jesus House for all Nation | ECPFRA0001 | ECZFRA0003 | ECAFRA0006 |
| `211581` | City of Joy, le Havre | ECPFRA0001 | ECZFRA0003 | ECAFRA0006 |

---

## 2. Units with no headquarters — 127

No parish in the unit carries the flag. Nothing breaks, but the unit has no
representative row, so anything reading "the HQ of X" finds nothing.

**Resolve each with:**

```http
POST /v1/hq-assignments/assign
{ "level": "<level>", "unitCode": "<unit>", "parishCode": "<the parish to head it>",
  "dryRun": true }
```

This also sets every flag beneath that level on the same parish, and reports —
without changing — any parish that already holds one of them.

### continent — 3 of 17 (`chq`)

| continentCode | name | members |
|---|---|---:|
| `CNTRCITY` | REDEMPTION CITY CONTINENT | 83 |
| `DEPARTMENTS` | DEPARTMENTS | 78 |
| `N/A` | N/A | 2 |

### sub-continent — 3 of 18 (`schq`)

| subContinentCode | name | members |
|---|---|---:|
| `CNTRCITYSUBCNT01` | REDEMPTION CITY SUBCONTINENT | 83 |
| `DEPARTMENTS` | DEPARTMENTS | 78 |
| `ECS0002` | (no name) | 3 |

### region — 2 of 124 (`rhq`)

| regionCode | name | members |
|---|---|---:|
| `DEPARTMENTS` | DEPARTMENTS | 78 |
| `ECREU0001` | (no name) | 3 |

### province — 11 of 709 (`phq`)

| provinceCode | name | members |
|---|---|---:|
| `GO02` | GOMBE PROVINCE 2 | 57 |
| `PR371603` | Province 1 | 2 |
| `PR372447` | Province 1 | 2 |
| `PR375759` | Province 1 | 2 |
| `PR378543` | Province 1 | 2 |
| `R01` | REGION 1 HQ | 1 |
| `R02` | REGION 2 HQ | 1 |
| `R0A` | REGION A HQ | 7 |
| `R1_P1` | (no name) | 3 |
| `RCCGNATIONAL` | RCCG NATIONAL | 78 |
| `UNCATEGORIZED` | (no name) | 2 |

### zone — 43 of 7,918 (`zhq`)

| zoneCode | name | members |
|---|---|---:|
| `N/A` | N/A | 1 |
| `UNCATEGORIZED` | UNCATEGORIZED | 520 |
| `ZN0000000027` | RCCG NATIONAL | 1 |
| `ZN0000003851` | SUNSHINE Tabernacle | 14 |
| `ZN0000005895` | THE TREASURED PLACE (TREASURED PLACE ZONE) | 6 |
| `ZN0000008860` | REIGNING KING | 5 |
| `ZN0000017873` | CHRIST THE OVERCOMER | 2 |
| `ZN0000020889` | STRONG TOWER | 2 |
| `ZN0000024851` | HIS GRACE | 6 |
| `ZN0000033490` | HOUSE OF VICTORY (ZONE 6) | 10 |
| `ZN0000034170` | HE'S ALIVE | 5 |
| `ZN0000035372` | RESTORATION HOUSE (ZONE 3) | 9 |
| `ZN0000037091` | MERCY SEAT | 7 |
| `ZN0000044898` | FAVOUR (ZONE 4) | 12 |
| `ZN0000045866` | KING OF GLORY | 8 |
| `ZN0000047027` | MARVELOUS GRACE | 3 |
| `ZN0000056009` | GRACE TARBERNACLE (12) | 1 |
| `ZN0000061537` | MESSIAH'S IMPACT CHAPEL | 1 |
| `ZN0000064080` | CHAMPION (FED.LOWCOST) | 6 |
| `ZN0000066462` | CITY OF GRACE | 5 |
| `ZN0000068828` | ALL SUFFICIENT GOD Â | 8 |
| `ZN0000071517` | THE AMBASSADORS | 4 |
| `ZN100016873` | LIBERTY PAVILLION (AREA 19) | 2 |
| `ZN100022688` | GRACE MODEL | 4 |
| `ZN100029077` | TREE OF LIFE | 4 |
| `ZN100087184` | DAYSTAR (AREA 16) | 2 |
| `ZN216691` | CONQUEROR | 1 |
| `ZN373616` | Zone 1 | 2 |
| `ZN375984` | Zone 1 | 1 |
| `ZN377010` | Zone 3 | 1 |
| `ZN377146` | Zone 1 | 2 |
| `ZN379812` | Zone 1 | 2 |
| `ZN3891518293` | MESSIAH'S EMBASSY | 2 |
| `ZN44400176` | Treasure House of God, Hemel-Hempstead | 14 |
| `ZN44407008` | Open Heavens, Fife | 3 |
| `ZN500207` | RCCG GOODNESS OF GOD ZONE | 2 |
| `ZN587982` | CITADEL OF POWER | 5 |
| `ZN7935627108` | SOLOMON'S TEMPLE (AREA 9) | 1 |
| `ZNIRE0000017` | PROVINCE 4 | 1 |
| `ZNMLD0000009` | ZONE 2 | 2 |
| `ZNMLD0000011` | ZONE 2 | 4 |
| `ZNMLD0000014` | ZONE 2 | 2 |
| `ZNUKCOF01` | UK CENTRAL OFFICE | 3 |

### area — 65 of 17,616 (`ahq`)

| areaCode | name | members |
|---|---|---:|
| ` ECANLD0001` | EAST AREA | 3 |
| `AR0000000911` | FLOURISHING | 1 |
| `AR0000000923` | FLOURISHING | 1 |
| `AR0000001598` | NEW GENERATION | 1 |
| `AR0000003015` | HOLY GHOST POWER CATHEDRAL | 1 |
| `AR0000003860` | GOSHENLAND | 1 |
| `AR0000004370` | BREAKTHROUGH | 1 |
| `AR0000005000` | DOMINION SANCTUARY | 1 |
| `AR0000005001` | FREEDOM ASSEMBLY | 1 |
| `AR0000006314` | JOY UNSPEAKABLE | 1 |
| `AR0000010941` | REDEMPTION CHAPEL | 1 |
| `AR0000013056` | DAYSTAR | 1 |
| `AR0000015530` | TESTIMONY | 1 |
| `AR0000016211` | RCCG LIVING SOUL | 2 |
| `AR0000017455` | GLORY TABERNACLE | 1 |
| `AR0000018220` | VICTORY | 2 |
| `AR0000021906` | THRONE OF MERCY | 1 |
| `AR0000022908` | CHAPEL OF BREAKTHROUGH | 1 |
| `AR0000024226` | FERTILE GROUND | 1 |
| `AR0000024279` | UPPER ROOM | 1 |
| `AR0000026890` | LIVING HOPE | 4 |
| `AR0000027058` | DIVINE ASSEMBLY | 1 |
| `AR0000027184` | PROVINCE HEADQUATERS ( LIVING STONE) | 1 |
| `AR0000030187` | POWER SANCTUARY | 1 |
| `AR0000032572` | SANCTUARY OF PRAISE | 1 |
| `AR0000032681` | ROYAL ASSEMBLY (AREA 14) | 1 |
| `AR0000035372` | RESTORATION HOUSE (AREA 18) | 3 |
| `AR0000040005` | PECULIAR PEOPLES | 1 |
| `AR0000040780` | ZONE 10 | 1 |
| `AR0000042171` | CENTRE OF DELIVERANCE | 2 |
| `AR0000046094` | SOLID ROCK | 1 |
| `AR0000047196` | VICTORY CENTRE | 1 |
| `AR0000047459` | RCCG OPEN HEAVEN | 1 |
| `AR0000052550` | MIRACLE SANCTUARY | 1 |
| `AR0000052731` | SHEPHERD HOUSE | 1 |
| `AR0000053477` | THE YOUNG ADULTS AND YOUTH CHURCH | 1 |
| `AR0000053493` | SHEPHERD | 1 |
| `AR0000053823` | OPEN HEAVENS | 1 |
| `AR0000059414` | CITY OF HIS REFUGE | 3 |
| `AR0000061537` | MESSIAH'S IMPACT CHAPEL | 1 |
| `AR0000064080` | CHAMPION (FED.LOWCOST) (CHAMPION (FED. LOWCOST)) | 1 |
| `AR0000068828` | ALL SUFFICIENT GOD Â | 8 |
| `AR371225` | Area 1 | 2 |
| `AR44407015` | Open Heavens, Fife | 3 |
| `AR444077013` | Bethel Parish, Docklands | 1 |
| `AR444077130` | His Royal House, Lakeside | 1 |
| `AR444077131` | His Royal House, Lakeside | 1 |
| `AR444077237` | AREA (AR444077237) | 4 |
| `AR444077368` | World Changers Church, Rugby | 3 |
| `AR56367522772` | TRUE VINE | 1 |
| `AR60008468559` | WIINNERS PARISH | 1 |
| `AR60644857732` | HIGHER GROUND PARISH | 1 |
| `AR61006969` | RCCG PEACE OF GOD AREA | 2 |
| `AR61023276` | TRUE VINE | 1 |
| `AR61322310` | CONQUEROR ASSEMBLY | 2 |
| `AR61552745` | RCCG GRACE TABERNACLE OF RIGHTEOUSNESS AREA | 2 |
| `AR61556581` | Area 16 | 1 |
| `AR61607677` | RCCG POSSIBILITY AREA | 2 |
| `ARMLD0000010` | AREA 2 | 2 |
| `ARMLD0000013` | AREA 1 | 1 |
| `ARMLD0000015` | AREA 2 | 1 |
| `ARMLD0000017` | AREA 2 | 4 |
| `ARUKCOF01` | UK CENTRAL OFFICE | 2 |
| `N/A` | N/A | 1 |
| `UNCATEGORIZED` | UNCATEGORIZED | 482 |

---

## 3. Headquarters missing the flags beneath them

A headquarters carries its own flag and every flag below it, so the head of a
region is also the head of its province, zone and area. These rows carry the
upper flag and not the lower one, so they head a region while some other parish
heads the province they sit in — or nobody does.

**Left alone deliberately.** Each one is a question about who actually heads the
lower unit, and a sweep would answer it by assumption. Fix one by assigning the
lower unit explicitly.

| Holds | Missing | Rows |
|---|---|---:|
| `chq` (continent) | `schq` | 4 |
| `chq` (continent) | `rhq` | 4 |
| `chq` (continent) | `phq` | 25 |
| `chq` (continent) | `zhq` | 22 |
| `chq` (continent) | `ahq` | 21 |
| `schq` (sub-continent) | `rhq` | 7 |
| `schq` (sub-continent) | `phq` | 28 |
| `schq` (sub-continent) | `zhq` | 25 |
| `schq` (sub-continent) | `ahq` | 24 |
| `rhq` (region) | `phq` | 131 |
| `rhq` (region) | `zhq` | 121 |
| `rhq` (region) | `ahq` | 116 |
| `phq` (province) | `zhq` | 31 |
| `phq` (province) | `ahq` | 28 |
| `zhq` (zone) | `ahq` | 42 |

### The largest: `rhq` without `phq` — 131 rows

| parishCode | parishName | region | province |
|---|---|---|---|
| `617361` | CLOUD OF GLORY | CNT03 | CNTHQ03 |
| `666250` | GRACE COURT | CNT01 | CNTHQ01 |
| `706150` | CHOSEN GENERATION | CNT03 | CNTHQ03 |
| `927918` | PRECIOUS PEOPLE | CNT12 | CNTHQ12 |
| `774630` | ROYAL GATE | CONTINENTEMERITUSR01 | CONTINENTEMERITUS01 |
| `781327` | RCCG SOLUTION POINT | CNT02 | CNTHQ02 |
| `965500` | VICTORY LAND PARISH  | CNT02 | CNTHQ02 |
| `669907` | RCCG, GLORY PALACE | CNT12 | CNTHQ12 |
| `732277` | DIVINE VICTORY | CNT02 | CNTHQ02 |
| `897249` | THE LORD'S ARK | CNT02 | CNTHQ02 |
| `601171` | RCCG LIVING WORD | CNT02 | CNTHQ02 |
| `548635` | PENIEL PLACE | CNT02 | CNTHQ02 |
| `739170` | THE CATALYST CHURCH | CNT02 | CNTHQ02 |
| `612119` | FAITH HUB INTERNET CHURCH | CNT02 | CNTHQ02 |
| `583402` | RCCG, GLORY CLOUD PARISH | CNT12 | CNTHQ12 |
| `936442` | RCCG, HOPE OF GLORY PARISH | CNT12 | CNTHQ12 |
| `710870` | RCCG LIVING SEED, THE CHOSEN GENERATION | CNT02 | CNTHQ02 |
| `955480` | RCCG CHAPEL OF LIGHT | CNT02 | CNTHQ02 |
| `600444` | RCCG TABERNACLE OF MERCY | CNT02 | CNTHQ02 |
| `962161` | LIVING SEED CHURCH -THE EMERGING CENTRE | CNT02 | CNTHQ02 |
| `579169` | RCCG LIGHT GENERATIONS, EXPRESSION CHURCH (CONT 3.ANNEX PARISH) | CNT12 | CNTHQ12 |
| `907359` | REVELATION PARISH | R20 | RG20 |
| `883665` | Overcomers Assembly | R31 | RG31 |
| `606013` | SURE MERCIES OF DAVID | R46 | RG46 |
| `650251` | MARANATHA CHAPEL | R29 | RG29 |
| `673829` | SPIRIT AND LIFE PARISH | R14 | RG14 |
| `674016` | HOUSE OF PRAYER | R31 | RG31 |
| `938456` | THE MASTERS COURT | R04 | RG04 |
| `708761` | R.C.C.G TOWER OF DAVID | R20 | RG20 |
| `968750` | REVIVAL CENTRE | R05 | RG05 |
| `788639` | CHAPEL OF GLORY | R05 | RG05 |
| `971411` | ROSE OF SHARON MEGA | R34 | RG34 |
| `902038` | CHRIST CHURCH | R17 | RG17 |
| `609681` | RESURRECTION POWER SANCTUARY | R17 | RG17 |
| `599781` | RCCG GRACE HUB | R01 | RG01 |
| `819365` | HIGH FLYERS (EAGLE'S WING) | R16 | RG16 |
| `651083` | HOUSE OF GLORY | R16 | RG16 |
| `605987` | CHAPEL OF GLADNESS | R05 | RG05 |
| `951668` | ALHERI PARISH | R16 | RG16 |
| `692669` | GRACE ARENA | R20 | RG20 |
| `571719` | FADAR YABO | R16 | RG16 |
| `733191` | TABERNACLE OF GLORY | R23 | RG23 |
| `670430` | HIGHFLYERS ASSEMBLY | R35 | RG35 |
| `577100` | AMG FELLOWSHIP | R16 | RG16 |
| `916277` | RCCG OVERCOMERS SANCTUARY | R31 | RG31 |
| `710658` | CELEBRATION HOUSE | R16 | RG16 |
| `595779` | JUBILEE HOUSE | R16 | RG16 |
| `760315` | CHERITH PARISH | R16 | RG16 |
| `763794` | HOUSE OF HOPE | R55 | RG55 |
| `775488` | HOUSE OF GLORY | R15 | RG15 |
| `569878` | RCCG HOUSE OF VICTORY PARISH | RCITYREG01 | REDEMPTIONCITYRG01 |
| `614309` | RCCG WORD ALIVE | RWC01 | RGWC01 |
| `790776` | GLORY ASSEMBLY | RWC01 | RGWC01 |
| `962933` | EAGLES COURFT | RWC01 | RGWC01 |
| `758022` | PATHFINDERS | R16 | RG16 |
| `654614` | VESSELS OF GOLD | R16 | RG16 |
| `881108` | HOUSE OF DOMINION | RWC01 | RGWC01 |
| `855539` | FATHERS HOUSE | RWC01 | RGWC01 |
| `627450` | FLOOD GATES OF HEAVEN | R16 | RG16 |
| `745909` | ROYAL ASSEMBLY PARISH  | RWC04 | RGWC04 |

_71 more — `GET /v1/hq-assignments/integrity` returns the full list._

---

## 4. Flag values no query can match — 3 rows

Every reader in the codebase compares these columns to the **string** `"1"`.
Anything else — `"2"`, a parish code, a number — is invisible: it looks set and
is unset. `PATCH /v1/parishDirectory/:id` produced these, validating the columns
as `Joi.number()` against a `String` schema with no role guard at all.

**`887690`** — RCCG NATIONAL  ·  parishType `AREA`

| field | value | should be |
|---|---|---|
| `phq` | `"2"` | `"1"` or `"0"` |
| `zhq` | `"2"` | `"1"` or `"0"` |

- `_id` `695bbf5f5c276df63321d9f2`

**`974486`** — YP13 HQTRS  ·  parishType `ZONE`

| field | value | should be |
|---|---|---|
| `phq` | `"2"` | `"1"` or `"0"` |

- `_id` `695bbfc75c276df6332285d1`

**`(no parishCode)`** — (no parishName)

| field | value | should be |
|---|---|---|
| `parish` | `"211001"` | `"1"` or `"0"` |

- `_id` `965ddc270175cc8bdb4499a3`

**Resolve with** `POST /v1/hq-assignments/vacate` to clear a bad headquarters
flag, or `POST /v1/hq-assignments/assign` to set the cascade properly. The row
carrying `parish: "211001"` has no `parishCode` and no `parishName` at all — it
is a stray document, and needs a person to decide whether it should exist.

---

## 5. The `parish` marker — 81 parishes unset, 41 departments wrongly set

`parish: "1"` means "this row is a real parish"; `"0"` means department or
placeholder. Filtering on `parish: "1"` is therefore missing 81 churches and
returning 41 departments that are not places.

**Resolve all of them at once:**

```bash
# reports what it would change, writes nothing
curl -X POST -H "Authorization: Bearer $SUPERADMIN_JWT" \
  "$API_HOST/v1/hq-assignments/admin/repair-parish-flag?dryRun=true"

# applies it
curl -X POST -H "Authorization: Bearer $SUPERADMIN_JWT" \
  "$API_HOST/v1/hq-assignments/admin/repair-parish-flag?dryRun=false"
```

This is the one sweep in this document that is safe to run unattended: the
correct value follows from `parishType` alone, with nothing to decide.

### Parishes missing the marker (first 60 of 81)

| parishCode | parishName | parishType | parish | province |
|---|---|---|---|---|
| `` |  |  | `"211001"` |  |
| `214015` | Breakforth Parish | AREA HQ | `null` | P01 |
| `214001` | Winners Assembly | AREA HQ | `null` | P01 |
| `214006` | House of Prayer | AREA HQ | `null` | P01 |
| `214003` | Christ's Ambassadors | PARISH | `null` | P01 |
| `214018` | House of Peace, Pancevo | PARISH | `null` | P02 |
| `214019` | House of Peace, Novi Sad | PARISH | `null` | P02 |
| `214022` | Winners House of Miracles | PARISH | `null` | P02 |
| `214026` | Testimony Parish Caen | PARISH | `null` | P03 |
| `214028` | Redemption House | PARISH | `null` | P03 |
| `214032` | True Vine | PARISH | `null` | P03 |
| `214008` | Ressurrection Parish | PARISH | `null` | P01 |
| `214009` | Open Haven | PARISH | `null` | P01 |
| `214011` | Faith Tabernacle | PARISH | `null` | P01 |
| `214012` | Christ's Love Assembly | PARISH | `null` | P01 |
| `214014` | Fruitful Vine | PARISH | `null` | P01 |
| `214016` | Worship The King Parish | PARISH | `null` | P01 |
| `214033` | Jubillee  House  | PARISH | `null` | P03 |
| `214034` | Jesus House For All Nations | PARISH | `null` | P03 |
| `214037` | House of Victory | PARISH | `null` | P04 |
| `214038` | Treasure of David | PARISH | `null` | P04 |
| `214040` | Jesus Arena | PARISH | `null` | P04 |
| `214004` | Bethel House | PARISH | `null` | P01 |
| `214020` | House of Glory Tirana | PARISH | `null` | P02 |
| `214021` | Glory House | PARISH | `null` | P02 |
| `214023` | Sanctuary of Power for All Nations | PARISH | `null` | P02 |
| `214024` | Jesus House Roen | PARISH | `null` | P03 |
| `214025` | Court of Reconciliation | PARISH | `null` | P03 |
| `214035` | Jesus House Grenoble | PARISH | `null` | P04 |
| `214039` | Gospel Chapel | PARISH | `null` | P04 |
| `214002` | Covenant Centre | PARISH | `null` | P01 |
| `214005` | Victory House Brussels | PARISH | `null` | P01 |
| `214007` | Jesus House | PARISH | `null` | P01 |
| `214013` | Living Waters | PARISH | `null` | P01 |
| `214027` | LSC The Bridge | PARISH | `null` | P03 |
| `214029` | Victory House  | PARISH | `null` | P03 |
| `214031` | The Lighthouse  | PARISH | `null` | P03 |
| `214062` | The King Parish Verona | PARISH | `null` | P02 |
| `214063` | Potter’s House Terni | PARISH | `null` | P02 |
| `214069` | House of Praise Voghera | PARISH | `null` | P03 |
| `214068` | House of David Cuneo | PARISH | `null` | P04 |
| `214061` | Dominion Domain Perugia | PARISH | `null` | P05 |
| `214047` | Goshen Parish | PARISH | `null` | P04 |
| `214048` | Jesus House, Pau | PARISH | `null` | P04 |
| `214054` | Freedom Hall Firenze | PARISH | `null` | P01 |
| `214058` | Strong Tower Modena | PARISH | `null` | P01 |
| `214066` | Sanctuary of Peace Treviso | PARISH | `null` | P02 |
| `214065` | Great House Orte | PARISH | `null` | P04 |
| `214073` | Restoration House Viterbo | PARISH | `null` | P04 |
| `214072` | Overcomers Parish Città di Castello | PARISH | `null` | P05 |
| `214074` | Mercy Land Gualdo Tadino | PARISH | `null` | P05 |
| `214043` | Jubilee Covenant | PARISH | `null` | P04 |
| `214044` | Shephard’s Hill | PARISH | `null` | P04 |
| `214051` | Destiny Sanctuary Torino | PARISH | `null` | P01 |
| `214052` | City of Hope Torino | PARISH | `null` | P01 |
| `214067` | Garden of Peace Genova | PARISH | `null` | P03 |
| `214070` | Jesus House Alessandria | PARISH | `null` | P03 |
| `214071` | Covenant Sanctuary Cagliari | PARISH | `null` | P03 |
| `214057` | Heaven's Gate Assembly Bergamo | PARISH | `null` | P05 |
| `214079` | Victory House Malta | PARISH | `null` | P10 |

---

## 6. Codes living under two parents — 340

A zone code identifies one zone in the whole tree. `ZN200001` may exist under
LA47 **or** LA54, never both — otherwise every query that groups by `zoneCode`
silently merges two different zones.

Excluded from this list: the sentinel codes `UNCATEGORIZED` and `N/A`, which are
buckets rather than units. `UNCATEGORIZED` alone appears under 149 different
provinces; it is not a split zone, it is a holding pen, and it needs emptying
rather than resolving.

**Resolve each** by moving the members that are in the wrong parent:

```http
POST /v1/hierarchy-transfers/admin/move
{ "level": "zone", "unitCode": "<code>", "toLevel": "province", "toCode": "<the right one>",
  "dryRun": true }
```

That moves the WHOLE unit to one parent. Where the two halves are genuinely
different zones that were given one code, they need re-coding instead — see
`POST /v1/hierarchy-transfers/promote` and the code registry.

### sub-continent — 1

| code | name | parents | members |
|---|---|---|---:|
| `ICT1SUBCNT01` | ICT 1 SUBCONTINENT 1 | `ICT1` `ECCT254254` `ECCT613510` `ECCT514652` `ECCT235592` | 62 |

### region — 5

| code | name | parents | members |
|---|---|---|---:|
| `ICT1` | ICT 1 | `AUPSUBCONT01` `ECSC670359` `ICT1SUBCNT01` `ECSC834007` | 43 |
| `R01` | REGION 1 | `CNT03SUBCNT01` `ECS0003` | 1026 |
| `R02` | REGION 2 | `CNT03SUBCNT01` `CNT01SUBCNT01` `ECS0003` | 766 |
| `R05` | REGION 5 | `CNT01SUBCNT01` `ECS0001` | 1060 |
| `R06` | REGION 6 | `ICT1SUBCNT01` `CNT01SUBCNT01` `ECS0001` | 910 |

### province — 6

| code | name | parents | members |
|---|---|---|---:|
| `ICTHQ1` | ICT HQ | `ECREU0962` `ECREU2556` `ECREU2423` `ECREU1646` `AUP2` `AUP1` `ECREU0939` `ECREU5265` _+14 more_ | 61 |
| `P01` | PROVINCE 1 | `R01` `R06` `R05` | 41 |
| `P02` | PROVINCE 2 | `R0A` `R06` `R05` | 38 |
| `P03` | PROVINCE 3 | `R02` `R06` `R05` | 33 |
| `P04` | PROVINCE 4 | `R0A` `R06` `R05` | 34 |
| `P05` | PROVINCE 5 | `R0A` `R05` | 17 |

### zone — 28

| code | name | parents | members |
|---|---|---|---:|
| `ZN0000000803` | CANAANLAND (GLORIOUS ZONE) | `LA110` `LA82` | 13 |
| `ZN0000005480` | WORD OF LIFE | `OY18` `OY24` | 12 |
| `ZN0000020888` | MOUNT OF PRAISE | `OS08` `OS17` | 9 |
| `ZN0000035508` | THE KING IS COMING | `REDEMPTIONCITY01` `REDEMPTIONCITYRG01` | 8 |
| `ZN0000040987` | DAVIDS COURT | `LA113` `LA76` | 12 |
| `ZN0000048588` | N/A | `REDEMPTIONCITYRG01` `REDEMPTIONCITY01` | 9 |
| `ZN0000057096` | CHANNEL OF BLESSING | `LA69` `LA104` | 17 |
| `ZN100012691` | ICT AREA | `ECPR313904` `ECPR471394` `ECPR256709` `ECPR496006` `ECPR216570` `ECPR697627` `ECPR244142` `ECPR316369` _+16 more_ | 52 |
| `ZN44400016` | Christos Palace, Stevenage | `UKRGPR01` `UKR11PR01` | 15 |
| `ZN44400023` | Conquerors Assembly, Cheltenham | `UKRGPR01` `UKR05PR01` | 11 |
| `ZN44400024` | Covenant of Peace, Sidcup | `UKR01PR04` `UKRGPR01` | 18 |
| `ZN44400052` | Freedom House, Dartford | `UKR07PR02` `UKR07PR01` | 16 |
| `ZN44400060` | Grace Chapel, Chesterfield | `UKR01PR04` `UKRGPR01` | 16 |
| `ZN44400073` | Hope Centre, York | `UKR01YP01` `UKRGPR01` | 6 |
| `ZN44400079` | House of Joy for all Nations, Harrow | `UKRGPR01` `UKR11PR02` | 21 |
| `ZN44400106` | Kings Court Chapel, Milton Keynes | `UKR10PR02` `UKRGPR01` | 16 |
| `ZN44400118` | Living Water Parish, Stoke-on-Trent | `UKR11PR01` `UKRGPR11` | 2 |
| `ZN44400123` | My Father's House, Salford | `UKR02PR02` `UKRGPR01` | 20 |
| `ZN44400127` | New Life Assembly, Borehamwood | `UKR08PR02` `UKR08PR01` | 12 |
| `ZN44400156` | Rivers of Love, Woolwich | `UKR03PR01` `UKRGPR01` | 14 |
| `ZN44400165` | The Chapel, Norwich | `UKRGPR01` `UKR09PR01` | 2 |
| `ZN44400167` | The City of God, Crayford | `UKRGPR01` `UKR07PR01` | 10 |
| `ZN44400171` | The New Creation Assembly, South Norwood | `UKRGPR01` `UKR03PR02` | 17 |
| `ZN44400189` | Victory House, London | `UKR01PR01` `UKRGPR01` | 2 |
| `ZN529076` | ICT ZONE | `ICTHQ1` `ECPR265702` | 4 |
| `ZNIRE0000025` | REGION A | `RG0A` `R0A` | 8 |
| `ZNMLD0000003` | Zone 1 | `P02` `P03` | 13 |
| `ZNMLD0000004` | Zone 1 | `P03` `P04` | 6 |

### area — 300

| code | name | parents | members |
|---|---|---|---:|
| `AR0000000438` | HOPE OF GLORY | `ZN0000001594` `ZN0000006161` | 3 |
| `AR0000001046` | HALL OF FAVOUR | `ZN0000000991` `ZN0000001255` | 3 |
| `AR0000001069` | ANOINTED THRONE | `ZN0000001080` `ZN0000001012` | 3 |
| `AR0000001094` | OPEN DOOR ASSEMBLY (AREA 33) | `ZN0000001094` `ZN0000001012` | 4 |
| `AR0000002189` | THE TRIUMPHANT COMPANY (TRIUMPHANT AREA) | `ZN0000002189` `ZN0000016385` | 3 |
| `AR0000002190` | HIS  MIGHTY FORTRESS | `ZN0000016385` `ZN0000002190` | 2 |
| `AR0000002629` | Fresh Fire Cathedral Ekiti 2 HQ | `ZN926932` `ZN0000002629` | 2 |
| `AR0000002657` | GLORYLAND  (PROVINCIAL HQ.) | `ZN767200` `ZN0000002657` | 2 |
| `AR0000002964` | HOSANNAH (AREA 5) | `ZN0000044898` `ZN4401860464` | 4 |
| `AR0000002974` | FOUNTAIN OF JOY | `ZN0000039322` `ZN100074693` | 3 |
| `AR0000002984` | HOUSE OF JOY (HOUSE OF JOY) | `ZN0000002984` `ZN888360` | 2 |
| `AR0000003094` | HALLELUYAH AREA | `ZN0000003094` `ZN0000002600` | 5 |
| `AR0000003239` | JESUS CITY | `ZN0000017773` `ZN0000007797` | 6 |
| `AR0000003286` | JESUS JEWEL | `ZN0000003284` `ZN0000017773` | 6 |
| `AR0000003567` | CITADEL OF HOPE | `ZN0000003519` `ZN0000003527` | 5 |
| `AR0000003851` | SUNSHINE Tabernacle | `ZN0000003851` `ZN9162586006` | 3 |
| `AR0000004085` | ABIDING GRACE | `ZN0000008077` `ZN0000004085` | 2 |
| `AR0000004096` | SANCTUARY OF TESTIMONY | `ZN0000004105` `ZN0000003674` | 4 |
| `AR0000004143` | LIGHT HOUSE | `ZN0000003674` `ZN0000004143` | 5 |
| `AR0000004622` | DIVINE PEACE | `ZN0000004175` `ZN2877397749` | 2 |
| `AR0000004713` | GREATER GLORY | `ZN0000004776` `ZN0000004667` | 3 |
| `AR0000004718` | ZION GATE | `ZN0000004718` `ZN0000004776` | 3 |
| `AR0000004720` | THE LORDS GLORY ASSEMBLY | `ZN8230354029` `ZN0000042812` | 3 |
| `AR0000004731` | CHAPEL OF GLORY | `ZN0000004731` `ZN0000004776` | 2 |
| `AR0000004767` | GRACE SANCTUARY | `ZN0000004767` `ZN0000004764` | 4 |
| `AR0000005491` | CALEB GENERATION | `ZN0000003559` `ZN0000005491` | 4 |
| `AR0000005493` | KING OF KINGS | `ZN0000003559` `ZN0000005491` | 4 |
| `AR0000005527` | TRIUMPHANT (BOMADI) | `ZN0000005527` `ZN0000006838` | 5 |
| `AR0000005556` | RCCG STRONG TOWER | `ZN0000005551` `ZN0000003674` | 6 |
| `AR0000005593` | MOUNT ZION | `ZN0000005507` `ZN100023833` | 7 |
| `AR0000005685` | CHAPEL OF HIS EXCELLENCY | `ZN0000015398` `ZN0000005685` | 2 |
| `AR0000005689` | AMAZING GRACE | `ZN0000015398` `ZN0000005689` | 4 |
| `AR0000005831` | LIVINGSTONE | `ZN0000006838` `ZN0000005831` | 5 |
| `AR0000005895` | THE TREASURED PLACE (TREASURED PLACE ZONE) | `ZN3941349903` `ZN0000005895` | 3 |
| `AR0000006009` | ABUNDANT LIFE | `ZN0000006009` `ZN0000006005` | 4 |
| `AR0000006028` | OFFSPRING OF DAVID | `ZN9507106135` `ZN0000009833` | 2 |
| `AR0000006474` | LION OF JUDAH | `ZN0000003519` `ZN0000006474` | 4 |
| `AR0000006921` | MERCY SEAT | `ZN0000006901` `ZN0000006838` | 7 |
| `AR0000007180` | HOLY GHOST PAVILLION | `ZN0000009587` `ZN0000051636` | 2 |
| `AR0000007361` | RESTORATION | `ZN0000007360` `ZN0000018649` | 5 |
| `AR0000007390` | JESUS VILLA | `ZN0000007449` `ZN0000017773` | 2 |
| `AR0000007416` | FLOURISHING HOUSE | `ZN0000021599` `ZN0000007422` | 3 |
| `AR0000007418` | SOLUTION ARENA | `ZN0000007419` `ZN0000003132` | 4 |
| `AR0000007424` | CITY OF JOY | `ZN0000007422` `ZN0000024914` | 4 |
| `AR0000007528` | JESUS REFUGE | `ZN0000017773` `ZN0000007528` | 4 |
| `AR0000007586` | VICTORY  (Idofian) | `ZN0000007579` `ZN4553764270` | 3 |
| `AR0000007700` | SOLUTION | `ZN0000007942` `ZN8287955511` | 2 |
| `AR0000007788` | CITY ON THE HILL | `ZN0000006948` `ZN0000002646` | 6 |
| `AR0000007789` | JESUS NATIONS | `ZN0000017773` `ZN0000007528` | 2 |
| `AR0000007797` | JESUS VESSEL | `ZN0000007797` `ZN0000017773` | 4 |
| `AR0000007900` | JESUS TEMPLE | `ZN0000007900` `ZN0000017773` | 4 |
| `AR0000007901` | JESUS FOUNTAIN | `ZN0000017773` `ZN0000065719` | 4 |
| `AR0000007975` | SOLUTION CHAPEL | `ZN0000007975` `ZN0000008073` | 4 |
| `AR0000007982` | MIRACLE CENTRE (AREA 19) | `ZN100042515` `ZN0000007781` | 3 |
| `AR0000008123` | ALPHA AND OMEGA MEGA | `ZN0000008123` `ZN0000006838` | 2 |
| `AR0000008149` | CITY OF FAITH | `ZN0000007422` `ZN0000007419` | 3 |
| `AR0000008218` | ABUNDANT LIFE (AREA 2) | `ZN8324162516` `ZN0000006495` | 4 |
| `AR0000008343` | EL-SHADDAI | `ZN0000008343` `ZN0000006838` | 4 |
| `AR0000008399` | SANCTUARY OF PRAISE (AREA 26) | `ZN100064375` `ZN0000008393` | 4 |
| `AR0000008645` | SUNRISE | `ZN0000008648` `ZN0000005164` | 4 |
| `AR0000008653` | ROYAL SANCTUARY | `ZN252145` `ZN0000008653` | 2 |
| `AR0000008742` | DIVINE MERCY | `ZN0000007947` `ZN7860561742` | 3 |
| `AR0000008811` | CHAPEL OF PEACE (AREA 46) | `ZN9112909191` `ZN0000013350` | 3 |
| `AR0000008860` | REIGNING KING | `ZN0000008860` `ZN0407642754` | 3 |
| `AR0000008961` | NEW LIFE ARENA | `ZN0119835980` `ZN0000007946` | 2 |
| `AR0000008963` | CANAAN LAND AREA | `ZN0000008963` `ZN0000002600` | 6 |
| `AR0000009162` | ROYAL ASSEMBLY | `ZN0000007724` `ZN7957409620` | 3 |
| `AR0000009164` | SALVATION | `ZN0119835980` `ZN0000007950` | 2 |
| `AR0000009165` | SHOWERS OF BLESSING | `ZN0000008955` `ZN6669949010` | 2 |
| `AR0000009166` | TRANSFORMATION | `ZN7957409620` `ZN0000007724` | 3 |
| `AR0000009182` | HIS ROYAL MAJESTY | `FVZN10` `ZN0000009182` `PHZN15` `WISZ17` `HPHZN18` | 5 |
| `AR0000009210` | JESUS HOUSE | `ZN0000008073` `ZN0000009210` | 4 |
| `AR0000009213` | OIL OF JOY | `ZN0000008073` `ZN0000009963` | 4 |
| `AR0000009240` | WIND OF CHANGE (AREA 28) | `ZN6475657590` `ZN0000009235` | 2 |
| `AR0000009244` | VICTORY AT LAST | `ZN0000009235` `ZN0012189792` `ZN0000009148` | 3 |
| `AR0000009386` | THRONE OF MERCY (AREA 26) | `ZN0000009159` `ZN100056249` | 3 |
| `AR0000009387` | FULLNESS OF JOY | `ZN0000009159` `ZN6475657590` | 3 |
| `AR0000009684` | WISDOM | `ZN0000025738` `ZN1761852964` | 3 |
| `AR0000009722` | NEW LIFE - EHOR | `ZN0000009722` `ZN0000006039` | 3 |
| `AR0000009731` | HIGHER GROUND MISSION ZONE | `ZN0000009731` `ZN0000013764` | 5 |

_220 more at this level._

---

## 7. Duplicate `parishCode` — 1

`parishCode` is declared unique on the model, but the index has never existed,
so nothing has ever enforced it.

| parishCode | rows |
|---|---:|
| `373486` | 2 |

**This blocks the unique index.** The org init endpoint counts duplicates
before attempting the build and reports them rather than failing with a
single `E11000`, so running init is safe — the index is simply skipped until
this is re-coded.

---

## 8. No index on any headquarters column

Confirmed against the live collection: not one of `chq`, `schq`, `rhq`, `phq`,
`zhq`, `ahq` or `parish` leads an index. "Which parish heads LA47?" is a scan of
53,049 rows, and the statistics endpoint does it six times over.

**Resolve with the org init endpoint**, which now carries all six:

```bash
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388?dryRun=true"
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388"
```

Not created: a unique index enforcing one headquarters per unit. It cannot build
while 130 units have two, and forcing it would mean picking a loser
automatically — which is the one thing this whole feature refuses to do.

---

## Suggested order

1. **Repair the `parish` marker** (§5) — mechanical, nothing to decide.
2. **Re-code the duplicate `parishCode`** (§7) — one code, then the unique index builds.
3. **Build the indexes** (§8) — everything below gets faster and safer.
4. **Fix the 3 invalid flag values** (§4) — tiny, and they are actively lying.
5. **Settle the 130 contested units** (§1), starting at continent and working down:
   a province resolved first can be re-contested by a region assignment above it.
6. **Fill the 127 vacancies** (§2), top down for the same reason.
7. **Split codes** (§6) and **cascade gaps** (§3) — largest, and each needs a person
   who knows the churches involved.

Steps 1–3 are safe to do now. Everything from 4 down changes who represents a
unit, and every endpoint above takes `"dryRun": true`.
