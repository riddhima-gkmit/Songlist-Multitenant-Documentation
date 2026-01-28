# Core Modules

Purpose, access rules, data scope, and business rules per module. Endpoints: [API Design](api-design.md).

**On this page**

| Module | Description |
|--------|-------------|
| [Users](#users) | Auth, profile, tenant user admin |
| [Tenants](#tenants) | Organizations, SUPER_ADMIN management |
| [Songs](#songs) | GLOBAL vs TENANT catalog |
| [Tenant-Song Links](#tenant-song-links) | Link songs to tenants |
| [Playlists](#playlists) | User playlists, ADMIN management |
| [Song Requests](#song-requests) | Request, review, fulfill |
| [Genres](#genres) | Platform-wide categories |
| [Payments](#payments) | Razorpay, subscriptions |

---

## Users

**Purpose:** Authentication, profile management, and tenant user administration.

**Who can access:**
- **LISTENER:** Own profile only; self-delete allowed.
- **ADMIN:** Tenant users (full CRUD); delete/restore LISTENER only; cannot delete self or other admins.
- **SUPER_ADMIN:** User count only (no PII); create ADMIN for any tenant; list all admins.

**Data scope:** Own (LISTENER), tenant (ADMIN), platform count / admin list (SUPER_ADMIN).

**Business rules:**
- Email unique per tenant (LISTENER/ADMIN); globally unique for SUPER_ADMIN.
- LISTENER/ADMIN must have a tenant; SUPER_ADMIN must not.
- All created users must verify email before login.
- Password change blacklists all access and refresh tokens.

---

## Tenants

**Purpose:** Multi-tenant organizations. Managed by SUPER_ADMIN; ADMIN can view their tenant only.

**Who can access:**
- **LISTENER:** No access.
- **ADMIN:** GET tenant details only (own tenant).
- **SUPER_ADMIN:** Full CRUD, activate, deactivate.

**Data scope:** Global (platform-level resource).

**Business rules:**
- Tenant cannot be deleted if it has any active users.
- Soft delete; unique name among non-deleted tenants.

---

## Songs

**Purpose:** Song catalog with GLOBAL vs TENANT visibility. GLOBAL = platform; TENANT = tenant-specific.

**Who can access:**
- **LISTENER:** No direct access; use `/api/v1/tenant/songs/` for approved tenant songs.
- **ADMIN:** TENANT songs in own tenant (and read GLOBAL); create/update/delete TENANT songs.
- **SUPER_ADMIN:** GLOBAL songs only; create/update/delete GLOBAL songs.

**Data scope:** GLOBAL (platform) or TENANT (per tenant).

**Business rules:**
- `visibility` is GLOBAL or TENANT.
- Release year validated (min year, not in future).
- Songs link to genre (required); genre protected from deletion when linked.

---

## Tenant-Song Links

**Purpose:** Link songs to tenants. Controls which songs a tenant can use; admins manage links.

**Who can access:**
- **LISTENER:** List/detail approved tenant songs only.
- **ADMIN:** Full CRUD on own tenant’s links; bulk add/delete.
- **SUPER_ADMIN:** No access.

**Data scope:** Tenant-only.

**Business rules:**
- Unique song per tenant (one link per song per tenant).
- `is_active` flags link state; soft delete on links.

---

## Playlists

**Purpose:** User-defined song collections. Owned by users; admins can manage all playlists in their tenant.

**Who can access:**
- **LISTENER:** Own playlists only.
- **ADMIN:** All playlists in tenant; can create on behalf of users (`user_id`).
- **SUPER_ADMIN:** No access.

**Data scope:** Tenant (via playlist owner’s tenant).

**Business rules:**
- Playlist name unique per user (among non-deleted).
- Only approved tenant songs can be added (enforced where used).
- Object-level permission: owner or ADMIN in same tenant.

---

## Song Requests

**Purpose:** Users request songs; admins review and fulfill (approve/reject, link or create song).

**Who can access:**
- **LISTENER:** Own requests only.
- **ADMIN:** All requests in tenant; create for any tenant user; review and fulfill.
- **SUPER_ADMIN:** No access.

**Data scope:** Tenant-only.

**Business rules:**
- Status flow: PENDING → APPROVED | REJECTED | FULFILLED.
- Unique `(song_title, tenant)` per request (model constraint).
- ADMIN can create requests with `user_id` for any tenant user.

---

## Genres

**Purpose:** Platform-wide song categorization. SUPER_ADMIN manages; ADMIN read-only.

**Who can access:**
- **LISTENER:** No access.
- **ADMIN:** Read-only (list, detail).
- **SUPER_ADMIN:** Full CRUD.

**Data scope:** Global.

**Business rules:**
- Genre cannot be deleted if any non–soft-deleted song is linked.
- Name unique globally.

---

## Payments

**Purpose:** Razorpay premium subscriptions (lifetime). Payment links, webhooks, subscription status.

**Who can access:**
- **LISTENER:** No access.
- **ADMIN:** Create payment link; view tenant subscription status.
- **SUPER_ADMIN:** List all subscriptions and payments; no payment-link creation.

**Data scope:** Tenant (links, subscription); platform (SUPER_ADMIN lists).

**Business rules:**
- Payment state machine: CREATED → PENDING → PAID → VERIFIED → ACTIVATED (or FAILED).
- Webhook signature verification required.
- Premium stored on tenant’s `Subscription` (e.g. `is_premium`).
- Idempotent processing for webhooks where applicable.
