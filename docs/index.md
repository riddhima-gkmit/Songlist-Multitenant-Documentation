# SongList Backend

Multi-tenant music content management API (Django REST Framework). Organizations (tenants) manage music libraries, playlists, song requests, and subscriptions with strict isolation and role-based access.

---

## Overview

### Roles

**LISTENER**, **ADMIN**, **SUPER_ADMIN**. See [Functional Documentation](functional-docs.md) for capabilities and user journeys.

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
