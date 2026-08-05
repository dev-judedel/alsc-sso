# SSO on VPS — Implementation Plan
**ALSC IT Department | Keycloak Single Sign-On Rollout**

---

## 1. Objective

Implement centralized authentication (Single Sign-On) across ALSC's internal systems using **Keycloak**, hosted on a dedicated VPS, while existing apps remain on their current hosting (shared hosting / current servers). This removes the need for employees to manage separate logins/passwords per system and gives IT a central point to manage access.

---

## 2. Systems in Scope

| System | Stack | Current Hosting | OIDC Integration Effort |
|---|---|---|---|
| ERP | PHP native | Shared/existing | Higher (manual OIDC library) |
| Vehicle Sticker System | Laravel | Shared/existing | Lower (Socialite package) |
| CAR | PHP native | Shared/existing | Higher (manual OIDC library) |
| GHP | Laravel | Shared/existing | Lower (Socialite package) |
| Booking System | Laravel | Shared/existing | Lower (Socialite package) |

**Scale:** ~50 public-facing users + 50–70 ERP users (~120 total accounts)

---

## 3. Architecture Decision: SSO on VPS, Apps Stay on Shared Hosting

**Decision:** Keycloak runs on a small, dedicated VPS. All existing systems remain on their current hosting (Hostinger shared hosting and/or existing servers). No app migration required.

### Why this works
- Keycloak and apps communicate over standard HTTPS (OIDC redirect flow) — they do not need to share a server or provider.
- Login flow: App redirects browser to Keycloak → user logs in once → Keycloak issues a token → app verifies token via server-to-server call.
- This is a standard, widely-used architecture pattern — identity providers are commonly hosted separately from applications.

### Why Keycloak must be isolated (not bundled with app servers)
- If Keycloak shares a server with an app and that app crashes or consumes resources, **login breaks for every connected system**, not just that one app.
- Isolating Keycloak on its own VPS means an app-side issue never takes down authentication for the rest of the company.

### Requirements on the shared hosting / app side
- Outbound HTTPS requests must be allowed (for token verification calls to Keycloak)
- Composer support (for `socialiteproviders/keycloak` on Laravel apps)
- Valid SSL on all apps (already standard on Hostinger)
- Negligible latency impact at this scale (120 users)

---

## 4. Infrastructure Plan

### Tiering decision
| Tier | Description | Est. Monthly Cost | Verdict |
|---|---|---|---|
| One droplet per system | Maximum isolation, highest overhead | ~$36–72/mo | Not justified at current scale |
| **Keycloak isolated, apps grouped/unchanged** | Login isolated; apps stay on shared hosting | **~$6–12/mo (VPS only)** | **Recommended** |
| Everything on one droplet | Cheapest, but single point of failure for all systems | ~$12–24/mo | Pilot/testing only, not production |

### Recommended VPS specs (Keycloak + its database)
| Resource | Spec |
|---|---|
| vCPU | 2 |
| RAM | 2 GB |
| Storage | 25–40 GB |
| Database | Bundled on same VPS (Postgres, via Docker) — sufficient at 120-user scale |

*A separate database server is not needed at this scale. Revisit only if this becomes part of a broader, company-wide database consolidation initiative.*

### Provider decision: DigitalOcean vs Hostinger VPS

| Factor | DigitalOcean | Hostinger VPS |
|---|---|---|
| Billing | Month-to-month, cancel anytime | 12–24 month lock-in |
| Renewal pricing | Flat, no jump | Often 2–3x increase at renewal |
| Team familiarity | Already used for BrewPOS | New environment |
| Setup cost | Reuse existing skills/scripts | Learning curve |

**Recommendation:** New (or existing) DigitalOcean droplet, ~$6–12/month, dedicated to Keycloak.

---

## 5. Access Control Model (RBAC via Keycloak — Native, Lean Approach)

