# Reglas de Infraestructura Docker — Fintech Microservicios

## Regla 0 — Instalación de Docker (pre-requisito único)

```
1. Descargar Docker Desktop: https://www.docker.com/products/docker-desktop/
2. Instalar y reiniciar el sistema
3. Verificar instalación:
   docker --version            # ej: Docker version 26.x.x
   docker compose version      # ej: Docker Compose version v2.x.x
   docker ps                   # debe listar contenedores (vacío está bien)
```

> Docker Desktop en Windows activa automáticamente el daemon y WSL 2 backend.

---

## Regla 1 — Estructura de archivos Docker por repositorio

Cada microservicio y el frontend debe tener:

```
{repo-raíz}/
├── docker/
│   ├── docker-compose.infra.yml   ← Solo infraestructura compartida (dev local)
│   └── docker-compose.yml         ← Stack completo: infra + microservicio
├── Dockerfile                     ← Multi-stage build del servicio
└── .dockerignore
```

---

## Regla 2 — docker-compose.infra.yml (infraestructura compartida de desarrollo)

Crear este archivo en la raíz de `PruebaNetCoreProject/docker/docker-compose.infra.yml`:

```yaml
version: '3.9'

services:

  mysql-fintech:
    image: mysql:8.0
    container_name: mysql-fintech
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: fintechroot2024
      MYSQL_DATABASE: fintech_db
      MYSQL_USER: fintech
      MYSQL_PASSWORD: fintech2024
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-pfintechroot2024"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb-fintech:
    image: mongo:7.0
    container_name: mongodb-fintech
    restart: unless-stopped
    environment:
      MONGO_INITDB_DATABASE: fintech_db
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis-fintech:
    image: redis:7-alpine
    container_name: redis-fintech
    restart: unless-stopped
    command: redis-server --save 20 1 --loglevel warning --requirepass fintech2024
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "fintech2024", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka-fintech:
    image: bitnami/kafka:3.7
    container_name: kafka-fintech
    restart: unless-stopped
    environment:
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka-fintech:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
      - KAFKA_CFG_DEFAULT_REPLICATION_FACTOR=1
      - KAFKA_CFG_NUM_PARTITIONS=3
    ports:
      - "9092:9092"
    volumes:
      - kafka_data:/bitnami/kafka
    healthcheck:
      test: ["CMD-SHELL", "kafka-topics.sh --list --bootstrap-server localhost:9092 || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 5

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui-fintech
    restart: unless-stopped
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: fintech-local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-fintech:9092
    depends_on:
      kafka-fintech:
        condition: service_healthy

volumes:
  mysql_data:
  mongo_data:
  redis_data:
  kafka_data:

networks:
  default:
    name: fintech-network
```

---

## Regla 3 — Dockerfile multi-stage para microservicio .NET Core 9

Crear `Dockerfile` en la raíz de cada microservicio .NET:

```dockerfile
# ─── Stage 1: Build ──────────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

COPY ["src/Presentation/API/API.csproj", "src/Presentation/API/"]
COPY ["src/Core/Application/Application.csproj", "src/Core/Application/"]
COPY ["src/Core/Domain/Domain.csproj", "src/Core/Domain/"]
COPY ["src/Infrastructure/Persistence/Persistence.csproj", "src/Infrastructure/Persistence/"]
COPY ["src/Infrastructure/DependencyInjection/DependencyInjection.csproj", "src/Infrastructure/DependencyInjection/"]

RUN dotnet restore "src/Presentation/API/API.csproj"

COPY . .
WORKDIR "/src/src/Presentation/API"
RUN dotnet build "API.csproj" -c Release -o /app/build

# ─── Stage 2: Publish ─────────────────────────────────────────────────────────
FROM build AS publish
RUN dotnet publish "API.csproj" -c Release -o /app/publish --no-restore

# ─── Stage 3: Runtime ─────────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "PruebaNetCoreProject.API.dll"]
```

