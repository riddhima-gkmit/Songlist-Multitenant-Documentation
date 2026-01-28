# Authentication & Authorization

SongList Backend uses **role-based access control (RBAC)** with three distinct roles and strict tenant isolation. Authentication is OTP-based with JWT tokens, and permissions enforce data access boundaries.

**On this page**

| Section | Description |
|---------|-------------|
| [Roles](#roles) | Role summary |
| [Role Capabilities Matrix](#role-capabilities-matrix) | Per-action access by role |
| [Authentication Flow](#authentication-flow) | Registration, login, OTP, tokens |
| [Permission Classes](#permission-classes) | AllowAny, IsAuthenticated, etc. |
| [User Deletion & Restoration](#user-deletion-restoration) | Self-delete, admin delete, restore |
| [Data Isolation Summary](#data-isolation-summary) | Per-resource access matrix |

---

## Roles

**LISTENER**

- Tenant users
- Own profile, playlists, song requests
- Approved tenant songs only
- Can self-delete

**ADMIN**

- Tenant-scoped
- Full CRUD on tenant users, playlists, requests, tenant-song links, TENANT songs
- Cannot delete self or other admins

**SUPER_ADMIN**

- Platform only
- Tenants, GLOBAL songs, genres, metrics
- View/update/delete ADMIN users; view/restore deleted admins
- No LISTENER or tenant data access

See [Functional Documentation](../functional-docs.md#roles) for capabilities and [Conventions](conventions.md#role-boundaries) for boundaries.

---

## Authorization

### Role Capabilities Matrix

| Action | LISTENER | ADMIN | SUPER_ADMIN |
|--------|----------|-------|-------------|
| **Authentication** |
| Register (LISTENER only) | ✅ | ❌ (contact admin) | ❌ (contact admin) |
| Verify Email | ✅ | ✅ | ✅ |
| Request OTP | ✅ | ✅ | ✅ |
| Verify OTP | ✅ | ✅ | ✅ |
| Refresh Token | ✅ | ✅ | ✅ |
| Change Password | ✅ | ✅ | ✅ |
| Password Reset | ✅ | ✅ | ✅ |
| Logout | ✅ | ✅ | ✅ |
| Resend Verification | ✅ | ✅ | ✅ |
| **User Management** |
| View Own Profile | ✅ | ✅ | ✅ |
| Update Own Profile | ✅ | ✅ | ✅ |
| Delete Own Account | ✅ (self-delete) | ❌ (safety) | ❌ |
| Create LISTENER User | ❌ | ✅ (own tenant) | ❌ |
| Create ADMIN User | ❌ | ❌ | ✅ (any tenant) |
| View All Users | ❌ | ✅ (tenant only) | ✅ (count only, no PII) |
| View Any User | ❌ | ✅ (tenant only) | ✅ (ADMIN only) |
| Update Any User | ❌ | ✅ (tenant only) | ✅ (ADMIN only) |
| Delete Any User | ❌ | ✅ (tenant only, LISTENER only, not self) | ✅ (ADMIN only) |
| Restore Deleted User | ❌ | ✅ (tenant only) | ✅ (deleted admins only) |
| View Deleted Users | ❌ | ✅ (tenant only) | ✅ (deleted admins only, platform-wide) |
| **Tenant Management** |
| View Tenants | ❌ | ❌ | ✅ (all) |
| Create Tenant | ❌ | ❌ | ✅ |
| Update Tenant | ❌ | ❌ | ✅ |
| Activate Tenant | ❌ | ❌ | ✅ |
| Deactivate Tenant | ❌ | ❌ | ✅ |
| Delete Tenant | ❌ | ❌ | ✅ (if no users) |
| View Tenant Details | ❌ | ✅ (GET only) | ✅ |
| View All Admins | ❌ | ❌ | ✅ (platform-wide) |
| **Tenant-Song Links** |
| View Tenant Songs | ✅ (approved only) | ✅ (all, own tenant) | ❌ (no data access) |
| View Tenant Song Detail | ✅ (approved only) | ✅ (all, own tenant) | ❌ (no data access) |
| Link Song to Tenant | ❌ | ✅ (own tenant) | ❌ (no data access) |
| Bulk Add Songs | ❌ | ✅ (own tenant) | ❌ (no data access) |
| Unlink Song | ❌ | ✅ (own tenant) | ❌ (no data access) |
| Bulk Delete Tenant Songs | ❌ | ✅ (own tenant) | ❌ (no data access) |
| **Song Management** |
| View Songs | ❌ (use tenant/songs) | ✅ (all, tenant) | ✅ (GLOBAL only) |
| View Song Details | ❌ (use tenant/songs) | ✅ (all, tenant) | ✅ (GLOBAL only) |
| Create Song | ❌ | ✅ (TENANT songs) | ✅ (GLOBAL songs) |
| Update Song | ❌ | ✅ (TENANT songs) | ✅ (GLOBAL songs) |
| Delete Song | ❌ | ✅ (TENANT songs) | ✅ (GLOBAL songs) |
| **Song Requests** |
| Create Song Request | ✅ (own) | ✅ (any user, tenant) | ❌ |
| View Any Request | ✅ (own only) | ✅ (any, tenant) | ❌ (no data access) |
| Update Any Request | ✅ (own only) | ✅ (any, tenant) | ❌ (no data access) |
| Delete Any Request | ✅ (own only) | ✅ (any, tenant) | ❌ (no data access) |
| View All Requests | ❌ | ✅ (tenant only) | ❌ (no data access) |
| Review Requests | ❌ | ✅ (tenant only) | ❌ (no data access) |
| **Playlist Management** |
| View Playlists | ✅ (own) | ✅ (tenant) | ❌ (no data access) |
| Create Playlist | ✅ (own) | ✅ (any user, tenant) | ❌ (no data access) |
| View Playlist Details | ✅ (own) | ✅ (tenant) | ❌ (no data access) |
| Update Playlist | ✅ (own) | ✅ (tenant) | ❌ (no data access) |
| Delete Playlist | ✅ (own) | ✅ (tenant) | ❌ (no data access) |
| Get Playlist Songs | ✅ (own) | ✅ (tenant) | ❌ (no data access) |
| Add Song to Playlist | ✅ (own) | ✅ (tenant) | ❌ (no data access) |
| Remove Song from Playlist | ✅ (own) | ✅ (tenant) | ❌ (no data access) |
| **Genre Management** |
| View Genres | ❌ | ✅ | ✅ |
| View Genre Detail | ❌ | ✅ | ✅ |
| Create Genre | ❌ | ❌ | ✅ |
| Update Genre | ❌ | ❌ | ✅ |
| Delete Genre | ❌ | ❌ | ✅ (no linked songs) |
| **Payments** |
| Create Payment Link | ❌ | ✅ | ❌ |
| View Subscription Status | ❌ | ✅ | ✅ |
| View All Subscriptions | ❌ | ❌ | ✅ |
| View All Payments | ❌ | ❌ | ✅ |
| Handle Webhooks | ❌ | ❌ | ❌ |

---

## Authentication Flow

### Registration

- **Endpoint**: `POST /api/v1/tenant/{tenant_id}/auth/register/`
- **Access**: LISTENER only (ADMIN and SUPER_ADMIN blocked)
- **Process**:
  1. User provides username, email, password, confirm_password
  2. Account created with `is_verified=false`
  3. Verification email sent with OTP
  4. User must verify email before login
- **Special Behavior**: Automatically restores self-deleted accounts

### Email Verification

- **Endpoint**: `POST /api/v1/tenant/{tenant_id}/auth/verify-email/`
- **Access**: All users (unverified accounts)
- **Process**:
  1. User provides OTP received via email
  2. OTP verified against cache
  3. Account marked as verified

### Login (Two-Step OTP)

**Step 1: Request OTP**

- **Endpoint**: `POST /api/v1/tenant/{tenant_id}/auth/login/`
- **Access**: All verified users
- **Process**: 
  1. User provides username/email and password
  2. Password verified
  3. OTP sent to email
  4. OTP cached in Redis

**Step 2: Verify OTP**

- **Endpoint**: `POST /api/v1/tenant/{tenant_id}/auth/login/verify-otp/`
- **Access**: All verified users
- **Process**:
  1. User provides OTP
  2. OTP verified against cache
  3. JWT access and refresh tokens issued
  4. OTP invalidated

### Super Admin Login

- **Step 1**: `POST /api/v1/auth/super-admin/login/` (no tenant required)
- **Step 2**: `POST /api/v1/auth/super-admin/login/verify-otp/`
- **Access**: SUPER_ADMIN only

### Token Refresh

- **Endpoint**: `POST /api/v1/token/refresh/`
- **Access**: All authenticated users
- **Process**:
  1. User provides valid refresh token
  2. New access token issued
  3. Old refresh token blacklisted (token rotation)
- **Token Rotation**: Enabled (old refresh token blacklisted)

### Password Reset

**Step 1: Request Reset OTP**

- **Endpoint**: `POST /api/v1/tenant/{tenant_id}/auth/password-reset/`
- **Access**: All users
- **Process**:
  1. User requests password reset with email/username
  2. OTP sent to registered email
  3. OTP cached in Redis

**Step 2: Confirm New Password**

- **Endpoint**: `POST /api/v1/tenant/{tenant_id}/auth/password-reset/confirm/`
- **Access**: All users
- **Process**:
  1. User provides OTP and new password
  2. OTP verified against cache
  3. New password set
  4. OTP invalidated

### Change Password

- **Endpoint**: `POST /api/v1/users/me/change-password/`
- **Access**: All authenticated users
- **Security**: Blacklists all active access and refresh tokens (forces re-login on all devices)

### Logout

- **Endpoint**: `POST /api/v1/auth/logout/`
- **Access**: All authenticated users
- **Process**:
  1. Refresh token blacklisted
  2. Access token denylisted

---

## Permission Classes

### AllowAny

Public access for:

- Registration
- Email verification
- Login (OTP request & verify)
- Password reset
- Resend verification

### IsAuthenticated

Logged-in users only. Used for:

- Profile management
- Change password
- Songs, playlists, genres
- Tenant-song links
- Song requests

### IsSuperAdmin

Platform management only. Used for:

- Tenant CRUD (except GET detail)
- Activate/deactivate tenants
- Genre CRUD
- Admin listing
- Payment/subscription listing

### IsAdmin

Tenant data management. Used for:

- User management
- Tenant-song links
- Song request review
- User deletion/restoration
- Bulk song operations
- Payment link creation

### IsAdminOrSuperAdmin

Admin or Super Admin access. Used for:
- User creation/list
- Subscription status
- Tenant detail GET

### IsOwnerOrAdmin

Object-level permission:

- ADMIN can access any object in their tenant
- Owner can access their own object
- Blocks SUPER_ADMIN from tenant data

Used for: Playlist detail operations

### IsTenantUser

Tenant members only (blocks SUPER_ADMIN). Used for:

- Song request endpoints

---

## User Deletion & Restoration {: #user-deletion-restoration }

### Self-Deletion

**Process**:

1. User calls `DELETE /api/v1/users/me/`
2. `deleted_by` set to user's own ID
3. User's content (songs, playlists) marked as deleted
4. User can re-register via `/api/v1/tenant/{tenant_id}/auth/register/`
5. Account automatically restored on re-registration
6. **Content stays deleted** (not restored)

**Access**: LISTENER only (ADMIN and SUPER_ADMIN blocked for safety)

### Admin Deletion

**Process**:

1. ADMIN calls `DELETE /api/v1/users/{user_id}/delete/`
2. `deleted_by` set to admin's ID
3. User's active songs/playlists marked as deleted
4. User blocked from re-registration
5. Registration attempt returns: "Your account has been deleted by admin. Please contact support."

**Rules**:
- Cannot delete themselves
- Cannot delete other ADMIN or SUPER_ADMIN users
- Can only delete LISTENER users in their tenant

### Admin Restoration

**Endpoint**: `POST /api/v1/users/{user_id}/restore/`

**Options**:

**Without Data** (`restore_data=false`):

- Account restored (user can login)
- All content stays deleted

**With Data** (`restore_data=true`):

- Account restored
- ALL deleted songs/playlists restored **EXCEPT** items user deleted themselves
- Includes content deleted by admin during user deletion
- Includes content deleted by other admins
- Excludes content where `deleted_by == user_id`

**Access:** ADMIN (tenant-scoped, any deleted user in tenant); SUPER_ADMIN (deleted admins only, platform-wide).

---

## Data Isolation Summary

| Resource | LISTENER | ADMIN | SUPER_ADMIN |
|----------|----------|-------|-------------|
| Users | Own only | Tenant only | Count only (no PII) |
| Songs | ❌ No direct access (use tenant/songs) | TENANT (own tenant) | GLOBAL only |
| Playlists | Own only | Tenant only (full CRUD) | ❌ No access |
| Song Requests | Own only | Tenant only | ❌ No access |
| Tenant-Songs | Approved only | Own tenant (full CRUD) | ❌ No access |
| Genres | ❌ No access | Read-only | Full control |
| Payments | ❌ No access | Create payment links, view subscription | View all subscriptions/payments |
| Tenants | ❌ No access | View details (read-only) | Full control |

---

Security guardrails (tenant isolation, deletion rules, password-change behavior) are in [Conventions](conventions.md#security-decisions).
