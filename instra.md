# Huddlefix AI OS Audit

Huddlefix AI OS Audit is a Docker-based local development stack containing a
React/TypeScript/Vite frontend, Laravel API, FastAPI AI service, MySQL 8.4, and
phpMyAdmin. The AI service uses Gemini 2.5 Flash through Vertex AI with each
developer's Google Application Default Credentials (ADC).

## Quick start

Complete the operating-system and Google authentication setup below first. Then:

```sh
cp .env.example .env
# Replace every placeholder in .env and set your GOOGLE_ADC_PATH.
docker compose up -d --build
docker compose exec backend php artisan migrate
docker compose ps
```

After initial setup, the normal start command is:

```sh
docker compose up -d
```

Docker automatically starts Laravel, Vite, and Uvicorn. Developers do **not**
manually run `php artisan serve`, `npm run dev`, or
`python -m uvicorn app.main:app`.

## Architecture

```text
Browser
  |
  +-- http://localhost:5173 ----------------> frontend (React + Vite)
  |                                                |
  |                                  browser API requests
  |                                                |
  +-- http://localhost:8000/api ------------> backend (Laravel)
                                                   |          |
                                          mysql:3306          |
                                                   |          |
                                                MySQL         |
                                                              |
                                              http://ai-service:8081
                                                              |
                                                       ai-service (FastAPI)
                                                              |
                                                   Vertex AI / Gemini 2.5 Flash
```

All services share the `huddlefix` Docker network. Container-to-container traffic
uses Docker service names; browser traffic uses `localhost`. Source is bind-mounted
for development, while Laravel `vendor`, frontend `node_modules`, and MySQL data
use named volumes.

The full Docker flow has been manually validated:
React → Laravel → MySQL → FastAPI → Gemini → Laravel → React. Laravel, frontend,
FastAPI, and MySQL health checks were observed as healthy. phpMyAdmin runs as part
of the stack but does not have a Docker health-status claim.

## Services and ports

| Service | Host address | Purpose |
|---|---|---|
| Frontend | <http://localhost:5173> | React/Vite application |
| Laravel | <http://localhost:8000> | Backend |
| Laravel health | <http://localhost:8000/api/health> | Backend health JSON |
| FastAPI health | <http://localhost:8081/health> | Public AI-service health JSON |
| FastAPI docs | <http://localhost:8081/docs> | Interactive API documentation |
| phpMyAdmin | <http://localhost:8082> | Browser database administration |
| MySQL | `localhost:3306` | MySQL protocol, not an HTTP browser URL |

Internal Docker addresses are:

- Laravel → MySQL: `mysql:3306`
- Laravel → FastAPI: `http://ai-service:8081`
- Browser → Laravel: `http://localhost:8000/api`

## Environment configuration

The root `.env` is the main Docker Compose configuration source:

```sh
cp .env.example .env
```

Required root variables:

| Variable | Purpose |
|---|---|
| `MYSQL_DATABASE` | Development database created by MySQL |
| `MYSQL_TEST_DATABASE` | Test database created during first initialization |
| `MYSQL_USER` | MySQL application user |
| `MYSQL_PASSWORD` | MySQL application-user password |
| `MYSQL_ROOT_PASSWORD` | Local MySQL root password |
| `APP_KEY` | Laravel encryption key |
| `AI_SERVICE_TOKEN` | Shared Laravel-to-FastAPI bearer token |
| `GOOGLE_ADC_PATH` | Absolute host path to the developer's ADC JSON |
| `GCP_PROJECT_ID` | Required project: `trusty-charmer-502915-r4` |
| `GCP_LOCATION` | Vertex AI region |
| `GEMINI_MODEL` | Gemini model, currently `gemini-2.5-flash` |

Never commit `.env` files, ADC JSON, or service-account keys.

Compose maps root `AI_SERVICE_TOKEN` to Laravel's `AI_SERVICE_TOKEN` and
FastAPI's `INTERNAL_API_TOKEN`, so they receive the same value. If
`backend/.env` is used, its `DB_PASSWORD` must match root `MYSQL_PASSWORD`.
Laravel's `DB_*` settings configure the Laravel database client; the MySQL
container's `MYSQL_*` settings initialize the server. They are related but are
not interchangeable variable names.

Generate an `APP_KEY` and paste its output into root `.env`:

```sh
docker compose run --rm backend php artisan key:generate --show
```

Generate an AI token with a password manager or a cryptographically secure tool,
then paste the output into `AI_SERVICE_TOKEN` without saving it in shell history or
documentation. For example:

```sh
openssl rand -base64 48
```

The README intentionally shows no generated output or real secret values.

### Existing MySQL volumes and password changes

`MYSQL_USER`, `MYSQL_PASSWORD`, and initialization scripts only take effect when
MySQL initializes an empty data directory. Changing root `.env` does not update a
user password already stored in `huddlefix_mysql_data`.

