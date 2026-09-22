# Principal officers — status and plan

Living document. Update the checkboxes as phases land, so anyone picking this up
knows where it stopped.

**Current state: phases 1–5 built and merged, enforcement OFF, nothing cleaned up.**

---

## What this is

Some roles are *offices*, not entitlements: there is one `prov-admin` of a
province, one `picr` of a region. Holding the role is not the same as being the
officer. This adds a register that records who actually holds each office, and
checks it when someone tries to act as one.

**Nothing in the system behaves differently until `PRINCIPAL_OFFICE_ENFORCEMENT`
is turned up.** It ships `off`.

---

## Design in one paragraph

`roles.principalOffice` (boolean) says which roles are offices — data, not a slug
list in code. `principalOfficeHolders` records who holds each one, with a partial
unique index on `{roleSlug, scopeCode, appointmentType}` where `active: true` —
that index *is* the guarantee. The unit comes from the role's existing
`level_type` via `LEVEL_MAP`, read off the holder's own profile for a primary
role or off a secondary grant for a granted one.

**Why a register rather than a constraint on roles:** `users.roles` is a *String*
(`'["picp","prov-admin"]'`), so no index can address one role inside it; and
entitlement lives in two collections (`users.roles` and
`secondaryRoleAssignments`), which no single unique index can span. A register is
the only place a database-level guarantee can live.

---

## The 26 office roles

Supplied by the business. Province level and above only.

`apicr-csr` and `apicp-csr-2` were added on 2026-09-21. `apicp-csr-2` is a
SECOND province CSR office, not a second holder of `apicp-csr`: uniqueness is per
slug per unit, so one province may hold one of each and no more.

| Level | Roles |
|---|---|
| province | `picp`, `prov-admin`, `prov-accountant`, `prov-asst-admin`, `prov-asst-accountant`, `apicp-admin`, `apicp-csr`, `apicp-csr-2` |
| region | `picr`, `apicr`, `apicr-csr`, `reg-admin`, `reg-accountant`, `reg-asst-admin`, `reg-asst-accountant`, `cgo` |
| sub-continent | `sco`, `asco`, `sub-cont-admin`, `sub-cont-accountant`, `sub-cont-ict`, `training-manager` |
| continent | `co`, `aco`, `cont-admin`, `cont-accountant` |

> **`training-manager` is an office, but not an administrative one.** It is
> listed here because it is a single-holder post — one per sub-continent, as the
> table says. It carries **no administrative authority** at that level: it cannot
> write geofencing rules, grant exemptions, appoint principal officers, move
> units, assign headquarters or approve transfers. Its remit is training, and it
> may create and manage the training managers under it. Enforced by
> `NON_ADMINISTRATIVE_ROLES` in `src/utils/officerAuthority.ts`, which
> `standingOf` consults so all six authority surfaces inherit it, and which is
> set from `NON_ADMINISTRATIVE_ROLES` in the environment.
>
> Region, province, zone, area and parish training managers are planned. **Each
> new slug must be added to that list** or it will silently inherit
> administrative authority at its own level.


Two things to **not** "correct" later:

- **Assistant roles are included.** An assistant post here is a single named
  office, unlike departmental assistants in `orgDesignations` where several may
  serve. This was an explicit decision.
- **The parish/area/zone tier is excluded.** `pic-parish` alone has 51,551 holders
  across 50,059 parishes; making it an office means seeding fifty thousand
  appointments and adjudicating 1,283 contested parishes first. Province and above
  is ~1,900 offices.

---

## Measured starting position

From production, 56,865 users, 108 roles:

| | |
|---|---|
| Distinct (role, scope) pairs, all 42 geographic roles | 79,520 |
| Pairs with more than one holder | 2,266 (2.8%) |
| Excess holders across all geographic roles | 2,951 |
| Excess within the then-24 office roles | **472** (measured 2026-09-03) |

Measured run, 2026-09-03, **across the 24 roles flagged at that date**:
**3,206 offices claimed, 2,950 uncontested and safe to seed, 256 contested,
472 excess claimants.** `apicr-csr` and `apicp-csr-2` were added afterwards and
are NOT in these figures; re-run `scripts/analysePrincipalOffices.js` for
current ones rather than adjusting them by hand. Full output in
`duplicate-office-holders-summary.txt`.

`ICTHQ1` (88 parishes) and `ICT1` (48) are **real** codes, not test data.
`632101` is "RCCG GLOBAL PARISH", a row typed `ZONE` — excluded via
`EXCLUDED_SCOPE_CODES`.

---

## Phases

- [x] **1 — Foundation.** `roles.principalOffice`, `principalOfficeHolders` model,
      indexes added to the org init endpoint, `utils/principalOffice.ts`.
- [x] **2 — Analysis.** `scripts/analysePrincipalOffices.js`, read-only, plus the
      `/conflicts` and `/vacancies` endpoints.
- [x] **3 — Flag the roles.** `scripts/flagPrincipalOfficeRoles.js`.
      *Built and dry-run against production (24/24 matched, before `apicr-csr` and
      `apicp-csr-2` were added). **Not yet applied.***
