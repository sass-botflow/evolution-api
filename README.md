# evolution-api

This repository is a **pointer** for the BotFlow Evolution API stack.

The real deployment lives in the **backend** monorepo:

| What | Where |
|------|--------|
| Docker Compose | [`sass-botflow/backend` → `deploy/evolution-api/docker-compose.yml`](https://github.com/sass-botflow/backend/blob/main/deploy/evolution-api/docker-compose.yml) |
| EasyPanel guide | [`deploy/evolution-api/EASYPANEL.md`](https://github.com/sass-botflow/backend/blob/main/deploy/evolution-api/EASYPANEL.md) |
| WhatsApp flow guide | [`sass-botflow/frontend` → `DEPLOY-WHATSAPP-DABA.md`](https://github.com/sass-botflow/frontend/blob/main/DEPLOY-WHATSAPP-DABA.md) |

## Do not deploy from this repo

Use **EasyPanel → Add Service → Compose** with:

- Repo: `sass-botflow/backend`
- Branch: `main`
- Compose path: `deploy/evolution-api/docker-compose.yml`

## Environment (Compose app only — 3 variables)

```env
SERVER_URL=https://evolution.api.botflow.ink
AUTHENTICATION_API_KEY=<openssl rand -hex 32>
DATABASE_CONNECTION_URI=postgresql://botflow:PASSWORD@sass-botflow_postgres:5432/evolution?schema=evolution_api
```

Create the database first:

```sql
CREATE DATABASE evolution;
```

## Backend (separate EasyPanel app)

`EVOLUTION_API_URL` and `EVOLUTION_API_KEY` go on the **BotFlow backend**, not here:

```env
EVOLUTION_API_URL=http://sass-botflow_evolution-api:8080
EVOLUTION_API_KEY=<same as AUTHENTICATION_API_KEY above>
```

## Verify

```bash
curl -s https://evolution.api.botflow.ink/health
curl -s https://api.botflow.ink/health | jq '.config.evolution'
```