When changing an existing user's password, update the MySQL account with an
authorized database-administration session, then update root `.env` and any
`backend/.env` using that password. Do not delete the volume merely to change a
password.

## Initial database setup

Run migrations after the services are healthy:

```sh
docker compose exec backend php artisan migrate
docker compose exec backend php artisan migrate:status
```

Standard setup does not use `migrate:fresh`, destructive migrations, or automatic
seeders.

## Verify the application

Health checks:

```sh
curl http://localhost:8000/api/health
curl http://localhost:8081/health
```

Expected responses:

```json
{"status":"ok"}
```

Both current health endpoints return that JSON. Create an audit session:

```sh
curl -i -X POST \
  -H "Accept: application/json" \
  http://localhost:8000/api/v1/audits
```

Expected status: `HTTP 201 Created`.

## Command reference

| Task | Command |
|---|---|
| Normal start | `docker compose up -d` |
| Initial start or rebuild | `docker compose up -d --build` |
| Status and health | `docker compose ps` |
| Follow all logs | `docker compose logs -f` |
| Stop safely | `docker compose stop` |
| Resume stopped services | `docker compose up -d` |
| Remove containers, preserve volumes | `docker compose down` |
| Restart backend | `docker compose restart backend` |
| Recent backend logs | `docker compose logs --tail=100 backend` |
| Recent AI-service logs | `docker compose logs --tail=100 ai-service` |
| Clear Laravel optimizations | `docker compose exec backend php artisan optimize:clear` |
| Clear Laravel config cache | `docker compose exec backend php artisan config:clear` |
| Show migration status | `docker compose exec backend php artisan migrate:status` |

`docker compose down` removes containers and the Compose network but preserves
named volumes. **Warning:** `docker compose down -v` deletes the MySQL database
volume and the dependency volumes. Never use it when data must be preserved.

Rebuild after changing `requirements.txt`, `composer.lock`, or
`package-lock.json`:

```sh
docker compose up -d --build
```

## Tests and static checks

```sh
docker compose exec backend php artisan test
docker compose exec ai-service python -m pytest
docker compose exec frontend npm run lint
docker compose exec frontend npm run build
```

## Ubuntu setup

1. Install Docker Engine and the Docker Compose plugin.
2. Install the Google Cloud CLI.
3. Where appropriate, add your user to the Docker group. Log out and back in, or
   restart your session, before testing Docker again. Docker-group membership is
   privileged access.
4. Authenticate:

   ```sh
   gcloud auth login
   gcloud config set project trusty-charmer-502915-r4
   gcloud auth application-default login
   ```

5. ADC is normally created at:

   ```text
   /home/<username>/.config/gcloud/application_default_credentials.json
   ```

6. Set the absolute path in root `.env`:

   ```dotenv
   GOOGLE_ADC_PATH=/home/<username>/.config/gcloud/application_default_credentials.json
   ```

7. Follow Quick start.

## macOS setup

1. Install and start Docker Desktop.
2. Install the Google Cloud CLI.
3. Run:

   ```sh
   gcloud auth login
   gcloud config set project trusty-charmer-502915-r4
   gcloud auth application-default login
   ```

4. ADC is normally at:

   ```text
   /Users/<username>/.config/gcloud/application_default_credentials.json
   ```

5. Configure:

   ```dotenv
   GOOGLE_ADC_PATH=/Users/<username>/.config/gcloud/application_default_credentials.json
   ```

6. If Docker cannot mount it, allow the containing directory in Docker Desktop's
   file-sharing settings.
7. Follow Quick start.

## Windows setup

### Preferred: WSL2

1. Use Windows 10/11 with WSL2 and an Ubuntu distribution.
2. Install Docker Desktop and enable integration for that WSL distribution.
3. Install `gcloud` inside WSL.
4. Clone the repository inside the WSL filesystem, such as
   `/home/<username>/projects`, for better bind-mount performance.
5. From WSL run:

   ```sh
   gcloud auth login
   gcloud config set project trusty-charmer-502915-r4
   gcloud auth application-default login
   ```

6. Configure the normal WSL ADC location:

   ```dotenv
   GOOGLE_ADC_PATH=/home/<username>/.config/gcloud/application_default_credentials.json
   ```

7. Run Docker Compose and all project commands from WSL.

### Alternative: native PowerShell

1. Install Docker Desktop and the Windows Google Cloud CLI.
2. In PowerShell run the same three `gcloud` authentication commands.
3. ADC is normally at:

   ```text
   C:/Users/<username>/AppData/Roaming/gcloud/application_default_credentials.json
   ```

4. Use forward slashes in root `.env`:

   ```dotenv
   GOOGLE_ADC_PATH=C:/Users/<username>/AppData/Roaming/gcloud/application_default_credentials.json
   ```

5. Ensure Docker Desktop has drive and file-sharing permission for the credential
   directory and repository.
6. Follow Quick start from PowerShell.

## Google Cloud access and team development

