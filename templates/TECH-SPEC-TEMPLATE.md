# Tech Spec — {Nombre del Proyecto}

**Versión:** 1.0
**Autor:** ARC-Vision (Arquitecto de Sistemas)
**Fecha:** {YYYY-MM-DD}
**Estado:** Borrador | En revisión | Aprobado

---

## 1. Contexto y Problema

### 1.1 ¿Qué es este sistema?
{Descripción del sistema en 2-3 párrafos. Enfocarse en el problema de negocio
que resuelve, no en la tecnología. Debe ser entendible por alguien no técnico.}

### 1.2 ¿Por qué ahora?
{Qué driver de negocio o técnico motiva este proyecto: deuda técnica,
crecimiento, nuevas funcionalidades, fin de soporte del legado, etc.}

### 1.3 Alcance
**Incluye:**
- {Lo que sí construiremos}

**No incluye (explícitamente fuera de alcance):**
- {Lo que NO construiremos en esta fase}

---

## 2. Objetivos y No-Objetivos

### Objetivos
| # | Objetivo | Métrica de éxito |
|---|----------|-----------------|
| 1 | {Objetivo técnico o de negocio} | {cómo sabremos que lo logramos} |

### No-Objetivos
- {Qué explícitamente NO buscamos lograr}

---

## 3. Arquitectura del Sistema

### 3.1 Vista de Alto Nivel (C4 — Nivel 2: Containers)

```mermaid
graph TB
  subgraph "Usuarios"
    USER_INT[Usuarios Internos]
    USER_EXT[Usuarios Externos]
  end

  subgraph "Sistema — {Nombre}"
    FE[Frontend\n{Framework}\n{Puerto}]
    BE[Backend\n{Framework}\n{Puerto}]
    DB[({Nombre DB}\n{Engine})]
    CACHE[({Cache}\nRedis)]
  end

  subgraph "Servicios Externos"
    AUTH[Autenticación\n{Provider}]
    MON[Monitoreo\n{Stack}]
  end

  USER_INT --> FE
  USER_EXT --> FE
  FE --> BE
  BE --> DB
  BE --> CACHE
  BE --> AUTH
  BE --> MON
```

### 3.2 Patrón Arquitectural
**Patrón:** {Monolito Modular | Microservicios | etc.}
**Razón:** {por qué este patrón para este proyecto}

### 3.3 Bounded Contexts / Módulos Principales
| Módulo/BC | Responsabilidad | Entidades Principales |
|-----------|----------------|----------------------|
| {Nombre} | {qué gestiona} | {entidad1, entidad2} |

---

## 4. Stack Tecnológico

| Capa | Tecnología | Versión | Justificación |
|------|-----------|---------|---------------|
| Backend | {Framework} | {X.Y} | {razón} |
| Frontend | {Framework} | {X.Y} | {razón} |
| Base de Datos | {Engine} | {X.Y} | {razón} |
| ORM | {ORM} | {X.Y} | {razón} |
| Caché | {Engine} | {X.Y} | {razón} |
| Autenticación | {Provider} | {X.Y} | {razón} |
| CI/CD | {Herramienta} | - | {razón} |
| Monitoreo | {Stack} | - | {razón} |

---

## 5. Modelo de Datos (Entidades Principales)

### 5.1 Diagrama Entidad-Relación (principales)
```mermaid
erDiagram
  {ENTIDAD_A} {
    int id PK
    string campo1
    string campo2
  }
  {ENTIDAD_B} {
    int id PK
    int entidadAId FK
    string campo1
  }
  {ENTIDAD_A} ||--o{ {ENTIDAD_B} : "tiene"
```

### 5.2 Estrategia de Persistencia
{Descripción de cómo se almacenan los datos, particionamiento si aplica,
estrategia de backup, etc.}

---

## 6. Seguridad

### 6.1 Autenticación
- **Estrategia:** {JWT | OAuth2 | etc.}
- **Provider:** {Spring Security | Keycloak | Auth0 | etc.}
- **Almacenamiento de tokens:** {httpOnly cookie | etc.}
- **Expiración de tokens:** {tiempo}

### 6.2 Autorización
- **Modelo:** RBAC (Role-Based Access Control)
- **Roles definidos:** {lista de roles}
- **Implementación:** {cómo se aplica en el código}

### 6.3 Compliance
{Si aplica: GDPR, PCI-DSS, HIPAA, etc. y cómo se cumple}

### 6.4 Otros controles de seguridad
- HTTPS/TLS en todos los ambientes
- Secrets en variables de entorno (nunca hardcodeados)
- CORS restrictivo
- Rate limiting en APIs públicas
- {Otros según el proyecto}

---

## 7. Infraestructura y Despliegue

### 7.1 Ambientes
| Ambiente | Propósito | URL | Aprobación requerida |
|----------|-----------|-----|---------------------|
| Development | Desarrollo local | localhost | - |
| Staging | QA y pruebas | staging.{dominio} | Tech Lead |
| Production | Producción | {dominio} | Tech Lead + PO |

### 7.2 Pipeline CI/CD
```
Push/PR → Build → Tests → Quality Gate → [Aprobación] → Deploy
```
**Herramienta:** {Jenkins | GitHub Actions | etc.}

### 7.3 Contenedores
**Estrategia:** {Docker + {orquestador}}
{Descripción de la estrategia de contenedores}

---

## 8. Observabilidad

| Pilar | Herramienta | Qué monitorea |
|-------|------------|---------------|
| Métricas | {Prometheus/Grafana | DataDog | etc.} | {qué métricas} |
| Logs | {ELK | CloudWatch | etc.} | {qué se loguea} |
| Trazas | {Jaeger | OpenTelemetry | etc.} | {qué se traza} |
| Alertas | {Grafana Alerts | PagerDuty | etc.} | {qué dispara alertas} |

### Métricas de Negocio Clave
| Métrica | Descripción | Alerta si |
|---------|-------------|-----------|
| {nombre} | {qué mide} | {umbral de alerta} |

---

## 9. Decisiones Clave (Resumen de ADRs)

| ADR | Decisión | Estado |
|-----|----------|--------|
| ADR-001 | {título breve} | Aceptado |
| ADR-002 | {título breve} | Aceptado |

{Ver carpeta /ADRs/ para el detalle completo de cada decisión}

---

## 10. Riesgos y Mitigaciones

| # | Riesgo | Probabilidad | Impacto | Mitigación |
|---|--------|-------------|---------|------------|
| 1 | {descripción} | Alta/Media/Baja | Alto/Medio/Bajo | {plan} |

---

## 11. Plan de Implementación por Fases

| Fase | Descripción | Duración Est. | Entregables |
|------|-------------|---------------|-------------|
| 0 | Setup de proyecto e infraestructura base | {N} semanas | Repo, CI/CD, ambientes |
| 1 | {Primer módulo/dominio} | {N} semanas | {entregables} |
| 2 | {Segundo módulo/dominio} | {N} semanas | {entregables} |
| N | {Último módulo + hardening} | {N} semanas | Sistema completo en producción |

---

## Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| Arquitecto | ARC-Vision | {fecha} | ✅ |
| Tech Lead | | | |
| Product Owner | | | |
