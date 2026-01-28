# Functional Documentation

This document describes **what** the SongList Backend does at a business level: roles, features, user journeys, and system behavior. It does not describe implementation or list endpoints.

**On this page**

| Section | Description |
|---------|-------------|
| [What the System Does](#what-the-system-does) | High-level purpose |
| [Roles](#roles) | LISTENER, ADMIN, SUPER_ADMIN |
| [Core Features](#core-features) | Auth, tenants, songs, playlists, etc. |
| [User Journeys](#user-journeys) | LISTENER, ADMIN, SUPER_ADMIN flows |
| [System Behavior](#system-behavior) | Isolation, deletion, safeguards |
| [Summary](#summary) | Recap and pointers |

---

## What the System Does

SongList Backend is a **multi-tenant music content management** system. Organizations (tenants) use it to:

- Run a music library and playlist experience for their users
- Manage users, songs, and song requests within each tenant
- Offer premium subscriptions per tenant
- Keep each tenant’s data fully isolated from others

**Platform operators:**

- Manage tenants
- Manage global song catalog
- Manage genres

**Tenant admins:**

- Manage their organization's users
- Manage songs and playlists
- Manage subscriptions

---

## Roles

### LISTENER

Regular tenant users.

**Can:**

- Register (within a tenant), verify email, log in, manage own profile, self-delete
- View approved songs in their tenant
- Create and manage own playlists; add/remove songs in own playlists
- Submit song requests

**Cannot:**

- Access songs directly (must use tenant songs)
- Access genres
- Approve or fulfill requests
- Manage other users
- Access other users' playlists or tenant admin features

### ADMIN

Tenant administrators.

**Can:**

- Everything a LISTENER can within the tenant, plus:
- Create LISTENER users
- View/update/delete tenant users (except self and other admins)
- View/update/delete any tenant playlist and manage playlist songs
- Link/unlink songs to the tenant
- Approve/reject/fulfill song requests
- Create payment links
- View tenant subscription and tenant details

**Cannot:**

- Delete themselves
- Delete other admins or super admins
- Access other tenants' data
- Manage tenants or platform-level config
- Manage GLOBAL songs or genres
- View platform-wide subscriptions/payments

### SUPER_ADMIN

Platform administrators.

**Can:**

- Log in without a tenant
- Manage tenants (create, update, activate, deactivate, delete when empty)
- Manage GLOBAL songs and genres
- Create ADMIN users for any tenant
- View user counts and all admins
- View all subscriptions and payments

**Cannot:**

- Access any tenant-specific data (users, playlists, song requests, tenant-song links, TENANT songs)
- Access PII beyond admin listing

---

## Core Features

### Authentication and Roles

**Registration:**

- LISTENER only
- Tenant-scoped
- ADMIN and SUPER_ADMIN are created by others, not self-registration

**Email verification:**

- Required before first login
- OTP-based

**Login:**

- Two-step OTP (request OTP → verify OTP)
- JWT access and refresh tokens issued
- SUPER_ADMIN uses a separate, tenant-free login

**Password reset:**

- OTP-based
- Tenant-scoped for tenant users

**Password change:**

- Blacklists all existing access and refresh tokens
- Forces re-login everywhere

**Logout:**

- Blacklists refresh token
- Denylists access token

### Tenants

**Purpose:**

- Represent organizations
- All tenant users and tenant-specific data belong to a tenant

**Management:**

- SUPER_ADMIN creates, updates, activates, deactivates, and deletes tenants
- Tenant cannot be deleted if it has any active users

**Visibility:**

- ADMIN can view their tenant's details only (read-only)
- LISTENER has no tenant management access

### Songs and Playlists

**Songs:**

- Two scopes: **GLOBAL** and **TENANT**
- GLOBAL songs are platform-wide, managed by SUPER_ADMIN
- TENANT songs are per-tenant, managed by tenant ADMIN
- LISTENER never access the song catalog directly
- LISTENER see only **tenant songs** (links between tenants and songs) that are approved for their tenant

**Tenant-song links:**

- Admins link songs to their tenant
- Admins mark links as approved or not
- LISTENER see only approved tenant songs

**Playlists:**

- User-owned
- LISTENER manage only their own
- ADMIN can manage all playlists in their tenant (create on behalf of users, add/remove songs, etc.)
- Only approved tenant songs can be added
- SUPER_ADMIN has no playlist access

### Song Requests

**Purpose:**

- Users request songs
- Admins review and fulfill (approve, reject, or fulfill by linking/creating a song)

**Flow:**

- LISTENER create and manage their own requests
- ADMIN view all tenant requests
- ADMIN create requests for any tenant user
- ADMIN approve/reject/fulfill requests
- SUPER_ADMIN has no access

**States:**

- PENDING → APPROVED, REJECTED, or FULFILLED

### Genres

**Purpose:**

- Categorize songs
- Platform-wide, shared by all tenants

**Access:**

- SUPER_ADMIN: full CRUD
- ADMIN: read-only
- LISTENER: no access

**Rule:**

- A genre cannot be deleted if any non–soft-deleted song is linked to it

### Payments and Subscriptions

**Purpose:**

- Per-tenant premium (e.g. lifetime) subscription via Razorpay

**Creation:**

- ADMIN create payment links for their tenant
- SUPER_ADMIN does not create payment links

**Status:**

- ADMIN can view tenant subscription status
- SUPER_ADMIN can view tenant subscription status
- SUPER_ADMIN can list all subscriptions and payments platform-wide

**Webhooks:**

- Razorpay webhooks are handled by the system (signature-verified)
- Not user-facing

---

## User Journeys

### LISTENER Flow

**1. Onboarding:**

- Register in a tenant (e.g. via invite or tenant-specific signup)
- Receive verification email
- Verify with OTP
- Log in (request OTP → verify OTP → receive tokens)

**2. Discovery:**

- Browse approved tenant songs only
- No direct access to full song catalog or genres

**3. Requests:**

- Submit song requests
- View and manage own requests
- Optionally request password reset

**4. Playlists:**

- Create playlists
- Add/remove approved tenant songs
- Update or delete own playlists

**5. Account:**

- Update profile
- Change password
- Self-delete (can re-register later; previous content stays deleted)

### ADMIN Flow

**1. Onboarding:**

- Created by SUPER_ADMIN (or existing admin) within a tenant
- Verify email
- Log in via tenant-scoped OTP flow

**2. User management:**

- Create LISTENER users
- View, update, delete LISTENER users in the tenant
- Restore admin-deleted users (with or without restoring content)
- Cannot delete self or other admins

**3. Content:**

- Create/manage TENANT songs
- Link/unlink songs to the tenant
- Bulk add/remove tenant-song links
- View GLOBAL songs as reference (cannot modify)

**4. Requests:**

- View all tenant song requests
- Create requests for any tenant user
- Approve/reject/fulfill requests

**5. Playlists:**

- View, create, update, delete any tenant playlist
- Add/remove songs in any tenant playlist

**6. Tenant and billing:**

- View tenant details (read-only)
- Create payment links
- View subscription status

**7. Account:**

- Manage own profile and password
- Cannot self-delete

### SUPER_ADMIN Flow

**1. Login:**

- Use super-admin login (no tenant)
- Two-step OTP, then JWT

**2. Tenants:**

- Create, update, activate, deactivate tenants
- Delete tenants only when they have no active users
- View all tenants and tenant details

**3. Catalog:**

- Manage GLOBAL songs and genres
- No access to TENANT songs, tenant-song links, or any tenant-specific data

**4. Admins:**

- Create ADMIN users for any tenant
- List all admins platform-wide
- View user counts only (no PII)

**5. Billing:**

- View all subscriptions and payments
- No payment-link creation

---

## System Behavior

### Data Isolation

- Tenant-scoped data is strictly isolated
- SUPER_ADMIN has no access to tenant-specific resources
- See [Conventions](technical/conventions.md) for rules and rationale

### User Deletion and Restoration

**Self-deletion (LISTENER):**

- User deletes own account
- Content remains deleted
- User can re-register in the same tenant
- Account is restored, content is not

**Admin deletion:**

- Admin deletes a LISTENER
- User is blocked from re-registration until restored
- Deleted user must contact support

**Restoration:**

- Admin can restore an admin-deleted user
- Option to restore account only
- Option to restore account plus content (excluding content the user deleted themselves)

### Safeguards

- Admins cannot delete themselves or other admins
- Only LISTENER users can be deleted by admins
- Password change invalidates all tokens for that user
- Genre deletion is blocked when songs are linked
- See [Conventions](technical/conventions.md) for rationale and enforcement

---

## Summary

- Role-based access
- Tenant isolation
- Clear platform vs tenant boundaries
- Per-resource access is documented in [Authentication & Authorization](technical/authentication.md)
