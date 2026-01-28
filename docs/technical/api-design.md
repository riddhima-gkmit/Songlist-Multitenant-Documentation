# API Design

The SongList API follows **RESTful principles** with a single base URL. All responses are **JSON**. Protected endpoints require **JWT Bearer** authentication. Access is controlled by **permissions** and **queryset filtering**, not by role-specific URLs.

**On this page**

| Section | Description |
|---------|-------------|
| [Base Configuration](#base-configuration) | Base URL, auth |
| [API Grouping](#api-grouping) | Auth, users, tenants, songs, etc. |
| [Data Scope Rules](#data-scope-rules) | Own, tenant, global |
| [Error Handling](#error-handling) | 401, 403, 404, etc. |
| [Pagination](#pagination) | Page, page_size |

---

## Base Configuration

- **Base URL:** `http://localhost:8000/api/v1`
- **Authentication:** JWT (Access & Refresh Tokens)
- **Core Rule:** Same resource → same endpoint; different access → permissions + queryset filtering

No role-specific or admin-specific URLs are used for resources.

---

## REST Conventions

- **Resource-Based Endpoints:** CRUD operations on nouns (users, songs, playlists, etc.)
- **HTTP Methods:** GET (list/retrieve), POST (create), PATCH (update), DELETE (delete)
- **Ids:** UUIDs for all primary keys
- **Pagination:** List endpoints support pagination (`page`, `page_size`)

---

## API Grouping

### 1. Authentication APIs

| Method | Endpoint | Access | Notes |
|--------|----------|--------|-------|
| POST | `/api/v1/tenant/{tenant_id}/auth/register/` | LISTENER only | Self-deleted restoration |
| POST | `/api/v1/tenant/{tenant_id}/auth/verify-email/` | All | Verify OTP |
| POST | `/api/v1/tenant/{tenant_id}/auth/resend-verification/` | All | Resend OTP |
| POST | `/api/v1/tenant/{tenant_id}/auth/login/` | All | Step 1: Request OTP |
| POST | `/api/v1/tenant/{tenant_id}/auth/login/verify-otp/` | All | Step 2: Verify OTP, get JWT |
| POST | `/api/v1/auth/super-admin/login/` | SUPER_ADMIN | Step 1 |
| POST | `/api/v1/auth/super-admin/login/verify-otp/` | SUPER_ADMIN | Step 2 |
| POST | `/api/v1/auth/logout/` | Authenticated | Blacklist tokens |
| POST | `/api/v1/token/refresh/` | All | Refresh access token |
| POST | `/api/v1/tenant/{tenant_id}/auth/password-reset/` | All | Step 1 |
| POST | `/api/v1/tenant/{tenant_id}/auth/password-reset/confirm/` | All | Step 2 |

*Permissions:* AllowAny for auth flows; IsAuthenticated for logout.

---

### 2. User Management APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Data Scope |
|--------|----------|:--------:|:-----:|:------------:|------------|
| GET | `/api/v1/users/me/` | ✅ | ✅ | ✅ | Own profile |
| PATCH | `/api/v1/users/me/` | ✅ | ✅ | ✅ | Own profile |
| DELETE | `/api/v1/users/me/` | ✅ | ❌ | ❌ | Self-delete |
| POST | `/api/v1/users/me/change-password/` | ✅ | ✅ | ✅ | Own password |
| GET | `/api/v1/users/` | ❌ | ✅ | ✅ | Tenant list / Count only |
| POST | `/api/v1/users/` | ❌ | ✅ | ✅ | Create LISTENER / ADMIN |
| GET | `/api/v1/users/{id}/` | ❌ | ✅ | ✅ | View LISTENER / ADMIN |
| PATCH | `/api/v1/users/{id}/` | ❌ | ✅ | ✅ | Update LISTENER / ADMIN |
| DELETE | `/api/v1/users/{id}/` | ❌ | ✅ | ✅ | Delete LISTENER / ADMIN |

*Query params (GET /users/):* `page`, `page_size`, `is_active`, `name`, `email`.

**User Deletion & Restoration:**

| Method | Endpoint | Access | Notes |
|--------|----------|--------|-------|
| POST | `/api/v1/users/{user_id}/restore/` | ADMIN / SUPER ADMIN | Body: `{"restore_data": true\|false}` |
| GET | `/api/v1/users/deleted/` | ADMIN / SUPER ADMIN | ADMIN: tenant deleted users; SUPER_ADMIN: deleted admins |

*Query params (GET /users/deleted/):* `page`, `page_size`, `deletion_type` (`self` \| `admin` \| `super admin`).

*Data filtering:* ADMIN sees tenant users (full details); SUPER_ADMIN sees count only (no PII) for GET /users/.

---

### 3. Tenant Management APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Notes |
|--------|----------|:--------:|:-----:|:------------:|------|
| GET | `/api/v1/tenants/` | ❌ | ❌ | ✅ | List all |
| POST | `/api/v1/tenants/` | ❌ | ❌ | ✅ | Create |
| GET | `/api/v1/tenants/{id}/` | ❌ | ✅ | ✅ | Details (ADMIN read-only) |
| PATCH | `/api/v1/tenants/{id}/` | ❌ | ❌ | ✅ | Update |
| DELETE | `/api/v1/tenants/{id}/` | ❌ | ❌ | ✅ | Delete if no users |
| PATCH | `/api/v1/tenants/{id}/activate/` | ❌ | ❌ | ✅ | Activate |
| PATCH | `/api/v1/tenants/{id}/deactivate/` | ❌ | ❌ | ✅ | Deactivate |
| GET | `/api/v1/super-admin/admins/` | ❌ | ❌ | ✅ | All admins |

*Query params (GET /tenants/):* `page`, `page_size`.

*Query params (GET /super-admin/admins/):* `page`, `page_size`, `is_active`, `name`, `email`, `tenant_id` (comma-separated UUIDs).

*Permissions:* IsSuperAdmin for mutate; IsAdminOrSuperAdmin for GET detail.

---

### 4. Tenant-Song Link APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Data Scope |
|--------|----------|:--------:|:-----:|:------------:|------------|
| GET | `/api/v1/tenant/songs/` | ✅ | ✅ | ❌ | Own tenant |
| POST | `/api/v1/tenant/songs/` | ❌ | ✅ | ❌ | Own tenant |
| GET | `/api/v1/tenant/songs/{id}/` | ✅ | ✅ | ❌ | Own tenant |
| DELETE | `/api/v1/tenant/songs/{id}/` | ❌ | ✅ | ❌ | Own tenant |
| POST | `/api/v1/songs/bulk-add/` | ❌ | ✅ | ❌ | Bulk add to tenant |
| POST | `/api/v1/tenant/songs/bulk-delete/` | ❌ | ✅ | ❌ | Bulk delete links |

*Query params (GET /tenant/songs/):* `page`, `page_size`, `title`, `artist`, `genre`, `album` (all optional filters).

LISTENER sees approved tenant songs only; ADMIN sees all in tenant. SUPER_ADMIN has no access.

---

### 5. Song Management APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Data Scope |
|--------|----------|:--------:|:-----:|:------------:|------------|
| GET | `/api/v1/songs/` | ❌ | ✅ | ✅ | Tenant / GLOBAL |
| POST | `/api/v1/songs/` | ❌ | ✅ | ✅ | TENANT / GLOBAL |
| GET | `/api/v1/songs/{id}/` | ❌ | ✅ | ✅ | Tenant / GLOBAL |
| PATCH | `/api/v1/songs/{id}/` | ❌ | ✅ | ✅ | TENANT / GLOBAL |
| DELETE | `/api/v1/songs/{id}/` | ❌ | ✅ | ✅ | TENANT / GLOBAL |

*Query params (GET /songs/):* `page`, `page_size`, `title`, `artist`, `genre`, `album` (all optional filters).

*Data separation:*
- LISTENER: No direct access; use `/api/v1/tenant/songs/`.
- ADMIN: TENANT songs only (visibility=TENANT, tenant=user's tenant).
- SUPER_ADMIN: GLOBAL songs only (visibility=GLOBAL).

---

### 6. Song Request APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Data Scope |
|--------|----------|:--------:|:-----:|:------------:|------------|
| GET | `/api/v1/song-requests/` | ✅ | ✅ | ❌ | Own / Tenant |
| POST | `/api/v1/song-requests/` | ✅ | ✅ | ❌ | Own / Any user (tenant) |
| GET | `/api/v1/song-requests/{id}/` | ✅ | ✅ | ❌ | Own / Tenant |
| PATCH | `/api/v1/song-requests/{id}/` | ✅ | ✅ | ❌ | Own / Tenant |
| DELETE | `/api/v1/song-requests/{id}/` | ✅ | ✅ | ❌ | Own / Tenant |
| POST | `/api/v1/song-requests/{id}/review/` | ❌ | ✅ | ❌ | Approve/Reject |
| POST | `/api/v1/song-requests/{id}/fulfill/` | ❌ | ✅ | ❌ | Fulfill request |

*Query params (GET /song-requests/):* `page`, `page_size`, `status` (ADMIN only; filter by request status).

*Permission:* IsAuthenticated + IsTenantUser. ADMIN can create requests for any tenant user via `user_id`.

---

### 7. Playlist APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Data Scope |
|--------|----------|:--------:|:-----:|:------------:|------------|
| GET | `/api/v1/playlists/` | ✅ | ✅ | ❌ | Own / Tenant |
| POST | `/api/v1/playlists/` | ✅ | ✅ | ❌ | Own / Any user (tenant) |
| GET | `/api/v1/playlists/{id}/` | ✅ | ✅ | ❌ | Own / Tenant |
| PATCH | `/api/v1/playlists/{id}/` | ✅ | ✅ | ❌ | Own / Tenant |
| DELETE | `/api/v1/playlists/{id}/` | ✅ | ✅ | ❌ | Own / Tenant |
| GET | `/api/v1/playlists/{id}/songs/` | ✅ | ✅ | ❌ | Own / Tenant |
| POST | `/api/v1/playlists/{id}/songs/` | ✅ | ✅ | ❌ | Own / Tenant |
| DELETE | `/api/v1/playlists/{id}/songs/` | ✅ | ✅ | ❌ | Own / Tenant |
| DELETE | `/api/v1/playlists/{playlist_id}/songs/{song_id}/` | ✅ | ✅ | ❌ | Remove song |

*Query params (GET /playlists/):* `page`, `page_size`. *Query params (GET /playlists/{id}/songs/):* `page`, `page_size`.

*Permission:* IsAuthenticated; IsOwnerOrAdmin for detail. SUPER_ADMIN has no access.

---

### 8. Genre APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Notes |
|--------|----------|:--------:|:-----:|:------------:|------|
| GET | `/api/v1/genres/` | ❌ | ✅ | ✅ | Read-only |
| POST | `/api/v1/genres/` | ❌ | ❌ | ✅ | Create |
| GET | `/api/v1/genres/{id}/` | ❌ | ✅ | ✅ | Read-only |
| PATCH | `/api/v1/genres/{id}/` | ❌ | ❌ | ✅ | Update |
| DELETE | `/api/v1/genres/{id}/` | ❌ | ❌ | ✅ | Only if no songs linked |

*Query params (GET /genres/):* `page`, `page_size`.

---

### 9. Payment APIs

| Method | Endpoint | LISTENER | ADMIN | SUPER_ADMIN | Notes |
|--------|----------|:--------:|:-----:|:------------:|------|
| POST | `/api/v1/payments/create-payment-link/` | ❌ | ✅ | ❌ | Razorpay link |
| GET | `/api/v1/payments/subscription/` | ❌ | ✅ | ✅ | Tenant subscription |
| POST | `/api/v1/payments/webhook/razorpay/` | — | — | — | Public, signature verified |
| GET | `/api/v1/payments/super-admin/subscriptions/` | ❌ | ❌ | ✅ | All subscriptions |
| GET | `/api/v1/payments/super-admin/payments/` | ❌ | ❌ | ✅ | All payments |

*Query params (GET /payments/super-admin/subscriptions/):* `page`, `page_size`, `tenant_id`, `is_premium`.

*Query params (GET /payments/super-admin/payments/):* `page`, `page_size`, `tenant_id`, `status`.

---

## Data Scope Rules

### Own Data

- Profile (`/users/me/`), own playlists, own song requests.
- LISTENER-only: self-delete.

### Tenant Data

- Users, playlists, song requests, tenant-song links, TENANT songs.
- ADMIN: full CRUD within tenant. LISTENER: own resources only.

### Global Data

- Tenants, GLOBAL songs, genres, platform subscriptions/payments.
- SUPER_ADMIN only (except tenant detail GET for ADMIN).

---

## Object Ownership Rules

- **IsOwnerOrAdmin:** Owner or ADMIN in same tenant can access. SUPER_ADMIN excluded from tenant objects.
- **Tenant Membership:** All tenant-scoped access checks `user.tenant` vs object’s tenant.
- **Cross-Tenant:** ADMIN cannot access other tenants’ data.

---

## Error Handling

| Code | Meaning |
|------|---------|
| **401 Unauthorized** | Missing or invalid token |
| **403 Forbidden** | Valid token but insufficient permissions |
| **404 Not Found** | Resource does not exist or outside caller’s scope |
| **400 Bad Request** | Validation or malformed request |
| **409 Conflict** | Duplicate or constraint violation |
| **500 Internal Server Error** | Unexpected server error |

- Prefer **403** when the user is authenticated but not allowed to perform the action.
- Use **404** when the resource is unknown or not in scope (no info leakage about existence).

---

## Pagination

List endpoints use page-based pagination.

- **Query params:** `page`, `page_size` (default/max per project settings).
- **Response:** `count`, `page`, `page_size`, `next`, `previous`, `data`.