---

## Regla 4 — Dockerfile multi-stage para frontend Next.js 15

Crear `Dockerfile` en la raíz de `fintech-frontend/`:

```dockerfile
# ─── Stage 1: Dependencies ────────────────────────────────────────────────────
FROM node:22-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

# ─── Stage 2: Build ──────────────────────────────────────────────────────────
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED 1

RUN npm run build

# ─── Stage 3: Runtime ─────────────────────────────────────────────────────────
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

---

## Regla 5 — Comandos de validación de servicios

### Levantar infraestructura
```bash
# Levantar todos los servicios de infraestructura
docker compose -f docker/docker-compose.infra.yml up -d

# Esperar a que todos estén healthy
docker compose -f docker/docker-compose.infra.yml ps
```

### Validar MySQL
```bash
docker exec mysql-fintech mysqladmin ping -h localhost -u root -pfintechroot2024
# Respuesta esperada: mysqld is alive

docker exec mysql-fintech mysql -u root -pfintechroot2024 -e "SHOW DATABASES;"
# Debe mostrar: fintech_db
```

### Validar MongoDB
```bash
docker exec mongodb-fintech mongosh --eval "db.adminCommand('ping')"
# Respuesta esperada: { ok: 1 }

docker exec mongodb-fintech mongosh --eval "db.getCollectionNames()" fintech_db
```

### Validar Redis
```bash
docker exec redis-fintech redis-cli -a fintech2024 ping
# Respuesta esperada: PONG

docker exec redis-fintech redis-cli -a fintech2024 info server | grep redis_version
```

### Validar Kafka
```bash
docker exec kafka-fintech kafka-topics.sh --list --bootstrap-server localhost:9092
# Debe devolver lista de topics (vacía al inicio está bien)

docker exec kafka-fintech kafka-broker-api-versions.sh --bootstrap-server localhost:9092
# Debe mostrar versiones sin error

# Kafka UI disponible en: http://localhost:8090
```

### Validar todos a la vez (health check rápido)
```bash
docker compose -f docker/docker-compose.infra.yml ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
# Todos deben mostrar status "healthy" o "running"
```

### Detener infraestructura
```bash
docker compose -f docker/docker-compose.infra.yml down
# Para borrar volúmenes (reset total):
docker compose -f docker/docker-compose.infra.yml down -v
```

---

## Regla 6 — .dockerignore (backend .NET)

Crear `.dockerignore` en la raíz del microservicio .NET:

```
**/bin/
**/obj/
**/.git/
**/*.user
**/.vs/
**/node_modules/
**/docker-compose*.yml
**/Dockerfile*
**/.env
**/*.md
```

---

## Regla 7 — Variables de entorno en contenedores (no hardcodear secretos)

- Nunca pasar contraseñas directamente en `docker compose up` inline.
- Usar archivo `.env` (gitignoreado) con las variables:

```env
# .env (en la raíz — AGREGAR AL .gitignore)
MYSQL_ROOT_PASSWORD=fintechroot2024
MYSQL_PASSWORD=fintech2024
REDIS_PASSWORD=fintech2024
JWT_AUTHORITY=https://auth-service:5001
JWT_AUDIENCE=fintech-api
```

- En `docker-compose.infra.yml` referenciar como: `${MYSQL_ROOT_PASSWORD}`

---

## Regla 8 — Actualizar connection strings al usar Docker

Con Docker, los `appsettings.Development.json` deben usar las credenciales de los contenedores:

```json
{
  "ConnectionStrings": {
    "MySQL":   "Server=127.0.0.1;Port=3306;Database=fintech_db;Uid=fintech;Pwd=fintech2024;",
    "MongoDB": "mongodb://localhost:27017",
    "Redis":   "localhost:6379,password=fintech2024"
  }
}
```

> La contraseña de Redis cambió: `password=fintech2024` en la connection string.
