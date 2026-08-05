# ALSC SSO

Centralized authentication infrastructure for Asian Land Strategies Corp., using Keycloak.

This repo contains the identity/SSO infrastructure only — it is separate from any
individual application's codebase. Each app (ERP, Booking, Vehicle Sticker, CAR, GHP)
integrates with this Keycloak instance via OIDC, but its integration code lives in
that app's own repository, not here.

## Structure

```
alsc-sso/
├── docker-compose.yml       Keycloak + Postgres
├── .env.example             Environment variable template (copy to .env, fill in real values)
├── nginx/
│   └── keycloak.conf        Reverse proxy config for the VPS
├── realm-export/            Keycloak realm backups (export after any config change)
├── docs/
│   ├── SSO_VPS_Implementation_Plan.md
│   └── role-matrix.md       Group -> role -> system access matrix
└── README.md
```

## Setup (on the VPS)

```bash
cp .env.example .env
# edit .env with real values

docker compose up -d
```

Then set up Nginx (copy nginx/keycloak.conf into sites-available), point DNS at the
VPS, and run certbot for TLS:

```bash
sudo certbot --nginx -d sso.asianland.ph
```

Access the admin console at `https://sso.asianland.ph` and log in with the
`ADMIN_USER` / `ADMIN_PASSWORD` from `.env`.

## After first setup

1. Create the `ALSC` realm (do not use `master` for real users)
2. Create Clients for each app (ERP, Booking, Vehicle Sticker, CAR, GHP)
3. Create Client Roles per app, per `docs/role-matrix.md`
4. Create Groups and map roles to them
5. Set `/Employee` as the default group
6. Export the realm config into `realm-export/` for backup:
   `docker exec <container> /opt/keycloak/bin/kc.sh export --dir /opt/keycloak/data/import --realm ALSC`

## Backups

Back up regularly:
- `keycloak_db_data` Docker volume (Postgres data)
- `realm-export/` JSON exports after any role/group/client changes
