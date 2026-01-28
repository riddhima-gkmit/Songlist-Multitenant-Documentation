# Schema Design

The schema supports multi-tenancy, tenant isolation, and soft deletes. All models use UUID primary keys unless noted.

**Source:** SongList Backend models. Only models present in the codebase are documented.

**On this page**

| Section | Description |
|---------|-------------|
| [Base and Soft-Delete](#base-and-soft-delete) | BaseModel, SoftDeleteModel |
| [Enums](#enums) | UserRole, SongVisibility, etc. |
| [Users](#users) | users table |
| [Tenants](#tenants) | tenants table |
| [Genres](#genres) | genres table |
| [Songs](#songs) | songs table |
| [TenantSong](#tenantsong) | tenant_song links |
| [Playlists](#playlists) | playlists table |
| [PlaylistSong](#playlistsong) | playlist_song |
| [SongRequest](#songrequest) | song_requests |
| [PaymentTransaction](#paymenttransaction) | payments |
| [Subscription](#subscription) | tenant subscriptions |
| [WebhookEvent](#webhookevent) | webhook events |
| [Relationships](#relationships-summary) | FK summary |
| [ER Diagram](#er-diagram) | Schema diagram |
| [Design Notes](#design-notes) | Deletion, constraints |

---

## Base and Soft-Delete

### BaseModel (abstract)

| Field      | Type      | Description        |
|------------|-----------|--------------------|
| id         | UUID (PK) | Primary key        |
| created_at | DateTime  | Creation time      |
| updated_at | DateTime  | Last update time   |

### SoftDeleteModel (abstract)

Extends `BaseModel`. Adds:

| Field      | Type       | Description                    |
|------------|------------|--------------------------------|
| deleted_at | DateTime   | Set when soft-deleted; null otherwise |
| deleted_by | FK → User  | User who performed delete; optional   |

`delete()` sets `deleted_at` and `deleted_by`; `restore()` clears them. `User` overrides `deleted_by` as a self-referential FK.

---

## Enums

| Enum           | Values                                                    |
|----------------|-----------------------------------------------------------|
| UserRole       | LISTENER, ADMIN, SUPER_ADMIN                              |
| SongVisibility | GLOBAL, TENANT                                            |
| RequestStatus  | PENDING, APPROVED, REJECTED, FULFILLED                    |
| PaymentStatus  | CREATED, PENDING, PAID, VERIFIED, ACTIVATED, FAILED, REFUNDED |

---

## Users

**Table:** `users`  
**Bases:** AbstractUser, SoftDeleteModel. User defines its own `deleted_by` → self.

| Field       | Type       | Description                                |
|------------|------------|--------------------------------------------|
| id         | UUID (PK)  | Primary key                                |
| tenant_id  | FK → Tenant| Null for SUPER_ADMIN; required otherwise   |
| email      | Email      | Unique per tenant; globally unique if tenant null |
| phone_no   | String(15) | Optional                                   |
| role       | Enum       | UserRole; default LISTENER                 |
| is_verified| Boolean    | Must be true to log in                     |
| deleted_by | FK → User  | Self-reference; who deleted (self or admin)|
| deleted_at | DateTime   | Soft delete                                |
| created_at | DateTime   |                                            |
| updated_at | DateTime   |                                            |
| username   | String     | From AbstractUser; globally unique         |
| password   | String     | Hashed                                     |
| first_name | String     |                                            |
| last_name  | String     |                                            |
| is_active  | Boolean    |                                            |
| date_joined| DateTime   |                                            |

**Constraints:** Unique `(tenant, email)` when `tenant` not null; unique `email` when `tenant` null.  
**Indexes:** `(tenant, email)`, `(tenant, role)`, `deleted_at`.  
**Soft delete:** Sets `deleted_at`, `deleted_by`, `is_active = False`; soft-deletes related songs and playlists.

---

## Tenants

**Table:** `tenants`  
**Base:** SoftDeleteModel.

| Field      | Type     | Description        |
|------------|----------|--------------------|
| id         | UUID (PK)| Primary key        |
| name       | String   | Unique among non-deleted |
| is_active  | Boolean  | Default true       |
| deleted_at | DateTime | Soft delete        |
| deleted_by | FK → User|                    |
| created_at | DateTime |                    |
| updated_at | DateTime |                    |

**Constraints:** Unique `name` where `deleted_at` is null.  
**Managers:** Default filters active, non-deleted; `all_tenants` includes inactive/deleted for SUPER_ADMIN.

---

## Genres

**Table:** `genres`  
**Base:** BaseModel (no soft delete).

| Field       | Type     | Description |
|-------------|----------|-------------|
| id          | UUID (PK)| Primary key |
| name        | String   | Unique      |
| description | Text     | Optional    |
| created_at  | DateTime |             |
| updated_at  | DateTime |             |

**Delete rule:** Cannot delete if any non–soft-deleted song references the genre.

---

## Songs

**Table:** `songs`  
**Base:** SoftDeleteModel.

| Field        | Type       | Description                          |
|--------------|------------|--------------------------------------|
| id           | UUID (PK)  | Primary key                          |
| user_id      | FK → User  | Creator                              |
| visibility   | Enum       | SongVisibility; GLOBAL or TENANT     |
| tenant_id    | FK → Tenant| Null for GLOBAL; required for TENANT |
| genre_id     | FK → Genre | Required; PROTECT on delete          |
| title        | String     |                                      |
| artist       | String     |                                      |
| album        | String     | Optional                             |
| duration     | SmallInt   | Seconds                              |
| release_year | SmallInt   | Validated (min year, not future)     |
| deleted_at   | DateTime   | Soft delete                          |
| deleted_by   | FK → User  |                                      |
| created_at   | DateTime   |                                      |
| updated_at   | DateTime   |                                      |

**Indexes:** `(tenant, visibility)`.  
**Validation:** `release_year` between configured min and current year.

---

## TenantSong

**Table:** `tenant_songs`  
**Base:** SoftDeleteModel. Links songs to tenants.

| Field      | Type       | Description        |
|------------|------------|--------------------|
| id         | UUID (PK)  | Primary key        |
| tenant_id  | FK → Tenant|                    |
| song_id    | FK → Song  |                    |
| is_active  | Boolean    | Default true       |
| deleted_at | DateTime   | Soft delete        |
| deleted_by | FK → User  |                    |
| created_at | DateTime   |                    |
| updated_at | DateTime   |                    |

**Constraints:** Unique `(tenant, song)`.  
**Indexes:** `(tenant, song)`, `(tenant, is_active)`.

---

## Playlists

**Table:** `playlists`  
**Base:** SoftDeleteModel.

| Field       | Type      | Description        |
|-------------|-----------|--------------------|
| id          | UUID (PK) | Primary key        |
| user_id     | FK → User | Owner              |
| name        | String    |                    |
| description | Text      | Optional           |
| deleted_at  | DateTime  | Soft delete        |
| deleted_by  | FK → User |                    |
| created_at  | DateTime  |                    |
| updated_at  | DateTime  |                    |

**Constraints:** Unique `(user, name)` where `deleted_at` is null.

---

## PlaylistSong

**Table:** `playlist_songs`  
**Base:** SoftDeleteModel. Join table between **Playlist** and **TenantSong** (not Song directly).

| Field         | Type        | Description    |
|---------------|-------------|----------------|
| id            | UUID (PK)   | Primary key    |
| playlist_id   | FK → Playlist |              |
| tenant_song_id| FK → TenantSong |            |
| deleted_at    | DateTime    | Soft delete    |
| deleted_by    | FK → User   |                |
| created_at    | DateTime    |                |
| updated_at    | DateTime    |                |

**Constraints:** Unique `(playlist, tenant_song)`.

---

## SongRequest

**Table:** `song_requests`  
**Base:** SoftDeleteModel.

| Field           | Type       | Description                |
|-----------------|------------|----------------------------|
| id              | UUID (PK)  | Primary key                |
| tenant_id       | FK → Tenant|                            |
| requester_id    | FK → User  |                            |
| song_title      | String     |                            |
| artist_name     | String     |                            |
| album_name      | String     | Optional                   |
| additional_notes| Text       | Optional                   |
| status          | Enum       | RequestStatus; default PENDING |
| reviewed_by_id  | FK → User  | Null until reviewed        |
| reviewed_at     | DateTime   |                            |
| rejection_reason| Text       |                            |
| fulfilled_song_id| FK → Song  | Null until fulfilled       |
| fulfilled_at    | DateTime   |                            |
| deleted_at      | DateTime   | Soft delete                |
| deleted_by      | FK → User  |                            |
| created_at      | DateTime   |                            |
| updated_at      | DateTime   |                            |

**Constraints:** Unique `(song_title, tenant)`.  
**Indexes:** `(tenant, status)`, `(requester, status)`, `(tenant, created_at)`.

---

## PaymentTransaction

**Table:** `payment_transactions`  
**Base:** BaseModel (no soft delete).

| Field                    | Type     | Description                    |
|--------------------------|----------|--------------------------------|
| id                       | UUID (PK)| Primary key                    |
| tenant_id                | FK → Tenant |                          |
| razorpay_payment_link_id | String   | Unique                         |
| razorpay_payment_id      | String   | Set when paid                  |
| razorpay_signature       | String   | Verification                   |
| payment_link_url         | URL      | From Razorpay                  |
| amount                   | Decimal  |                                |
| currency                 | String(3)| Default INR                    |
| status                   | Enum     | PaymentStatus                  |
| attempt_number           | SmallInt |                                |
| error_message            | Text     |                                |
| metadata                 | JSON     |                                |
| paid_at                  | DateTime |                                |
| verified_at              | DateTime |                                |
| activated_at             | DateTime |                                |
| created_at               | DateTime |                                |
| updated_at               | DateTime |                                |

**Indexes:** `(tenant, status)`; indexes on Razorpay IDs.  
**State flow:** CREATED → PENDING → PAID → VERIFIED → ACTIVATED (or FAILED).

---

## Subscription

**Table:** `subscriptions`  
**Base:** BaseModel. One-to-one per tenant; premium status stored here.

| Field                 | Type       | Description      |
|-----------------------|------------|------------------|
| id                    | UUID (PK)  | Primary key      |
| tenant_id             | OneToOne → Tenant |        |
| is_premium            | Boolean    | Default false    |
| activated_at          | DateTime   |                  |
| source                | String(50) | e.g. razorpay    |
| payment_transaction_id| FK → PaymentTransaction | Nullable |
| created_at            | DateTime   |                  |
| updated_at            | DateTime   |                  |

---

## WebhookEvent

**Table:** `webhook_events`  
**Base:** BaseModel. Razorpay webhook log; audit and idempotency.

| Field            | Type     | Description     |
|------------------|----------|-----------------|
| id               | UUID (PK)| Primary key     |
| razorpay_event_id| String   | Unique          |
| event_type       | String   |                 |
| payload          | JSON     |                 |
| signature        | String   |                 |
| signature_verified| Boolean  |                 |
| processed        | Boolean  |                 |
| processed_at     | DateTime |                 |
| error_message    | Text     |                 |
| idempotency_key  | String   | Unique          |
| created_at       | DateTime |                 |
| updated_at       | DateTime |                 |

---

## Relationships (Summary)

- **User** → Tenant (optional), deleted_by (self).  
- **Tenant** → users, songs, tenant_songs, song_requests, payment_transactions, subscription.  
- **Song** → User, Tenant (optional), Genre; tenant_songs, fulfilled_requests.  
- **TenantSong** → Tenant, Song; playlist_songs.  
- **Playlist** → User; playlist_songs.  
- **PlaylistSong** → Playlist, TenantSong.  
- **SongRequest** → Tenant, User (requester, reviewed_by), Song (fulfilled_song).  
- **Genre** → songs.  
- **PaymentTransaction** → Tenant; subscriptions.  
- **Subscription** → Tenant (OneToOne), PaymentTransaction.

---

## ER Diagram

![SongList Schema Design](../assets/images/SongList-Multitenant-Schema.png)

---
## Design Notes

- **Soft deletes:** Used on users, tenants, songs, tenant_songs, playlists, playlist_songs, song_requests. `deleted_at` and `deleted_by` record when and by whom.
- **Enums:** Roles, visibility, request status, and payment status restrict invalid values.
- **Foreign keys:** Enforce referential integrity; Genre uses PROTECT to block deletes when songs exist.
- **Unique constraints:** Email per tenant; genre name; tenant+song; user+playlist name; playlist+tenant_song; song_request (song_title, tenant); Razorpay and idempotency identifiers.
