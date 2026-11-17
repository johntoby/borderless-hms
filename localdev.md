# Local Development Guide

This guide documents the steps to run the Borderless Hospital Management System locally using Docker Desktop.

---

## Prerequisites

- Docker Desktop 24+
- Python 3.x (for load testing)
- Git Bash or WSL (recommended over CMD for running shell commands)

---

## 1. Create the `.env` File

Copy the template and fill in real local values:

```bash
cp .env.example .env
```

Set at least:

- `POSTGRES_PASSWORD` — any strong local password (used both to initialise the Postgres container and by the backend)
- `SECRET_KEY` — at least 32 random characters, e.g. `openssl rand -hex 32` (the backend refuses to start with a weak or placeholder key)

> `docker-compose.yml` overrides `POSTGRES_SERVER=postgres` and `POSTGRES_SSL_MODE=disable` for the backend,
> so the RDS values in `.env.example` do not need to change for local use. No AWS credentials are needed.

---

## 2. Start the Application

```bash
docker compose up --build
```

This builds and starts:
- `hms-postgres` — PostgreSQL 16 database
- `hms-backend` — FastAPI backend on port 8000
- `hms-frontend` — React frontend (Nginx) on port 3000
- `hms-proxy` — Nginx reverse proxy on port 8080

### Access the App

| URL | Description |
|-----|-------------|
| http://localhost:8080 | Full app (via nginx proxy) |
| http://localhost:8000/api/v1/docs | Swagger API docs |

### Default Login

| Username | Password | Role |
|----------|----------|------|
| admin | Admin@12345 | System Administrator |

---

## 3. Stopping and Cleaning Up

```bash
# Stop containers (keeps data)
docker compose down

# Stop containers AND delete the database volume (full reset)
docker compose down -v
```

> Use `down -v` when you need a completely fresh database. If you just run `down`, the postgres volume persists and data is kept between restarts.

---

## 4. Rebuilding After Code Changes

If you change backend code, rebuild only the backend (faster than rebuilding everything):

```bash
docker compose up -d --build backend
```

For frontend changes:

```bash
docker compose up -d --build frontend
```

---

## 5. Viewing Logs

```bash
# All services
docker compose logs -f

# Backend only
docker logs hms-backend --tail 50

# Postgres only
docker logs hms-postgres --tail 50
```

---

## 6. Database Access (pgAdmin)

A pgAdmin service is included for browsing the database via a GUI.

### Start pgAdmin

```bash
docker compose --profile tools up -d pgadmin
```

### Access pgAdmin

Open http://localhost:5050

| Field | Value |
|-------|-------|
| Email | admin@admin.com |
| Password | admin |

> Defaults for local use only; override with `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD` in your shell or `.env`.

### Connect to the Database

1. Right-click **Servers → Register → Server**
2. **General tab** — Name: `HMS`
3. **Connection tab**:

| Field | Value |
|-------|-------|
| Host | postgres |
| Port | 5432 |
| Database | hospital_db |
| Username | postgres |
| Password | the `POSTGRES_PASSWORD` from your `.env` |

### Viewing Table Data

1. Expand **Servers → HMS → Databases → hospital_db → Schemas → public → Tables**
2. Right-click a table (e.g. `patients`)
3. Click **View/Edit Data → All Rows**

---

## 7. Load Testing with Locust

### Install Locust

```bash
cd load-tests
python -m pip install locust
```

### Run with Web UI (Git Bash / WSL)

```bash
export LOAD_TEST_USERNAME=admin
export LOAD_TEST_PASSWORD=Admin@12345
python -m locust -f locustfile.py --host=http://localhost:8080
```

Open http://localhost:8089, then set:
- **Number of users**: `10` (keep low for local)
- **Spawn rate**: `2`
- Click **Start**

### Run Headless

```bash
export LOAD_TEST_USERNAME=admin
export LOAD_TEST_PASSWORD=Admin@12345
python -m locust -f locustfile.py --host=http://localhost:8080 --users=10 --spawn-rate=2 --run-time=1m --headless
```

> Keep users at 10–20 locally. The 500-user target in the README is for the AWS deployment.

> **Note:** Use `export` in Git Bash/WSL. Use `set` only in Windows CMD.

---

