# Authority matrix — hierarchy, unit transfers and principal officers

Who may perform each operation, and who must approve it. This is the technical
copy, with endpoints and error codes. The administrator's copy, without
endpoints, is `authority_matrix_for_hierarchy_and_unit_transfers_users.md`.

State of the `dev` branch at `023cbb8` (2026-09-18). Rows marked **dev only**
are not yet in production; production runs `main`.

## Key

| Term | Meaning |
|---|---|
| **Unbounded** | `super-admin`, or an elevated role — `nat-support` by default (`ELEVATED_ROLES`). Passes every check below. |
| **Administrator** | A role in `HIERARCHY_RESTRUCTURE_ROLES`: `prov-admin`, `reg-admin`, `sub-cont-admin`, `cont-admin`. Assistants and `picp` are never administrators. |
| **Standing** | The units a caller acts in: one code per level, taken from the roles they hold and the matching code on their own profile (`province`, `region`, …). Non-administrative roles such as `training-manager` give no standing. |
| **Contains** | One of the caller's units contains the target, from at or above its level, judged on the target's ancestry as resolved from the database — never from the request. |
| **Inside** | Both ends of a move sit within one unit the caller holds standing in. |

---

## 1. Unit moves — parish, area, zone, province

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Move a unit **within** their own unit | `POST /v1/hierarchy-transfers/transfer` | Anyone with standing over a unit containing **both** ends — `picp`, `prov-admin`, `reg-admin` … — or unbounded | None |
| **Pull** a unit in from another province | `POST /v1/hierarchy-transfers/transfer` | **Administrator** of the receiving unit — `prov-admin` of the destination province, `reg-admin` of a region containing it — or unbounded. **dev only** (`b566ab8`) | None. The receiving administrator's action *is* the approval. A pending `UNIT_TRANSFER` aimed at their province is settled `APPROVED` by it; one aimed elsewhere refuses `409 REQUEST_PENDING_ELSEWHERE`. |
| Same pull, by a non-administrator (`picp`, `prov-asst-admin`) | `POST /v1/hierarchy-transfers/transfer` | Refused `403 TRANSFER_NOT_PERMITTED`; the message names the administrator who can | — |
| **Push** a unit out to another province | `POST /v1/approvals/unit-transfer` (raise) | `prov-admin` of the **source** province, anyone whose unit contains it, or unbounded | `prov-admin` of the **receiving** province; `reg-admin` whose region contains the destination (**dev only**, `00f4efe`); or unbounded. Never the raiser. `403 TRANSFER_NEEDS_APPROVAL` if attempted directly. |
| Move any unit, any level, any distance | `POST /v1/hierarchy-transfers/admin/move` | Unbounded only | None |
| A move that would **strand a headquarters** | `/transfer`, `/admin/move`, `/admin/realign` with `acknowledgeDemotion: true` | Unbounded only. Everyone else is refused and told to vacate the HQ first via `POST /v1/hq-assignments/vacate` | None |
| Realign a split unit (repair) | `POST /v1/hierarchy-transfers/admin/realign` | Unbounded only | None |
| Realign within own scope | `POST /v1/hierarchy-transfers/scoped/realign` | Administrator with standing over the unit **and** the destination | None |
| Preview any of the above | `GET /v1/hierarchy-transfers/preview` | Any signed-in user; reports `permissionCode` and `permittedVia` for the caller | — |
| Roll back a completed job | `POST /v1/hierarchy-transfers/jobs/:id/rollback` | Unbounded only | None |
| List / inspect jobs | `GET /v1/hierarchy-transfers/jobs`, `/jobs/:id` | Unbounded only | — |
| Integrity report (split units, orphans) | `GET /v1/hierarchy-transfers/integrity` | Unbounded, or a scoped API key | — |

## 2. Promotion of units — parish → area, area → zone, …

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Promote any unit, with `absorbCodes` | `POST /v1/hierarchy-transfers/promote` | Unbounded only | None |
| Promote within own scope | `POST /v1/hierarchy-transfers/scoped/promote` | Administrator with standing over the unit. `absorbCodes` is refused for a bounded caller | None |
| Promote a unit whose HQ would be demoted | either, with `acknowledgeDemotion` | Unbounded only | None |

