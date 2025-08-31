# 🚀 Stack n8n + PostgreSQL + Redis (Queue Mode)

Este repositorio contiene la configuración en **Docker Compose** para desplegar **n8n** con:
- **PostgreSQL** como base de datos persistente.  
- **Redis (Bull)** como gestor de colas (Queue Mode).  
- **Workers** escalables para manejar alta concurrencia.  

---

## 📂 Estructura de proyecto

```bash
stack-n8n/
├── docker-compose.yml
├── .env
├── n8n_data/          # Datos persistentes de n8n
├── postgres_data/     # Datos persistentes de PostgreSQL
└── redis_data/        # Datos persistentes de Redis
```

---

📌 Genera tu clave de cifrado:
```bash
openssl rand -hex 32
```
Esa clave va en `N8N_ENCRYPTION_KEY` y **no debe cambiarse**.

---

## 🐳 Docker Compose (`docker-compose.yml`)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: n8n-postgres
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_DB: ${POSTGRES_DB}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB} -h 127.0.0.1"]
      interval: 10s
      timeout: 5s
      retries: 10
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    networks: [n8n_net]

  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "${REDIS_PASSWORD}"]
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "PING"]
      interval: 10s
      timeout: 5s
      retries: 10
    volumes:
      - ./redis_data:/data
    networks: [n8n_net]

  n8n-main:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n-main
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "${N8N_PORT}:5678"
    environment:
      TZ: ${TZ}

         # --- NUEVO / RECOMENDADO ---
      N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS: "true"
      N8N_RUNNERS_ENABLED: "true"
      OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS: "true"
      
      GENERIC_TIMEZONE: ${GENERIC_TIMEZONE}
      DB_TYPE: ${DB_TYPE}
      DB_POSTGRESDB_HOST: ${DB_POSTGRESDB_HOST}
      DB_POSTGRESDB_PORT: ${DB_POSTGRESDB_PORT}
      DB_POSTGRESDB_DATABASE: ${DB_POSTGRESDB_DATABASE}
      DB_POSTGRESDB_SCHEMA: ${DB_POSTGRESDB_SCHEMA}
      DB_POSTGRESDB_USER: ${DB_POSTGRESDB_USER}
      DB_POSTGRESDB_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
      EXECUTIONS_MODE: ${EXECUTIONS_MODE}
      QUEUE_BULL_REDIS_HOST: ${QUEUE_BULL_REDIS_HOST}
      QUEUE_BULL_REDIS_PORT: ${QUEUE_BULL_REDIS_PORT}
      QUEUE_BULL_REDIS_DB: ${QUEUE_BULL_REDIS_DB}
      QUEUE_BULL_REDIS_PASSWORD: ${QUEUE_BULL_REDIS_PASSWORD}
      QUEUE_BULL_PREFIX: ${QUEUE_BULL_PREFIX}
      QUEUE_HEALTH_CHECK_ACTIVE: ${QUEUE_HEALTH_CHECK_ACTIVE}
      N8N_HOST: ${N8N_HOST}
      N8N_PORT: 5678
      N8N_EDITOR_BASE_URL: ${N8N_EDITOR_BASE_URL}
      WEBHOOK_URL: ${WEBHOOK_URL}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      N8N_DIAGNOSTICS_ENABLED: ${N8N_DIAGNOSTICS_ENABLED}
    volumes:
      - ./n8n_data:/home/node/.n8n
    networks: [n8n_net]
    restart: unless-stopped

  n8n-worker:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n-worker-1
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      TZ: ${TZ}
      GENERIC_TIMEZONE: ${GENERIC_TIMEZONE}
      DB_TYPE: ${DB_TYPE}
      DB_POSTGRESDB_HOST: ${DB_POSTGRESDB_HOST}
      DB_POSTGRESDB_PORT: ${DB_POSTGRESDB_PORT}
      DB_POSTGRESDB_DATABASE: ${DB_POSTGRESDB_DATABASE}
      DB_POSTGRESDB_SCHEMA: ${DB_POSTGRESDB_SCHEMA}
      DB_POSTGRESDB_USER: ${DB_POSTGRESDB_USER}
      DB_POSTGRESDB_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
      EXECUTIONS_MODE: ${EXECUTIONS_MODE}
      QUEUE_BULL_REDIS_HOST: ${QUEUE_BULL_REDIS_HOST}
      QUEUE_BULL_REDIS_PORT: ${QUEUE_BULL_REDIS_PORT}
      QUEUE_BULL_REDIS_DB: ${QUEUE_BULL_REDIS_DB}
      QUEUE_BULL_REDIS_PASSWORD: ${QUEUE_BULL_REDIS_PASSWORD}
      QUEUE_BULL_PREFIX: ${QUEUE_BULL_PREFIX}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      N8N_MODE: worker   # Define que es un worker
      WEBHOOK_URL: ${WEBHOOK_URL}
    networks: [n8n_net]
    restart: unless-stopped

networks:
  n8n_net:
    driver: bridge
```

---

## 🚦 Arranque

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Accede a la UI en:  
👉 [http://localhost:5678](http://localhost:5678)

---

## 🔄 Escalar Workers

```bash
docker compose up -d --scale n8n-worker=3
```

Esto levanta 3 workers adicionales que consumen de la misma cola Redis.

---

## 🛠️ Desarrollo vs Producción

| Opción                         | Desarrollo       | Producción           |
|--------------------------------|------------------|----------------------|
| `OFFLOAD_MANUAL_EXECUTIONS...` | ❌ (off)         | ✅ (on, todo en cola) |
| `N8N_DIAGNOSTICS_ENABLED`      | opcional         | `false` (recomendado) |
| `N8N_SECURE_COOKIE`            | `false`          | `true` (si HTTPS)     |
| `WEBHOOK_URL`                  | `http://localhost:5678` | `https://n8n.midominio.com` |

---

## 🧾 Conceptos clave

- **Healthcheck**: prueba si el servicio está “sano” antes de usarse.  
- **Redis + Bull**: manejan la cola de ejecuciones en `queue mode`.  
- **Main vs Worker**:
  - *Main*: UI + webhooks → envía jobs a la cola.  
  - *Worker*: consume jobs de Redis y ejecuta workflows.  
- **Network bridge**: todos los contenedores se comunican en `n8n_net` usando nombres de servicio, no IPs.

---

## 📦 Backup

- **Base de datos (Postgres)**: copia `postgres_data/` o haz `pg_dump`.  
- **Credenciales (n8n)**: carpeta `n8n_data/`.  
- **Redis**: carpeta `redis_data/` (logs de cola, no críticos si tienes Postgres).

---

## 🔐 Seguridad

- Usa un `N8N_ENCRYPTION_KEY` fuerte y **persistente**.  
- Configura `N8N_SECURE_COOKIE=true` con HTTPS.  
- No expongas Redis ni Postgres directamente a internet.  

---

## 📈 Diagrama simplificado

```txt
        ┌──────────────┐
        │   n8n-main   │  ← UI + Webhooks
        └──────┬───────┘
               │  (jobs)
               ▼
        ┌──────────────┐
        │    Redis     │  ← Cola (Bull)
        └──────┬───────┘
               │
        ┌──────┴───────┐
        │   Workers    │  ← Ejecutan workflows
        └──────┬───────┘
               │
        ┌──────┴───────┐
        │  PostgreSQL  │  ← Almacena ejecuciones, credenciales
        └──────────────┘
```

---

## 🔄 Actualización

```bash
docker compose pull
docker compose up -d
```

Mantén siempre la misma `N8N_ENCRYPTION_KEY` para no perder credenciales cifradas.
