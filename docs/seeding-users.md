# Seeding Realm Data & Users

Two files in `realm-export/` work together like a database seeder:

| File | Purpose | Run |
|---|---|---|
| `alsc-realm-seed.json` | Structure: realm roles, client roles, groups (+ role mappings), `/Employee` default group, the 5 app clients, and an `employee_code` token claim mapper on each client | **Once**, when the realm is created |
| `users-seed-template.json` | Batch of user accounts (username, email, group, `employee_code` attribute, temp password) | **Repeatedly**, any time you onboard new hires |

## 1. Initial realm bootstrap (one-time)

Admin Console → realm dropdown (top-left) → **Create Realm** → **Browse** → select
`realm-export/alsc-realm-seed.json` → **Create**.

This creates the `ALSC` realm with roles, groups, and the 5 client apps (ERP, Booking,
Vehicle Sticker, CAR, GHP) already wired per `docs/role-matrix.md`.

**Before go-live**, for each client under **Clients**:
- Replace the placeholder `redirectUris` (currently `https://REPLACE-WITH-<APP>-DOMAIN/*`) with the real app URL
- Copy the auto-generated client secret from the **Credentials** tab into that app's `.env`

## 2. Onboarding users (repeatable)

Duplicate a user block in `users-seed-template.json`, fill in real values, remove the
`_comment` key and any unused example entries, then:

Realm (`ALSC`) → **Realm Settings** → **Action** menu (top-right) → **Partial import** →
upload the file → set **"If a resource exists"** to **Skip** → **Import**.

Using Partial Import (not a full realm re-import) means re-running the file is safe —
existing users are skipped, not overwritten or duplicated.

Each seeded user:
- Belongs to one or more groups (`/Accounting`, `/IT-Admin`, etc.) — group membership is what
  grants app access, not individual role assignment
- Has `attributes.employee_code` set — this is your internal HR/employee ID
- Gets a temporary password and `UPDATE_PASSWORD` required action, so they set their own
  password on first login

## `employee_code` in the token

Every client already has an `employee_code` protocol mapper (added by
`alsc-realm-seed.json`), so any app the user logs into receives it as a claim in the
ID token / access token / userinfo response — apps can use it to match the SSO identity
to their local employee records without an extra lookup.

## Optional: show `employee_code` as a labeled field in Admin Console

The seed files set the attribute correctly regardless, but by default Keycloak's user
edit form only shows the field once it's declared. To add it (one-time, ~30 seconds):
Realm Settings → **User profile** → **Create attribute** → name `employee_code` → Save.
