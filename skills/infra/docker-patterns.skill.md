# Skill: Docker Patterns — Optimized Dockerfiles and Compose
**Used by:** `@cicd-agent` | **Layer:** Infrastructure

---

## Backend Dockerfile (Spring Boot — Multi-stage)

```dockerfile
# ============================================================
# STAGE 1: Build — full image with Maven and JDK
# ============================================================
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app

# Copy pom.xml first — leverage Docker layer cache
# If only source code changes, dependencies are NOT re-downloaded
COPY pom.xml .
RUN mvn dependency:go-offline -q

# Copy source code and build
COPY src ./src
RUN mvn package -DskipTests -q

# ============================================================
# STAGE 2: Runtime — minimal image with JRE only
# ============================================================
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Non-root user for security — NEVER run as root in production
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only the JAR from the previous stage
COPY --from=builder /app/target/*.jar app.jar

# Switch to non-privileged user
USER appuser

EXPOSE 8080

# Health check — Docker knows if the container is healthy
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD wget -q -O- http://localhost:8080/actuator/health | grep -q '"status":"UP"' || exit 1

# Recommended JVM flags for containers
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Dspring.profiles.active=${SPRING_PROFILE:-prod}", \
  "-jar", "app.jar"]
```

---

## Frontend Dockerfile (Next.js — Standalone)

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app

# ============================================================
# STAGE 1: Dependencies
# ============================================================
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci --frozen-lockfile

# ============================================================
# STAGE 2: Build
# ============================================================
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build-time environment variables (public ones only)
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL

# Enable standalone output (smaller image)
# Requires: output: 'standalone' in next.config.js
RUN npm run build

# ============================================================
# STAGE 3: Runtime
# ============================================================
FROM base AS runtime
ENV NODE_ENV production

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only the necessary files from the standalone build
COPY --from=builder --chown=appuser:appgroup /app/.next/standalone ./
COPY --from=builder --chown=appuser:appgroup /app/.next/static ./.next/static
COPY --from=builder --chown=appuser:appgroup /app/public ./public

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s \
  CMD wget -q -O- http://localhost:3000/api/health || exit 1

ENV PORT 3000
CMD ["node", "server.js"]
```

---

## docker-compose.yml — Development Environment

```yaml
# docker-compose.yml — full local development environment
# Command: docker-compose up -d
version: '3.8'

services:

  # ── Spring Boot Backend ──────────────────────────────────
  backend:
    build:
      context: .
      dockerfile: jenkins/Dockerfile.backend
    container_name: app-backend
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILE: dev
      SPRING_DATASOURCE_URL: jdbc:sqlserver://sqlserver:1433;databaseName=AppDB;TrustServerCertificate=true
      SPRING_DATASOURCE_USERNAME: sa
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}   # from .env
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      sqlserver:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  # ── Next.js Frontend ─────────────────────────────────────
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        NEXT_PUBLIC_API_URL: http://localhost:8080
    container_name: app-frontend
    ports:
      - "3000:3000"
    environment:
      API_URL: http://backend:8080        # internal container-to-container communication
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped

  # ── SQL Server ────────────────────────────────────────────
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-latest
    container_name: app-sqlserver
    ports:
      - "1433:1433"
    environment:
      ACCEPT_EULA: Y
      SA_PASSWORD: ${DB_PASSWORD}
      MSSQL_PID: Developer              # free license for development
    volumes:
      - sqlserver_data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "$$SA_PASSWORD" -Q "SELECT 1" || exit 1
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - app-network

  # ── Redis (if cache applies) ──────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: app-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: redis-cli ping | grep PONG
      interval: 10s
    networks:
      - app-network

volumes:
  sqlserver_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

---

## .env.example — Variables Template

```bash
# Copy to .env and fill in values — NEVER commit .env to the repository
# .gitignore must include: .env, .env.local, .env.production

# Database
DB_PASSWORD=YourStrongPassword123!

# JWT
JWT_SECRET=base64-of-at-least-256-bits-generated-with-openssl-rand-base64-32
JWT_EXPIRATION_MS=3600000

# Frontend
NEXT_PUBLIC_APP_NAME=My Application

# CI/CD only
DOCKER_REGISTRY=registry.company.com
SONAR_TOKEN=
```

---

## Docker Checklist — Before Generating

```
□ Does the Dockerfile use multi-stage build (builder + runtime)?
□ Is pom.xml/package.json copied BEFORE source code (cache)?
□ Does the final image run with a non-root user?
□ Is there a HEALTHCHECK in the Dockerfile?
□ Do secrets come from environment variables (never ARG/ENV with values)?
□ Is .env.example in the repo but .env in .gitignore?
□ Does docker-compose use depends_on with condition: service_healthy?
□ Do volumes persist DB data (not lost when recreating containers)?
□ Does the Next.js image use output: 'standalone' to optimize size?
