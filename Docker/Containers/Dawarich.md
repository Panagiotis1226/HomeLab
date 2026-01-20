## Dawarich Stack (Redis + Postgres/PostGIS + Rails App + Sidekiq)

This directory contains a Docker Compose stack for running the **Dawarich** application with:

- **Redis** for caching and background jobs
- **Postgres + PostGIS** as the main database
- **Rails web app** (`dawarich_app`)
- **Sidekiq worker** (`dawarich_sidekiq`)

All services are attached to a private Docker network called `dawarich`.

---

### Services Overview

- **`dawarich_redis`**
  - Image: `redis:7.4-alpine`
  - Role: Key‑value store / cache / job backend.
  - Volumes:
    - `./dawarich_shared:/data` – Redis data and shared state.

- **`dawarich_db`**
  - Image: `imresamu/postgis:17-3.5-alpine`
  - Role: PostgreSQL database with PostGIS extensions.
  - Volumes:
    - `./dawarich_db_data:/var/lib/postgresql/data` – persistent database data.
    - `./dawarich_shared:/var/shared` – shared directory (used by the app and DB).
  - Env (with defaults):
    - `POSTGRES_USER` (default: `postgres`)
    - `POSTGRES_PASSWORD` (default: `password`)
    - `POSTGRES_DB` (default: `dawarich_development`)

- **`dawarich_app`**
  - Image: `freikin/dawarich:latest`
  - Role: Rails web application.
  - Ports:
    - `DAWARICH_APP_PORT:3000` → **Host port defaults to 3000.**
      - **This is the port you must port‑forward from your router/firewall if you want external access.**
  - Volumes:
    - `./dawarich_public:/var/app/public`
    - `./dawarich_watched:/var/app/tmp/imports/watched`
    - `./dawarich_storage:/var/app/storage`
    - `./dawarich_db_data:/dawarich_db_data`
  - Key Env (with defaults):
    - `RAILS_ENV` (default: `development`)
    - `REDIS_URL` (default: `redis://dawarich_redis:6379`)
    - `DATABASE_HOST` (default: `dawarich_db`)
    - `DATABASE_PORT` (default: `5432`)
    - `DATABASE_USERNAME` (default: `postgres`)
    - `DATABASE_PASSWORD` (default: `password`)
    - `DATABASE_NAME` (default: `dawarich_development`)
    - `MIN_MINUTES_SPENT_IN_CITY` (default: `60`)
    - `APPLICATION_HOSTS` (default: `localhost,::1,127.0.0.1`)
    - `TIME_ZONE` (default: `America/Toronto`)
    - `APPLICATION_PROTOCOL` (default: `http`)
    - `PROMETHEUS_EXPORTER_ENABLED` (default: `false`)
    - `PROMETHEUS_EXPORTER_HOST` (default: `0.0.0.0`)
    - `PROMETHEUS_EXPORTER_PORT` (default: `9394`)
    - `SECRET_KEY_BASE` (default: `"CHANGE_ME"` – **change this in production**)
    - `RAILS_LOG_TO_STDOUT` (default: `true`)
    - `SELF_HOSTED` (default: `true`)
    - `STORE_GEODATA` (default: `true`)

- **`dawarich_sidekiq`**
  - Image: `freikin/dawarich:latest`
  - Role: Sidekiq background job processor.
  - Volumes:
    - `./dawarich_public:/var/app/public`
    - `./dawarich_watched:/var/app/tmp/imports/watched`
    - `./dawarich_storage:/var/app/storage`
  - Key Env (with defaults):
    - `RAILS_ENV` (default: `development`)
    - `REDIS_URL` (default: `redis://dawarich_redis:6379`)
    - `DATABASE_HOST` (default: `dawarich_db`)
    - `DATABASE_PORT` (default: `5432`)
    - `DATABASE_USERNAME` (default: `postgres`)
    - `DATABASE_PASSWORD` (default: `password`)
    - `DATABASE_NAME` (default: `dawarich_development`)
    - `APPLICATION_HOSTS` (default: `localhost,::1,127.0.0.1`)
    - `BACKGROUND_PROCESSING_CONCURRENCY` (default: `10`)
    - `APPLICATION_PROTOCOL` (default: `http`)
    - `PROMETHEUS_EXPORTER_ENABLED` (default: `false`)
    - `PROMETHEUS_EXPORTER_HOST` (default: `dawarich_app`)
    - `PROMETHEUS_EXPORTER_PORT` (default: `9394`)
    - `SECRET_KEY_BASE` (default: `"CHANGE_ME"` – **must match app**)
    - `RAILS_LOG_TO_STDOUT` (default: `true`)
    - `SELF_HOSTED` (default: `true`)
    - `STORE_GEODATA` (default: `true`)

---

### Port Forwarding

- **Internal container port:** `3000` (Rails app).
- **Exposed host port:** `${DAWARICH_APP_PORT:-3000}`.
- To reach Dawarich from outside your LAN:
  - Forward **TCP port 3000** on your router to the Docker host’s **port 3000** (or whatever value you set for `DAWARICH_APP_PORT`).
  - Access it from the internet via `http://<your-public-ip-or-domain>:<DAWARICH_APP_PORT>`.

If you change `DAWARICH_APP_PORT` in your `.env` file, update your router/firewall forwarding accordingly.

---

### Environment Configuration

You can override all `${VAR:-default}` values using a `.env` file in the same directory as your `docker-compose.yml`. Example:

```env
POSTGRES_USER=dawarich
POSTGRES_PASSWORD=supersecurepassword
POSTGRES_DB=dawarich_production

DAWARICH_APP_PORT=8080
RAILS_ENV=production
APPLICATION_HOSTS=yourdomain.com
TIME_ZONE=Europe/Athens
SECRET_KEY_BASE=your_really_long_secret_key
```

> **Important:** For any non‑test deployment, set a strong `POSTGRES_PASSWORD` and `SECRET_KEY_BASE`, and change `RAILS_ENV` to `production`.

---

### Volumes & Data Persistence

The stack creates and uses the following local directories:

- `./dawarich_db_data` – persistent PostgreSQL/PostGIS data.
- `./dawarich_shared` – shared files between DB and app.
- `./dawarich_public` – public assets served by the Rails app.
- `./dawarich_watched` – directory watched for imports.
- `./dawarich_storage` – application storage (uploads, generated files, etc.).

Back up these directories if you need to preserve data between hosts or rebuilds.

---

### How to Run

1. **Copy the Docker Compose file** into this directory and name it `docker-compose.yml` (if it isn’t already).
2. **(Optional) Create a `.env` file** next to `docker-compose.yml` to override defaults (see example above).
3. **Start the stack**:

   ```bash
   docker compose up -d
   ```

4. **Check container status**:

   ```bash
   docker compose ps
   docker compose logs -f dawarich_app
   ```

5. **Access the app**:
   - Local: `http://localhost:${DAWARICH_APP_PORT:-3000}`
   - Remote: `http://<your-public-ip-or-domain>:<DAWARICH_APP_PORT>`

---

### Stopping and Removing

- **Stop containers (keep data):**

  ```bash
  docker compose down
  ```

- **Stop containers and remove named volumes (wipe data):**

  ```bash
  docker compose down -v
  ```

Use `down -v` only if you are sure you want to delete all Dawarich data.

