# ALSC SSO — Role Matrix

Fill this in before rollout. This becomes the blueprint for Keycloak Group and Role configuration.

## Groups → Client Role Mapping

| Group | ERP | Booking | Vehicle Sticker | CAR | GHP |
|---|---|---|---|---|---|
| /Employee (default) | — | — | — | — | — |
| /IT-Admin | admin | admin | admin | admin | admin |
| /Accounting | erp-accounting | booking-viewer | — | — | — |
| /Front-Desk | — | booking-manager | — | — | — |
| /Engineering | erp-engineering | — | vehicle-sticker-viewer | — | — |
| /HOA-Officers | — | — | vehicle-sticker-manager | car-manager | — |

*(Replace with your actual department names, roles, and system access needs.)*

## Client Roles Reference (per app)

Define these inside each Keycloak Client before assigning to groups.

### ERP
- `erp-admin`
- `erp-accounting`
- `erp-engineering`
- `erp-readonly`

### Booking
- `booking-admin`
- `booking-manager`
- `booking-viewer`

### Vehicle Sticker
- `vehicle-sticker-admin`
- `vehicle-sticker-manager`
- `vehicle-sticker-viewer`

### CAR
- `car-admin`
- `car-manager`
- `car-viewer`

### GHP
- `ghp-admin`
- `ghp-viewer`

## Offboarding checklist

- [ ] Disable user in Keycloak (do not delete immediately — retain for audit trail)
- [ ] Confirm no app-local admin accounts exist outside Keycloak for this user
- [ ] Log disablement date/reason in audit notes
