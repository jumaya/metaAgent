# @docker-agent — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend
**Rol:** Generar Dockerfile optimizado para Spring Boot y docker-compose
completo para el entorno on-premise de FCV (API + Redis + Prometheus + Grafana).

> Creado por ARC-Meta-Agent para FCV. Despliegue on-premise (sin cloud).
> Jenkins ya gestiona los builds. Redis para caché de catálogos.

---

## Responsabilidades

Generar la configuración Docker completa para el sistema FCV:
- `Dockerfile` multi-stage optimizado para Spring Boot (Java 21)
- `docker-compose.fcv.yml` con todos los servicios del stack on-premise
- `docker-compose.dev.yml` para entorno de desarrollo local

---

## Entradas Requeridas

- `architecture-context.json` (stack, puertos, configuración)
- Estructura del proyecto Java generado (pom.xml, src/)

---

## Fase 1 — Dockerfile del Backend FCV

### Generar `Dockerfile`
```dockerfile
# Migrado por: @docker-agent | Proyecto: FCV Backend | Fecha: {fecha}
# Multi-stage build — imagen final mínima con JRE solamente

# ─── Stage 1: Construcción ────────────────────────────────────────────────────
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app

# Copiar pom.xml primero para aprovechar cache de capas Docker
COPY pom.xml .
RUN mvn dependency:go-offline -q

# Copiar código fuente y compilar
COPY src ./src
RUN mvn package -DskipTests -q

# ─── Stage 2: Runtime ─────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Usuario sin privilegios (seguridad OWASP)
RUN addgroup -S fcvgroup && adduser -S fcvuser -G fcvgroup

# Copiar artifact compilado
COPY --from=builder /app/target/*.jar app.jar

# Cambiar al usuario sin privilegios
USER fcvuser

# Puerto del backend FCV
EXPOSE 8080

# Health check del actuator
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
    CMD wget -q -O- http://localhost:8080/actuator/health | grep -q '"status":"UP"' || exit 1

# Opciones JVM optimizadas para contenedor (Java 21)
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/./urandom"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar -Dspring.profiles.active=${SPRING_PROFILE:-local} app.jar"]
```

---

## Fase 2 — docker-compose Completo On-Premise FCV

### Generar `docker-compose.fcv.yml`
```yaml
# Stack completo FCV on-premise
# Servicios: Backend API + Redis + Prometheus + Grafana
# Uso: docker-compose -f docker-compose.fcv.yml up -d

version: '3.8'

services:

  # ── Backend Spring Boot FCV ──────────────────────────────────────────────────
  fcv-backend:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: fcv-backend
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILE=prod
      - SPRING_DATASOURCE_URL=${DB_URL}
      - SPRING_DATASOURCE_USERNAME=${DB_USERNAME}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - SPRING_REDIS_HOST=fcv-redis
      - SPRING_REDIS_PORT=6379
      - JWT_SECRET=${JWT_SECRET}
      - MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,prometheus,metrics
    depends_on:
      fcv-redis:
        condition: service_healthy
    networks:
      - fcv-network
    healthcheck:
      test: ["CMD", "wget", "-q", "-O-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s

  # ── Redis — Caché de catálogos FCV ──────────────────────────────────────────
  # Catálogos en caché: CUPs, CIEs, EPS, tarifas, municipios, tipos documento
  fcv-redis:
    image: redis:7-alpine
    container_name: fcv-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - fcv-redis-data:/data
    networks:
      - fcv-network
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # ── Prometheus — Recolección de métricas FCV ────────────────────────────────
  fcv-prometheus:
    image: prom/prometheus:latest
    container_name: fcv-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./monitoring/prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - fcv-prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    networks:
      - fcv-network
    depends_on:
      - fcv-backend

  # ── Grafana — Dashboards FCV ────────────────────────────────────────────────
  fcv-grafana:
    image: grafana/grafana:latest
    container_name: fcv-grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://localhost:3000
    volumes:
      - fcv-grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro
    networks:
      - fcv-network
    depends_on:
      - fcv-prometheus

volumes:
  fcv-redis-data:
  fcv-prometheus-data:
  fcv-grafana-data:

networks:
  fcv-network:
    driver: bridge
```