**Decision:** For the pilot, use Keycloak's built-in Groups + Roles UI directly. No custom ERP access-management module for now — revisit only if the volume of access changes or approval-workflow needs outgrow the native console.

### Core concepts

- **Realm roles** — apply across all systems (e.g. `employee`, `it-admin`)
- **Client roles** — specific to one app, defined per client (e.g. `erp-readonly`, `booking-manager`)
- **Groups** — bundle client/realm roles together and map to how the business actually thinks about access (by department/function), so IT manages *groups*, not individual role assignments per person
- Each app still enforces its own permission logic based on the roles present in the token Keycloak issues — Keycloak controls *who* and *what roles*, the app decides what those roles are allowed to click/see
- A user only sees/accesses an app if they're assigned a role for that app's Keycloak client — otherwise access is blocked at the app level

### Suggested starter group structure

| Group | Client roles assigned | Applies to |
|---|---|---|
| `/Employee` (default, auto-assigned) | `employee` (realm role) | Everyone |
| `/IT-Admin` | Full admin role on all clients | IT department |
| `/Accounting` | `erp-accounting`, `booking-viewer` | Accounting staff |
| `/Front-Desk` | `booking-manager` | Front desk / reception |
| `/Engineering` | `erp-engineering`, `vehicle-sticker-viewer` | Engineering team |
| `/HOA-Officers` | `car-manager`, `vehicle-sticker-manager` | HOA officers |

*(Fill in actual department names/roles as the real matrix is finalized.)*

### Setup steps in Keycloak console

1. Create **Client Roles** first, inside each client (ERP, Booking, Vehicle Sticker, CAR, GHP) — these are the roles the app's code actually checks for.
2. Create **Groups** matching departments/functions.
3. Under each Group's **Role Mapping** tab, attach the relevant client roles.
4. Add users to the appropriate group(s) — this becomes the day-to-day operation for onboarding.
5. Set `/Employee` as the **default group** so new users automatically get baseline access.

### Offboarding benefit

Disabling a user's single Keycloak account immediately revokes access across all connected systems — no need to deactivate accounts in 5 separate apps.

**Action item:** Finalize the real department → role → system matrix before rollout using the starter table above as a template.

---

## 6. How SSO Actually Works Across Systems (Step-by-Step)

This section explains the technical login flow in plain terms — useful for explaining to non-technical stakeholders or onboarding a new dev to the pattern.

### The key idea
None of the apps store passwords anymore. Keycloak is the only place a password ever gets typed. Apps and Keycloak communicate using a protocol called **OIDC (OpenID Connect)** — a standardized way for an app to say "I don't know who this is, please verify" and for Keycloak to reply "here's proof of who they are, and what they're allowed to do."

### Step-by-step flow (first login of the day)

1. **User opens an app** (e.g. ERP). ERP checks: is there a valid session/token? No → redirect the browser to Keycloak's login page.
2. **User logs in at Keycloak** — one username, one password, entered only here.
3. **Keycloak verifies credentials** against its own user database, and checks group/role membership.
4. **Keycloak issues an ID token + access token** — signed data confirming who the user is and what roles/groups they belong to — and redirects the browser back to ERP with these tokens attached.
5. **ERP receives the token**, verifies its signature came from Keycloak (not forged), and reads the roles inside it.
6. **ERP creates a local session** for the user based on the token's identity and roles, and the user is now logged in.
7. **Keycloak also sets its own session cookie** in the browser, separate from ERP's session — this is what enables the "no re-login" effect for the next app.

### Step-by-step flow (opening a second app afterward)

1. **User opens Booking system.** Booking checks: no local session yet → redirect to Keycloak.
2. **Keycloak sees the existing session cookie** from the earlier login — recognizes the user is already authenticated.
3. **No login form is shown.** Keycloak immediately issues a new token (scoped for Booking this time) and redirects back.
4. **Booking system verifies the token**, creates its own local session, user is in — with zero typing required.

### What each app needs to implement (once, per app)