The required GCP project ID is `trusty-charmer-502915-r4`. Each developer must:

- use their own authorized Huddlefix Google account;
- have access to that project;
- receive suitable minimum Vertex AI permissions, not broad Owner or Editor access;
- create and mount their own local ADC file.

Never distribute one developer's ADC to another developer.

The recommended future model is developer service-account impersonation with
short-lived credentials. Production should use a separate service account attached
to its Cloud Run workload. Do not create or circulate shared downloadable
service-account JSON keys.

## Troubleshooting

### Port already in use

Identify Linux listeners with:

```sh
ss -ltnp
```

On macOS, or systems with `lsof`:

```sh
lsof -nP -iTCP:8081 -sTCP:LISTEN
```

Replace `8081` with 3306, 5173, 8000, or 8082 as needed. Stop or reconfigure the
conflicting process, then run `docker compose up -d`.

### FastAPI `/` returns `{"detail":"Not Found"}`

This is normal because FastAPI has no root route. Use
<http://localhost:8081/health> or <http://localhost:8081/docs>.

### `localhost:3306` does not open in a browser

MySQL uses its own protocol, not HTTP. Use a MySQL client or phpMyAdmin at
<http://localhost:8082>.

### Laravel database access denied

Confirm root `MYSQL_USER`/`MYSQL_PASSWORD` agree with any backend
`DB_USERNAME`/`DB_PASSWORD`. Compare values without printing them. For example,
in a shell where the two values are already loaded:

```sh
test "${#MYSQL_PASSWORD}" -eq "${#DB_PASSWORD}" && echo "lengths match"
printf '%s' "$MYSQL_PASSWORD" | sha256sum
printf '%s' "$DB_PASSWORD" | sha256sum
```

Matching hashes indicate matching values. Hash output still deserves care: do not
paste it into tickets or screenshots. Avoid commands that echo the passwords.

If root `.env` changed after MySQL was first initialized, the named volume retains
the old account password. Update the MySQL account through an authorized admin
session instead of deleting the volume.

### MySQL says host is not allowed

The application user must accept connections from the Docker network. The provided
initialization grants the configured user access from `%`. For an existing volume,
inspect and update that user's MySQL host grant using an authorized admin session.

### Laravel accidentally uses SQLite

Docker must use:

```dotenv
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
```

Clear stale configuration after correcting it.

### Laravel connection refused

Inside the backend container, `127.0.0.1` refers to the backend container itself,
not MySQL. Use `DB_HOST=mysql`, confirm `docker compose ps` reports MySQL healthy,
and inspect `docker compose logs mysql`.

### Laravel CLI works but HTTP requests fail

A stale cached configuration file may affect the long-running backend:

```sh
rm -f backend/bootstrap/cache/config.php
docker compose up -d --force-recreate backend
docker compose exec backend php artisan config:clear
```

The first command removes only Laravel's generated configuration cache.

### Cache clear fails before migrations

`CACHE_STORE=database` requires the cache table. Run migrations first:

```sh
docker compose exec backend php artisan migrate
docker compose exec backend php artisan config:clear
```

### Gemini returns 502 because ADC expired

Refresh personal ADC and restart the AI service:

```sh
gcloud auth application-default login
docker compose restart ai-service
```

### `GOOGLE_ADC_PATH` is missing or cannot mount

Use an existing absolute host file path, not the container path. On Docker Desktop,
grant file-sharing permission to its directory. Native Windows paths in `.env` must
use forward slashes. Inspect `docker compose logs --tail=100 ai-service`.

### FastAPI returns 401

Laravel's `AI_SERVICE_TOKEN` and FastAPI's `INTERNAL_API_TOKEN` differ. Compose maps
both from root `AI_SERVICE_TOKEN`; correct root `.env` and recreate both services.

### phpMyAdmin rejects a saved browser password

Log in with the current `MYSQL_USER` and `MYSQL_PASSWORD` from root `.env`. Update
or remove the browser/password-manager credential saved for localhost if it is
stale.

### Frontend retains an audit after the database was reset

In browser DevTools Console run:

```js
localStorage.clear();
location.reload();
```

### Backend shows unhealthy

Open or request <http://localhost:8000/api/health>, then inspect:

```sh
docker compose logs --tail=100 backend
docker compose exec backend php artisan optimize:clear
```

The backend Docker health check requests
`http://127.0.0.1:8000/api/health` from inside the container.

## Security

- Never commit real passwords, `.env` files, `APP_KEY`, `AI_SERVICE_TOKEN`, ADC
  data, or service-account keys.
- Screenshots, shell history, copied terminal output, and support tickets can expose
  credentials. Redact them before sharing.
- Rotate any credential that was publicly exposed.
- The ADC mount is read-only and credentials are not copied into images. Docker
  daemon access can still inspect mounted files, so treat Docker access as
  privileged.
- Every developer uses a separate personal ADC file; never share one developer's
  ADC with the team.
