# Evolution API — EasyPanel Setup

Deploy from **this repo** (`sass-botflow/evolution-api`).

## EasyPanel → Source (Git)

| Field | Value |
|-------|-------|
| Owner | `sass-botflow` |
| Repository | `evolution-api` |
| Branch | `main` |
| Build Path | `/` |
| Docker Compose File | `docker-compose.yml` |

Click **Save**, then **Deploy**.

A healthy first deploy takes **2–4 minutes** (pulling images + healthchecks).  
If it finishes in **3 seconds**, Source was empty or compose file missing.

## Environment (3 variables only)

```env
SERVER_URL=https://evolution.api.botflow.ink
AUTHENTICATION_API_KEY=<openssl rand -hex 32>
DATABASE_CONNECTION_URI=postgresql://botflow:PASSWORD@sass-botflow_postgres:5432/evolution?schema=evolution_api
```

Create database in Postgres first:

```sql
CREATE DATABASE evolution;
```

## Domain

| Setting | Value |
|---------|-------|
| Service | `evolution-api` |
| Port | `8080` |
| Domain | `evolution.api.botflow.ink` |

## BotFlow backend (separate service)

Add on **backend** app, not here:

```env
EVOLUTION_API_URL=http://sass-botflow_evolution-api:8080
EVOLUTION_API_KEY=<same as AUTHENTICATION_API_KEY>
```

## Verify

```bash
curl -s https://evolution.api.botflow.ink/health
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Deploy 3 seconds | Fill Repository + Branch, ensure `docker-compose.yml` exists on `main` |
| Orange compose warning | Remove extra env vars — keep only the 3 above |
| `invalid interpolation format` | Delete `WEBHOOK_GLOBAL_ENABLED` etc. from Environment |
| Memory NaN | Service not running — check Logs after deploy |
