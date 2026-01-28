# SongList Backend

Multi-tenant music content management API built with Django REST Framework. It lets organizations (tenants) manage music libraries, playlists, song requests, and subscriptions with strict data isolation and role-based access.

---

## Project overview

SongList is the backend for a SaaS platform that serves many independent organizations (tenants), such as venues, labels, and event companies.  
Each tenant can:

- Maintain its own music catalog and playlists
- Accept and manage live song requests
- Offer subscription-based access for listeners

All of this runs on a shared platform that enforces tenant isolation and central administration.

## Problem statement

Most existing music request or playlist systems are built for a **single** organization. This leads to several issues when you try to scale to many tenants:

- Hard to onboard and manage many independent tenants in one system
- Weak or manual data isolation between organizations
- No clear separation of responsibilities between listeners, tenant admins, and platform admins
- Inconsistent or ad-hoc APIs for mobile/web clients

SongList aims to solve these issues with a purpose-built multi-tenant backend.

## Proposed solution and objectives

SongList provides:

- A multi-tenant architecture with clear separation between GLOBAL and TENANT-level data
- Role-based access control for listeners, tenant admins, and platform super admins
- Consistent, well-documented REST APIs for mobile and web clients
- Extensible modules for songs, playlists, requests, billing, and operational workflows

The main objectives are to:

- Make onboarding new tenants simple and predictable
- Guarantee strong tenant isolation and auditable access patterns
- Keep the API surface consistent, discoverable, and easy to integrate with
- Leave room for future features like analytics, recommendations, and third-party integrations


### Functional documentation

The functional documentation focuses on user roles, capabilities, and end-to-end journeys:

- **LISTENER**: Interacts with playlists and song requests.
- **ADMIN**: Manages a single tenant’s catalogs, playlists, and operational workflows.
- **SUPER_ADMIN**: Manages the overall platform, tenants, and global settings.

See [Functional Documentation](functional-docs.md) for full details on behavior, edge cases, and flows.

### Technical documentation

| Topic | Description |
|-------|-------------|
| [API Design](technical/api-design.md) | REST endpoints, data scope, error handling |
| [Authentication](technical/authentication.md) | OTP, JWT, permissions, deletion & restoration |
| [Core Modules](technical/core-modules.md) | Users, tenants, songs, playlists, and more |
| [Tech Stack](technical/tech-stack.md) | Django, PostgreSQL, Redis, Celery, Razorpay |
| [Schema Design](technical/schema-design.md) | Models, relationships, constraints |
| [Developer Setup](technical/developer-setup.md) | Local run, env vars, migrations, seeding |
| [Conventions](technical/conventions.md) | Tenant isolation, GLOBAL vs TENANT, guardrails |
