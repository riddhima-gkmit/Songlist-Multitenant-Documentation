# Developer Setup

Short, operational guide to run the SongList Backend locally.

**On this page**

| Section | Description |
|---------|-------------|
| [Prerequisites](#prerequisites) | Python, PostgreSQL, Redis, Razorpay |
| [Environment Variables](#environment-variables) | .env reference |
| [Local Setup](#local-setup) | Clone, configure, DB, run |
| [Migrations](#migrations) | makemigrations, migrate |
| [Seeding](#seeding) | seed_genres, seed_tenants, seed_songs |

---

## Prerequisites

- **Python 3.13+**
- **PostgreSQL**
- **Redis** (cache, Celery broker)
- **Razorpay** account (sandbox for development)

---

## Environment Variables

Create a `.env` file in the project root. The app reads these via `python-dotenv`.

| Variable | Description | Example |
|----------|-------------|---------|
| `SECRET_KEY` | Django secret key | `your-secret-key-here` |
| `DEBUG` | Debug mode (`1`/`0`) | `1` |
| `ALLOWED_HOSTS` | Comma-separated hosts | `localhost,127.0.0.1` |
| `DB_NAME` | PostgreSQL database name | `songlist_db` |
| `DB_USER` | Database user | `postgres` |
| `DB_PASSWORD` | Database password | — |
| `DB_HOST` | Database host | `localhost` |
| `DB_PORT` | Database port | `5432` |
| `REDIS_URL` | Redis URL | `redis://127.0.0.1:6379/1` |
| `CELERY_BROKER_URL` | Celery broker | `redis://127.0.0.1:6379/0` |
| `CELERY_RESULT_BACKEND` | Celery results | `redis://127.0.0.1:6379/0` |
| `RAZORPAY_KEY_ID` | Razorpay key (sandbox: `rzp_test_*`) | `rzp_test_xxxxx` |
| `RAZORPAY_KEY_SECRET` | Razorpay secret | — |
| `RAZORPAY_WEBHOOK_SECRET` | Webhook signing secret | — |
| `EMAIL_BACKEND` | Email backend | `django.core.mail.backends.console.EmailBackend` (dev) |
| `EMAIL_HOST` | SMTP host | `smtp.gmail.com` |
| `EMAIL_PORT` | SMTP port | `587` |
| `EMAIL_USE_TLS` | Use TLS | `True` |
| `EMAIL_HOST_USER` | SMTP user | — |
| `EMAIL_HOST_PASSWORD` | SMTP password | — |
| `DEFAULT_FROM_EMAIL` | From address | `noreply@songlist.com` |
| `FRONTEND_URL` | Frontend base URL (e.g. email links) | `http://localhost:3000` |
| `LANGUAGE_CODE` | Language | `en-us` |
| `TIME_ZONE` | Timezone | `Asia/Kolkata` |
| `CELERY_TIMEZONE` | Celery timezone | `Asia/Kolkata` |
| `CELERY_TASK_TIME_LIMIT` | Task time limit (seconds) | `1800` |
| `RATE_LIMIT_ENABLE` | Enable rate limiting | `0` (dev) / `1` |
| `RATE_LIMIT_LOGIN_ATTEMPTS` | Max login attempts per window | `10` |
| `RATE_LIMIT_LOGIN_WINDOW` | Window (seconds) | `60` |

For local development, use `EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend` to print emails to the console.

---

## Local Setup

### 1. Clone and install

```bash
git clone <repository-url>
cd SongList-Backend
uv sync
```

### 2. Configure environment

Copy `.env.example` to `.env` and set at least:

- `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`
- `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`
- `REDIS_URL`
- `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` (sandbox)
- `EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend` (dev)

### 3. Database

Create the PostgreSQL database, then:

```bash
uv run python manage.py migrate
```

### 4. Super admin (SUPER_ADMIN)

```bash
uv run python manage.py createsuperuser
```

Use email as identifier. The created user is SUPER_ADMIN (no tenant).

### 5. Default tenant

Create at least one tenant (e.g. via Django shell):

```bash
uv run python manage.py shell
```

```python
from tenants.models import Tenant
t = Tenant.objects.create(name="Default Tenant", is_active=True)
print(t.id)  # Use for tenant-scoped auth URLs
```

### 6. Run the server

```bash
uv run python manage.py runserver
```

API base: `http://localhost:8000/api/v1/`.

### 7. Optional: Redis and Celery

- **Redis:** Ensure Redis is running (e.g. `redis-server`) for cache and Celery.
- **Celery worker** (payment polling, scheduled tasks):

  ```bash
  celery -A songlist_backend worker -l info
  ```

- **Celery beat** (periodic tasks):

  ```bash
  celery -A songlist_backend beat -l info
  ```

---

## Migrations

- Create migrations: `uv run python manage.py makemigrations`
- Apply: `uv run python manage.py migrate`

---

## Seeding

Management commands live in `common`. Run from project root:

| Command | App | Purpose |
|---------|-----|---------|
| `seed_genres` | common | Seed genres (run first) |
| `seed_tenants` | common | Seed tenants |
| `seed_songs` | common | Seed songs (requires superuser and genres) |

**Example:**

```bash
uv run python manage.py seed_genres
uv run python manage.py seed_tenants   # if using common
uv run python manage.py seed_songs
```

Ensure tenants, users, and genres exist before `seed_songs`.