- [x] **4 — Visibility.** `officeStatus` on `/v1/users/me/roles` and the login
      `roles` array; `/v1/principal-officers/*` read endpoints.
- [x] **5 — Appointment CRUD.** appoint / end / transfer, super-admin only.
- [x] **6 — The check.** `switch-role` consults the register. Ships `off`.
- [x] **6a — Level-scoped visibility and appointment.** `/roster` shows filled and
      vacant offices with contact details to anyone with standing at that unit;
      appointment opened from super-admin-only to officers at their own unit.
- [x] **6b — Approval workflow.** `approvalRequests` + `/v1/approvals/*`: officer
      transfer and promotion (super-admin approves), user parish transfer
      (province admin of either province, or super-admin). See
      `APPROVALS_AND_TRANSFERS_DOCS.md`.
- [ ] **7 — Seed the register.** Backfill uncontested offices; leave contested ones
      vacant for a human. *Script not written yet — deliberately, so the
      conflicts report can be reviewed first.*
- [ ] **8 — Turn to `warn` on dev**, then production. Read the WARNING activity
      logs to measure the real blast radius.
- [ ] **9 — Turn to `enforce`**, per environment, once the warn logs are quiet.
- [ ] **10 — Cleanup.** Strip confirmed non-holders from `users.roles`. Separate
      approval, backup collection first, per the duplicate-user precedent.
- [ ] **11 — Acting appointments.** `appointmentType` is already in the index key,
      so no index rebuild is needed. Needs date-window evaluation and an expiry
      sweeper — the existing org assignments record `effectiveFrom`/`effectiveTo`
      but never evaluate them, and that weakness must be fixed here first.

---

## Deployment steps, in order

```bash
# 1. Create the indexes (production runs MONGO_AUTO_INDEX=false)
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388?dryRun=true"
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388"

# 2. Flag the 26 roles — dry run first
node scripts/flagPrincipalOfficeRoles.js --dry-run
node scripts/flagPrincipalOfficeRoles.js

# 3. Read the real numbers
node scripts/analysePrincipalOffices.js
node scripts/analysePrincipalOffices.js --contested   # the worklist

# 4. Only then: phase 7 onwards
```

Step 2 is reversible with `--clear`. Steps 1–3 change no behaviour: with
enforcement off, `switch-role` never even calls the resolver.

---

## Environment

```
PRINCIPAL_OFFICE_ENFORCEMENT=off      # off | warn | enforce
```

- `off` — no check, no query. The default and the current setting.
- `warn` — allow the switch, but write a `WARNING` activity log saying what would
  have been refused. This is how the blast radius gets measured.
- `enforce` — refuse with `409`.

Fails closed: only the exact strings `warn` and `enforce` do anything. An unset,
misspelled, or inline-commented value means `off` — dotenv 4 does not strip
inline `#` comments, which is how New Relic log forwarding sat silently disabled
for weeks.

---

## Known gaps, deliberately not closed here

**Three unguarded paths can still write roles**, so any rule enforced at
`switch-role` is bypassable at the point of assignment:

| Where | Problem |
|---|---|
| `PATCH /v1/users/:id/roles` | No guard — any authenticated user can rewrite any user's roles. |
| `PATCH /v1/users/:id` | `Users/service.ts` computes `safeBody` to strip `roles`, then updates with `body` instead. Dead code. |
| ~~`PATCH /v1/users/:id/roles`~~ | **Partly closed.** Still unguarded as to *who* may call it, but *what* it can assign is now vetted: super-admin never, and an office never while someone else holds it. |
| ~~`PATCH /v1/users/:id`~~ | **Partly closed.** Same vetting, applied here rather than relying on the dead `safeBody` strip. |
| ~~`POST /v1/usersTemp`~~ | **Closed.** The mount is pinned to the one role it legitimately assigns (`rpms-member`) by `restrictProvisioningRole`. All 268 `userType: rpms-member` accounts hold exactly that role, so no real caller is affected. |

**What is still open on those two is authentication, not authorisation.** Any
authenticated user can still call them and change another user's roles — they
simply can no longer grant super-admin or take an occupied office. Adding a
caller guard is a breaking change and is left alone on instruction. They should be closed before enforcement goes to
`enforce`, ideally by logging callers first to see who would break.

**`/v1/usersTemp` is still wider than it looks.** It mounts the whole
`UsersRouter` unauthenticated, so beyond user creation an anonymous caller can
also list and read every user, search them, delete one, and change any user's
password or roles — the routes carrying their own super-admin guards
(`impersonate`, `/:id/status`) are the only ones that refuse. The role
restriction closes the escalation to `super-admin`; it does not narrow the mount.
The migration path already noted in `routes/index.ts` — the scoped
"Backend User Provisioning" API key limited to POST and DELETE — is what actually
fixes this.

**Also not done:** `requireDbRole` is untouched, so an office role still passes
its guards regardless of the register. Only `switch-role` consults it. Widening
that changes who can reach every guarded endpoint and needs its own decision.