- A **redirect handler** — sends unauthenticated users to Keycloak's login URL
- A **callback handler** — receives the token after Keycloak redirects back, verifies it, and creates a local session
- **Role/permission mapping logic** — translates the roles inside the token into what the app actually allows the user to do (Keycloak doesn't dictate UI/permissions, only identity + role facts)
- For Laravel apps: `socialiteproviders/keycloak` handles most of this out of the box
- For native PHP apps (ERP, CAR): a library like `jumbojett/openid-connect-php` handles the OIDC handshake manually

### What happens on logout

Logging out can be scoped two ways:
- **Local logout** — only ends the session in that one app; Keycloak session and other apps stay logged in
- **Global logout (Single Logout / SLO)** — ends the Keycloak session too, which signals all connected apps to also end their sessions on next check. Worth deciding early whether "logout" should mean "log out of everything" for your use case (likely yes, for shared/kiosk-style front-desk machines).

### What happens on token expiry

Tokens are short-lived by design (commonly 5–15 minutes for access tokens). Apps use a **refresh token** behind the scenes to silently get a new one without bothering the user, as long as the Keycloak session is still valid — this is invisible to the user in normal use.

---

## 7. Rollout Phases

### Phase 1 — Infrastructure Setup (~2–3 days)
- Provision VPS (DigitalOcean)
- Install Docker, deploy Keycloak + Postgres via `docker-compose.yml`
- Configure Nginx reverse proxy + TLS (Certbot)
- Create ALSC Realm, admin access, base configuration

### Phase 2 — Pilot Integration (~3–5 days)
- Choose one Laravel app as pilot (recommended: GHP or Vehicle Sticker — lower integration effort)
- Integrate via `socialiteproviders/keycloak`
- Build and test role mapping for pilot app
- Validate login/logout flow, session behavior, token expiry

### Phase 3 — Expand to Remaining Laravel Apps (~1–2 days per app)
- Booking System
- Second Laravel app (whichever wasn't the pilot)
- Reuse integration pattern from Phase 2

### Phase 4 — Native PHP Apps (ERP, CAR) (~5–7 days first app, ~2–3 days second)
- Build reusable OIDC handling library using `jumbojett/openid-connect-php`
- Integrate ERP and CAR using the shared pattern
- ERP should be scheduled last, given its criticality — pattern proven on lower-stakes systems first

### Phase 5 — Rollout & Cutover
- User account creation/import into Keycloak
- Role assignment per employee
- Branded login page (ALSC logo/theme)
- Employee communication and go-live

---

## 8. Business Case Summary (for management/CFO)

- **Problem:** Multiple standalone systems each require separate logins — creates password fatigue, slower onboarding/offboarding, and no centralized way to revoke access if an account is compromised.
- **Solution:** Centralized login (SSO) via Keycloak, hosted on a small, dedicated, low-cost VPS.
- **Cost:** ~$6–12/month — comparable to a rounding error against existing infra spend (e.g. current Business WordPress hosting renews at ~PHP 5,268/year).
- **Risk reduction:** Centralized identity management reduces exposure from reused/weak passwords and simplifies access revocation.
- **Scope discipline:** This is additive infrastructure — existing hosting (shared hosting, current app servers) is untouched. No migration risk to current systems or the public website.
- **Approach:** Phased pilot (starting with 1–2 lower-stakes Laravel systems) before expanding to ERP — low risk, reversible if it doesn't pan out.

---

## 9. Open Items / Decisions Needed

- [ ] Confirm final VPS provider and plan (DigitalOcean recommended)
- [ ] Confirm pilot app (GHP vs Vehicle Sticker)
- [ ] Build full RBAC/role matrix per system
- [ ] Decide domain for Keycloak (e.g. `sso.asianland.ph` or similar)
- [ ] CFO/mancom approval for VPS budget line
- [ ] Determine account migration strategy for existing users (fresh accounts vs. imported credentials)