## 3. Principal officers — appointing and ending

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Appoint an officer at their **own** unit (`prov-admin` appoints `picp` of LA47) | `POST /v1/principal-officers` | Any officer with standing at that exact unit, or unbounded | None |
| Appoint an officer in a unit they **contain** (area-admin appoints `pic-parish` in AR1; `prov-admin` anywhere in LA47) | `POST /v1/principal-officers` | Administrator of the containing unit, or unbounded. **dev only** (`023cbb8`) | None |
| Appoint upward (area-admin appoints a province officer) | `POST /v1/principal-officers` | Refused `403 NO_STANDING_AT_UNIT` | Raise `officer-promotion` instead |
| Appoint into a unit that is not the user's own | `POST /v1/principal-officers` with `scopeCode` | Same as above, **and** the target user must hold an active secondary grant of that role there | None |
| Appoint a non-administrative office (`training-manager`) | `POST /v1/principal-officers` | Same rules; the holder gains **no** administrative authority | None |
| End an appointment | `POST /v1/principal-officers/:id/end` | Unbounded only | None |
| Transfer an appointment to another person | `POST /v1/principal-officers/:id/transfer` | Unbounded only | None |

## 4. Principal officers — through approvals

| Action | Endpoint | Who can raise | Who approves / rejects |
|---|---|---|---|
| Replace the holder of an office | `POST /v1/approvals/officer-transfer` | Any authenticated user | Administrator whose unit **contains** the office, at or above its level, or unbounded. Never the raiser, except a `super-admin`. |
| Move a person into a different office | `POST /v1/approvals/officer-promotion` | Any authenticated user | Same as above |
| Cancel either | `POST /v1/approvals/:id/cancel` | The raiser, or unbounded | — |

## 5. Principal officers — register hygiene

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Roles held with no appointment / appointments with no role | `GET /v1/principal-officers/register-audit` | Unbounded | — |
| Strip an office role from users who hold no appointment | `POST /v1/principal-officers/admin/strip-roles` | Unbounded. Refuses `ACTIVE_APPOINTMENT` when the holder is actually appointed | None |
| Two people holding one office | `GET /v1/principal-officers/conflicts` | Unbounded | — |
| Offices with no holder | `GET /v1/principal-officers/vacancies` | Unbounded | — |
| Roster of offices and holders for the caller's own units | `GET /v1/principal-officers/roster` | Any signed-in officer; sees only units they hold standing at. Each row carries `canAppoint` for the caller | — |
| Full list, single record, one office, config | `GET /v1/principal-officers`, `/:id`, `/office`, `/config` | Unbounded only | — |

## 6. Pastor in charge of a parish (the PIC record)

Distinct from the principal-officer register: `parishPicHolders` says who
actually leads a parish. Holding the `pic-parish` role does not.

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Appoint the pastor in charge of a parish | `POST /v1/parish-pics` | Administrator of any unit containing the parish — area, zone, province, region, sub-continent, continent — or unbounded. Judged on the parish's own ancestry | None |
| The parish's own pastor (`pic-parish`) appointing their successor | `POST /v1/parish-pics` | Refused `403 NO_STANDING_OVER_PARISH` — a parish cannot appoint its own head | — |
| End a PIC appointment | `POST /v1/parish-pics/:id/end` | Same as appointing | None |
| Who leads this parish / list appointments | `GET /v1/parish-pics/parish/:parishCode`, `GET /v1/parish-pics` | Any signed-in user | — |

## 7. People and headquarters

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Move a user to another parish | `POST /v1/approvals/user-transfer` (raise) | Any authenticated user | `prov-admin` of the **source or destination** province, or unbounded. Never the raiser. |
| Assign / vacate / resolve a headquarters | `POST /v1/hq-assignments/assign`, `/vacate`, `/resolve-conflict` | Standing over the unit, or unbounded | None |
| Assign roles to a user | `PATCH /v1/users/:id/roles` | Gated by `ROLE_ASSIGNMENT_ENFORCEMENT`: the caller may grant only roles listed by `GET /v1/users/me/allowed-roles` | None |

## 8. Approval lifecycle

| Action | Endpoint | Who |
|---|---|---|
| Approve / reject | `POST /v1/approvals/:id/approve`, `/reject` | Whoever the type's rule above names. Nobody decides their own request except a `super-admin`. |
| Cancel | `POST /v1/approvals/:id/cancel` | The raiser, or unbounded |
| View | `GET /v1/approvals`, `/:id` | Requests in the caller's units; unbounded sees all |

---

## Not yet in production

| Commit | Change |
|---|---|
| `00f4efe` | A region administrator may approve a unit transfer between two provinces in their region |
| `b566ab8` | The receiving province's administrator may pull a unit in through `/transfer` alone |
| `023cbb8` | An administrator may appoint principal officers in the units they contain |

Production still requires the two-sided raise-and-approve flow for every
cross-province move, and exact-level standing for principal-officer appointments.

## Open question

Ending and transferring a principal-officer appointment remain unbounded-only
while appointing is delegated. An area-admin can appoint a pastor to the
register but cannot end that appointment. Bringing `/:id/end` and
`/:id/transfer` under the same containment rule is a one-line decision.
