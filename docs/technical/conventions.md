# Conventions & Guardrails

Rules and design choices that govern tenant isolation, resource scoping, roles, and security.

**On this page**

| Section | Description |
|---------|-------------|
| [Tenant Isolation Rules](#tenant-isolation-rules) | Strict isolation, queryset filtering |
| [GLOBAL vs TENANT Resource Rules](#global-vs-tenant-resource-rules) | Platform vs tenant data |
| [Role Boundaries](#role-boundaries) | SUPER_ADMIN, ADMIN, LISTENER |
| [Security Decisions](#security-decisions) | No tenant data for super admin, etc. |
| [Design Philosophy](#design-philosophy) | Same endpoint, fail closed, soft deletes |

---

## Tenant Isolation Rules

- **Strict isolation:** An ADMIN can **never** access data belonging to another tenant.
- **Tenant context:** Tenant is determined by the authenticated user (`user.tenant`). LISTENER and ADMIN always have a tenant; SUPER_ADMIN does not.
- **URL-based tenant in auth:** Tenant-scoped auth endpoints include `tenant_id` in the path (`/api/v1/tenant/{tenant_id}/auth/...`). The backend validates that the tenant exists and is active.
- **Queryset filtering:** Tenant-scoped resources are filtered by `tenant` (or `user.tenant`) in views. Use tenant-aware managers where applicable (e.g. `User.objects` vs `User.all_users`).
- **No cross-tenant queries:** Avoid any query that could return rows from multiple tenants for tenant-scoped resources.

---

## GLOBAL vs TENANT Resource Rules

### GLOBAL Resources

- **Definition:** Platform-level data shared across tenants or used only by platform administration.
- **Examples:** GLOBAL songs, genres, tenants, platform-wide subscription/payment lists.
- **Who manages:** SUPER_ADMIN only (except tenant detail GET, which ADMIN can do for own tenant).
- **Who can access:** Varies by resource (e.g. ADMIN can read GLOBAL songs and genres; LISTENER cannot access genres).

### TENANT Resources

- **Definition:** Data that belongs to a single tenant.
- **Examples:** TENANT songs, tenant-song links, playlists, song requests, tenant users.
- **Who manages:** ADMIN (within own tenant); LISTENER (own data only).
- **Who can access:** ADMIN for full tenant scope; LISTENER for own scope only. SUPER_ADMIN has **no** access to tenant resources.

### Song Visibility

- **`visibility` field:** `GLOBAL` or `TENANT`.
- **GLOBAL songs:** Created and managed by SUPER_ADMIN. Can be linked to tenants via tenant-song links by ADMIN.
- **TENANT songs:** Created and managed by tenant ADMIN. Visible only within that tenant.
- **Enforcement:** Implemented in song views and permissions (role-based filtering).

---

## Role Boundaries

### SUPER_ADMIN

**Scope:**

- Platform only
- No tenant

**Can:**

- Manage tenants
- Manage GLOBAL songs
- Manage genres
- Create ADMINs
- View platform metrics (user counts, subscriptions, payments)
- List all admins

**Cannot:**

- Access any tenant-specific data (users, playlists, song requests, tenant-song links, TENANT songs)
- Access PII

### ADMIN

**Scope:**

- Single tenant (`user.tenant`)

**Can:**

- Full CRUD on tenant users (except delete self or other admins)
- Full CRUD on TENANT songs
- Full CRUD on tenant-song links
- Full CRUD on playlists
- Full CRUD on song requests
- Create payment links
- View tenant subscription
- View tenant details

**Cannot:**

- Access other tenants
- Manage GLOBAL songs
- Manage genres
- Delete self
- Delete other ADMIN/SUPER_ADMIN users

### LISTENER

**Scope:**

- Own data only within their tenant

**Can:**

- Manage profile
- Manage own playlists
- Manage own song requests
- View approved tenant songs via `/api/v1/tenant/songs/`
- Self-delete

**Cannot:**

- Access `/api/v1/songs/` directly
- Access genres
- Manage other users
- Access other users’ playlists

---

## Security Decisions

### SUPER_ADMIN and Tenant Data

- **Decision:** SUPER_ADMIN has **zero** access to tenant data.
- **Rationale:** Reduces attack surface and enforces separation between platform operations and tenant data. Platform admins manage configuration and global content, not tenant PII or content.
- **Enforcement:** Permission classes (e.g. `IsTenantUser`, `IsOwnerOrAdmin`) and view logic explicitly block SUPER_ADMIN from tenant-scoped endpoints.

### Admin Self-Deletion

- **Decision:** Admins cannot delete their own accounts (via any endpoint).
- **Rationale:** Prevents lockout and accidental loss of tenant administration.
- **Enforcement:** User deletion logic checks that the target is not the requesting user and that the target is a LISTENER.

### Deletion of Other Admins

- **Decision:** Only LISTENER users can be deleted by an ADMIN.
- **Rationale:** Keeps admin lifecycle under control (e.g. via SUPER_ADMIN or out-of-band processes).
- **Enforcement:** Admin delete endpoints restrict targets to LISTENER.

### User Creation and Verification

- **Decision:** LISTENER self-registers; ADMIN created by SUPER_ADMIN; ADMIN creates LISTENER. All created users must verify email before login.
- **Rationale:** Ensures valid, reachable identities and controlled admin provisioning.
- **Enforcement:** Registration and user-creation flows; `is_verified` and email verification checks at login.

### Password Change and Sessions

- **Decision:** Changing password blacklists all access and refresh tokens.
- **Rationale:** Forces re-login everywhere after a password change, limiting exposure if a password was compromised.
- **Enforcement:** Password-change handler invalidates tokens accordingly.

### Genre and Tenant Deletion

- **Genre:** Cannot delete a genre that has any linked (non–soft-deleted) songs.
- **Tenant:** Cannot delete a tenant that has any active users.
- **Enforcement:** Implemented in model delete logic or view logic, returning clear errors when constraints are violated.

---

## Design Philosophy

- **Same resource, same endpoint:** One set of URLs per resource. Access differences are enforced by permissions and queryset filtering, not by separate role-specific routes.
- **Explicit scope:** Always filter by tenant (or ownership) in tenant-scoped APIs. Avoid implicit or optional tenant context.
- **Fail closed:** Unclear or missing permission leads to denial (403), not grant.
- **Soft deletes:** Deletions are soft where applicable (`deleted_at`). Supports restore and audit; hard delete only when necessary and safe.
- **Object-level checks:** For ownership-sensitive resources (e.g. playlists), use object-level permissions (e.g. `IsOwnerOrAdmin`) in addition to view-level checks.
