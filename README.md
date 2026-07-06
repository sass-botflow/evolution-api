# evolution-api

BotFlow WhatsApp stack — Evolution API + Redis for EasyPanel Compose.

## EasyPanel Git settings

| Field | Value |
|-------|-------|
| Owner | `sass-botflow` |
| Repository | `evolution-api` |
| Branch | `main` |
| Build Path | `/` |
| Docker Compose File | `docker-compose.yml` |

Full guide: [EASYPANEL.md](./EASYPANEL.md)

> **Important:** use EasyPanel **Compose**, not **App**.  
> If logs show `docker buildx build ... Dockerfile` → wrong service type.

## Environment (3 variables)

```env
SERVER_URL=https://evolution.api.botflow.ink
AUTHENTICATION_API_KEY=<openssl rand -hex 32>
DATABASE_CONNECTION_URI=postgresql://botflow:PASSWORD@sass-botflow_postgres:5432/evolution?schema=evolution_api
```

Copy from [easypanel.env.example](./easypanel.env.example).

## Backend connection

On **sass-botflow/backend** EasyPanel app:

```env
EVOLUTION_API_URL=http://sass-botflow_evolution-api:8080
EVOLUTION_API_KEY=<same as AUTHENTICATION_API_KEY>
```
