# Tech Stack

Technologies used in the SongList Backend. All items below are present in the codebase or dependency configuration.

**On this page**

| Section | Description |
|---------|-------------|
| [Backend](#backend) | Django, DRF, Python |
| [Database and ORM](#database-and-orm) | PostgreSQL, Django ORM |
| [Authentication](#authentication) | JWT, OTP, Redis |
| [Cache and Background Jobs](#cache-and-background-jobs) | Redis, Celery, Beat |
| [Payments](#payments) | Razorpay |
| [Storage](#storage) | Static, media |
| [Email](#email) | SMTP |
| [Tooling and Config](#tooling-and-config) | uv, dotenv, migrations |
| [Security and Middleware](#security-and-middleware) | Rate limit, denylist, etc. |
| [Deployment Assumptions](#deployment-assumptions) | Processes, external services |

---

## Backend

| Technology | Purpose |
|------------|---------|
| **Django** | Core backend framework |
| **Django REST Framework** | REST API |
| **Python** | Language (>=3.13.7 per pyproject) |

---

## Database and ORM

| Technology | Purpose |
|------------|---------|
| **PostgreSQL** | Relational database |
| **Django ORM** | Data access, migrations |

---

## Authentication

| Technology | Purpose |
|------------|---------|
| **djangorestframework-simplejwt** | JWT access and refresh tokens; blacklist after rotation |
| **pyotp** | OTP generation and verification (login, email verification, password reset) |
| **Token blacklist** | Invalidate refresh tokens (e.g. logout, rotation) |
| **Redis** | Cache for OTPs, email verification, access-token denylist |

---

## Cache and Background Jobs

| Technology | Purpose |
|------------|---------|
| **Redis** | Cache backend (django-redis); Celery broker |
| **Celery** | Task queue (payment polling, webhook cleanup, token cleanup) |
| **Celery Beat** | Scheduled tasks (e.g. poll pending payments, cleanup) |

---

## Payments

| Technology | Purpose |
|------------|---------|
| **Razorpay** | Payment gateway; payment links, webhooks, signature verification |

---

## Storage

| Technology | Purpose |
|------------|---------|
| **Local / configurable** | Static files (`STATIC_ROOT`), media (`MEDIA_ROOT`). No dedicated object storage in default setup. |

---

## Email

| Technology | Purpose |
|------------|---------|
| **SMTP** | Transactional email (verification, password reset). Backend configurable (e.g. Gmail). |

---

## Tooling and Config

| Item | Purpose |
|------|---------|
| **uv** | Dependency and environment management (pyproject.toml, uv.lock) |
| **python-dotenv** | Environment variables from `.env` |
| **Django migrations** | Schema changes |
| **Logging** | Request/response and application logging to files |

---

## Security and Middleware

- **Rate limiting:** Login and auth endpoints (Django cache–backed).
- **Token denylist middleware:** Rejects denylisted access tokens.
- **Custom exception handler:** JSON error responses.
- **Django password validators:** Default validators enabled.

---

## Deployment Assumptions

- **Processes:** Django app (WSGI/ASGI), Celery worker, Celery beat.
- **External services:** PostgreSQL, Redis, SMTP, Razorpay (and webhook URL).
- **Config:** Env vars (e.g. `DB_*`, `REDIS_URL`, `RAZORPAY_*`, `EMAIL_*`).

