# 2026-09-23 05:40 — Snapshot restore, nat-support erase, and the docs move

**Branch:** `dev` · **Verified:** 1199 passing

---

## Behaviour changes on deploy

| Change | Effect |
|---|---|
| **National support may erase a user** | `DELETE /v1/users/:id?hard=true` now works for elevated roles, not just super-admin |
| Everyone else sending `hard=true` | **unchanged** — still gets the soft delete they are entitled to, not an error |

---

## 1. National support may permanently delete a user

They could not delete a user **at all** before this — `canDeleteUser` only
short-circuited for `isSuperAdmin`, so a nat-support caller with no units hit
`NO_STANDING` long before the hard-delete branch.

`canDeleteUser` now admits `isElevated`, returning `elevated:nat-support` as the
authority, and the hard-delete gate reads that verdict rather than deriving a
second rule that could disagree with it.

**What this crosses, stated plainly:** `ELEVATED_ROLES` is documented as widening
the hierarchy, principal-officer, HQ and approval surfaces *while leaving account
administration to super-admin alone*. Deleting an account is account
administration. This is a deliberate exception, asked for because support field
the requests — not the rule extending naturally.

Still bounded by the two rules that bind every non-super-admin: an account
holding a **protected or sensitive role is untouchable**, and you cannot delete
yourself.

```http
DELETE /v1/users/6520a10000000000000000bb?hard=true
Authorization: Bearer <natSupportToken>
```

A scoped administrator sending `hard=true` continues to receive the **soft**
delete — long-standing, deliberate, and tested. They are entitled to remove the
account, it still ends up unusable, and the audit line records that erasure was
requested and withheld.

## 2. Restore from a deletion snapshot

`POST /v1/deletions/:id/restore` — **super-admin only.**

The route of last resort, for a record that was *erased*. A soft-deleted record
goes back through the restore route on its own collection, which is cheaper and
safer.

```http
POST /v1/deletions/652f1a0000000000000000dd/restore
Authorization: Bearer <superAdminToken>
```

**200**

```json
{
  "restored": true,
  "module": "PARISH_DIRECTORY",
  "collection": "parishDirectory",
  "recordId": "652f1a0000000000000000aa",
  "restoredBy": "the.superadmin",
  "references": {
    "unreconciled": true,
    "parishCode": "211003",
    "usersOnThisParish": 42,
    "note": "Those users kept pointing at this parish code the whole time it was gone..."
  }
}
```

### What it does not do, and says so

Re-inserting the document **does not put the world back**. A parish code is
referenced by users, offices and the hierarchy, so a blind re-insert produces a
row half the system still believes is gone. The component said this from the
start and it is still true.

So the response carries `references.unreconciled: true` and the counts. You are
told what was restored and what still needs a human, rather than being left to
assume the system is whole again.

For a **user**, `references.credentialRestored` is `false`: the password and OTP
secret were stripped when the snapshot was taken and are not resurrected. The
account cannot sign in until a password is set — said in the response rather
than discovered at the next login attempt.

**400** — no snapshot on that tombstone, or a live row already holds that id.
Restoring over something is not a restore.

## 3. Documentation moved to `documentations/v1/`

All 28 root markdown files now live in `documentations/v1/`. The root has no
loose `.md` left. Everything stays **tracked** — moving them into an ignored
folder would have removed the whole reference set from the repository and from
the open PR, which is the opposite of the intent.

Every cross-reference was rewritten and checked: doc-to-doc links, the dated
notes pointing into `v1/`, and the source paths inside the moved files, which
are now two levels deeper. A link checker over the whole tree reports none
broken.

`DEPLOYMENT_GUIDE.md` moved too and stays gitignored — the pattern has no
leading slash, so it follows the file.

## 4. Environment variables are in `.env.example`

All 21 new variables, with their defaults and what each one costs:
`PRIVILEGED_AUDIT_*`, `HIERARCHY_SYNC_*`, `PARISH_REACTIVATION_*`.

They still need registering in `deployment-scripts/lib.sh` to reach ECS — that
directory is gitignored and not reachable from here.

---

## Correction to the 2026-09-22 note

That note said `parishDirectory`'s index problem was an `IndexOptionsConflict`
(code 85). **Reproduced locally, and it is not.**

The schema declares `unique: "Duplicate Parish Code ({VALUE})"` — a *string*
where MongoDB wants a boolean — so `ensureIndexes` is rejected with
**`TypeMismatch` (code 14)** and the schema-declared index never builds at all.
With `MONGO_AUTO_INDEX` defaulting to true and nothing listening on the index
event, it fails silently on every boot.

That is why the live index had to be created by hand as `parish_code_unique`,
and it is the same class of fault `Users/model.ts:500-519` documents. The fix is
the plugin's string form, not the index. The earlier note has been corrected.

---

## Known gaps

- Snapshot restore **reports** references; it does not reconcile them. Deliberate
  — auto-repointing from a months-old snapshot needs its own dry-run design.
- Nothing stops a super-admin approving a parish disable they raised themselves.
  Left as agreed.
- `/deleted` and `/restore` remain elevated-only rather than province-scoped.
  `/restore` staying elevated-only is deliberate.