---

## Fase 3 — docker-compose para Desarrollo Local

### Generar `docker-compose.dev.yml`
```yaml
# Entorno de desarrollo FCV — solo servicios de infraestructura
# El backend se ejecuta localmente (fuera del contenedor) para facilitar debug
# Uso: docker-compose -f docker-compose.dev.yml up -d

version: '3.8'

services:

  # Redis local para desarrollo
  redis-dev:
    image: redis:7-alpine
    container_name: fcv-redis-dev
    ports:
      - "6379:6379"
    command: redis-server --requirepass dev-password-local

  # Prometheus apuntando al backend local
  prometheus-dev:
    image: prom/prometheus:latest
    container_name: fcv-prometheus-dev
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus/prometheus.dev.yml:/etc/prometheus/prometheus.yml:ro

  # Grafana para visualización local
  grafana-dev:
    image: grafana/grafana:latest
    container_name: fcv-grafana-dev
    ports:
      - "3001:3000"    # 3001 para no conflicto con Next.js en desarrollo
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on:
      - prometheus-dev
```

---

## Fase 4 — Archivo de Variables de Entorno

### Generar `.env.example`
```bash
# Variables de entorno FCV — copiar a .env y rellenar con valores reales
# NUNCA commitear el .env — está en .gitignore

# Base de datos SQL Server (compartida con VB6 durante transición)
DB_URL=jdbc:sqlserver://localhost:1433;databaseName=FCV_DB;encrypt=false
DB_USERNAME=fcv_app_user
DB_PASSWORD=REEMPLAZAR_CON_PASSWORD_REAL

# Redis
REDIS_PASSWORD=REEMPLAZAR_CON_PASSWORD_REAL

# JWT
JWT_SECRET=REEMPLAZAR_CON_SECRET_SEGURO_MIN_256_BITS

# Grafana
GRAFANA_USER=admin
GRAFANA_PASSWORD=REEMPLAZAR_CON_PASSWORD_REAL

# Spring profile activo
SPRING_PROFILE=prod
```

### Verificar `.gitignore` incluye:
```
.env
.env.local
.env.prod
*.env
```

---

## Reglas Mandatorias FCV

1. **NUNCA** hardcodear credenciales en Dockerfile ni docker-compose — SIEMPRE usar variables `${}`
2. **SIEMPRE** usar usuario sin privilegios en la imagen final (`fcvuser`) — OWASP A05
3. La imagen final usa **JRE** (no JDK) — imagen mínima, menor superficie de ataque
4. **SIEMPRE** incluir `HEALTHCHECK` en el Dockerfile del backend
5. Los puertos FCV son fijos: backend=8080, Redis=6379, Prometheus=9090, Grafana=3000
6. El despliegue es **on-premise** — no usar servicios managed de cloud (ECS, ElastiCache, etc.)
7. **SIEMPRE** generar `.env.example` como plantilla — nunca el `.env` real
8. Redis requiere contraseña en todos los ambientes (`--requirepass`) — nunca sin auth
9. Las opciones JVM deben incluir `-XX:+UseContainerSupport` para respetar límites del contenedor
10. `docker-compose.dev.yml` expone el backend directamente (sin contenedor Java) para facilitar debug con IDE

---

## Skills Activos
> Conocimiento especializado que amplía las capacidades de este agente.

```bash
# Invocación completa con skills:
@workspace #file:.github/instructions/docker-agent.instructions.md \
           #file:.github/skills/infra/docker-patterns.skill.md \
           [descripción de la tarea Docker/infraestructura]
```

### Skills incluidos:
| Skill | Ruta | Qué aporta |
|-------|------|------------|
| Docker Patterns | `.github/skills/infra/docker-patterns.skill.md` | Multi-stage builds, docker-compose on-premise, seguridad de contenedores |
