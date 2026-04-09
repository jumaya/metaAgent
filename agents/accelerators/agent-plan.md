# Plan de Agentes — Fundación Cardiovascular (FCV)
**Generado por:** ARC-Meta-Agent  
**Basado en:** `architecture-context.json` producido por ARC-Vision  
**Fecha:** 2026-03-12  
**Alcance:** Solo backend — Java 21 + Spring Boot 3.x + `com.fcv`

---

## Resumen

| Métrica | Valor |
|---------|-------|
| Total de agentes activos | 13 |
| Agentes de análisis | 4 (vb6-analyzer, sp-migration-planner, sql-analyzer, domain-mapper) |
| Agentes de generación | 4 (reuse-tracker, backend-generator, traceability-documenter, migration-orchestrator) |
| Agentes de calidad | 2 (security-reviewer, test-generator) |
| Agentes de infraestructura | 3 (cicd-agent, monitoring-agent, docker-agent) |
| Agentes fuera de alcance (fas 1) | 3 (component-library-agent, frontend-analyzer, nextjs-generator) |

---

## Estado de la Base de Agentes

**Base encontrada:** Sí — carpeta de agentes de referencia disponible (proyecto anterior).  
**Resultado de clasificación:**

| Archivo | Decisión | Razón |
|---------|----------|-------|
| `backend-generator.instructions.md` | **ADAPT** | Paquete genérico → `com.fcv`; comentarios en inglés → español; faltaban módulos FCV, cumplimiento Resolución 866/2021, AuditoriaAcceso, Micrometer |
| `sql-analyzer.instructions.md` | **ADAPT** | Sin estrategia A/B/C de SPs (ADR-004); faltaba `context_forward → sp-migration-planner`; sin anti-patrones T-SQL |
| `reuse-tracker.instructions.md` | **ADAPT** | Tabla de BCs genérica → 5 BCs FCV; faltaba regla "AuditoriaAcceso nunca duplicar"; sin prioridad `configuracion` |
| `vb6-analyzer.instructions.md` | **ADAPT** | Mapeo de formularios genérico → módulos FCV (`frmAdm*`, `frmHC*`, etc.); `context_forward → sp-migration-planner` (no a sql-analyzer directamente) |
| `migration-orchestrator.instructions.md` | **ADAPT** | Removido todo lo de Next.js/frontend; pipeline de 8 pasos backend-only FCV; flujo de estados sin FRONTEND_LISTO |
| `security-reviewer.instructions.md` | **ADAPT** | Removido frontend; añadida Ley 1581/2012, Resolución 866/2021; roles FCV específicos |
| `traceability-documenter.instructions.md` | **ADAPT** | `context_forward → security-reviewer` (no frontend-analyzer); ejemplos reales FCV |
| `test-generator.instructions.md` | **ADAPT** | Removido Jest; añadido patrón AuditoriaAcceso obligatorio en HC; `@DisplayName` en español |
| `monitoring-agent.instructions.md` | **ADAPT** | Prefijo métricas `fcv.*`; panel HC accesos (Res. 866/2021); umbral p95 → 800ms; alerta acceso anómalo HC > 100/min |
| `domain-mapper.instructions.md` | **ADAPT** | BCs en español; prefijo `frmAdm*` → `com.fcv.admision`; Res. 866/2021 para `historiaclinica` |
| `cicd-agent.instructions.md` | **ADAPT** | Jenkins ya instalado on-premise (no instalar); solo backend; credenciales `fcv-`; sin pipeline frontend |
| `sp-migration-planner.instructions.md` | **NEW** | No existía en la base. Creado desde cero para ADR-004: clasificador A/B/C, cap 20% categoría B, HC siempre A |
| `docker-agent.instructions.md` | **NEW** | No existía en la base. Creado para infraestructura on-premise FCV: multi-stage JRE-only, `docker-compose.fcv.yml` |
| `component-library-agent.instructions.md` | **USE_AS_IS** | ⚠️ **FUERA DE ALCANCE** — Frontend Next.js es fase posterior |
| `frontend-analyzer.instructions.md` | **USE_AS_IS** | ⚠️ **FUERA DE ALCANCE** — Frontend es fase posterior |
| `nextjs-generator.instructions.md` | **USE_AS_IS** | ⚠️ **FUERA DE ALCANCE** — Frontend es fase posterior |

---

## Agentes por Fase

### Fase SETUP — Inicialización (ejecutar una sola vez al inicio del proyecto)

| Agente | Archivo | Origen | Justificación |
|--------|---------|--------|---------------|
| `@migration-orchestrator` | `migration-orchestrator.instructions.md` | ADAPT | Coordina todos los demás agentes; mantiene `master-index.json` actualizado |
| `@domain-mapper` | `domain-mapper.instructions.md` | ADAPT | Define los 5 BCs FCV antes de generar el primer CommandHandler |
| `@cicd-agent` | `cicd-agent.instructions.md` | ADAPT | Configura Jenkins on-premise existente con credenciales `fcv-`; pipeline backend-only |
| `@monitoring-agent` | `monitoring-agent.instructions.md` | ADAPT | Configura dashboards Grafana con métricas `fcv.*` y alertas críticas HC |
| `@docker-agent` | `docker-agent.instructions.md` | NEW | Genera `Dockerfile` + `docker-compose.fcv.yml` para entorno on-premise |

### Fase ANALIZANDO — Por formulario VB6

| Agente | Archivo | Paso | Origen | Justificación |
|--------|---------|------|--------|---------------|
| `@vb6-analyzer` | `vb6-analyzer.instructions.md` | 1 | ADAPT | Lee `.frm` y extrae lógica, SQL inline, módulo FCV destino |
| `@sp-migration-planner` | `sp-migration-planner.instructions.md` | 2 | NEW | Clasifica SPs del formulario en A/B/C (ADR-004); HC siempre A |
| `@sql-analyzer` | `sql-analyzer.instructions.md` | 3 | ADAPT | Traduce SQL/SPs categorizados a JPQL o Spring Data methods |
| `@domain-mapper` | `domain-mapper.instructions.md` | 4 | ADAPT | Confirma o refina el BC al que pertenece el formulario |

### Fase BACKEND_LISTO — Por formulario VB6

| Agente | Archivo | Paso | Origen | Justificación |
|--------|---------|------|--------|---------------|
| `@reuse-tracker` | `reuse-tracker.instructions.md` | 5 | ADAPT | Verifica si el servicio ya existe en `services-catalog.json` antes de generar |
| `@backend-generator` | `backend-generator.instructions.md` | 6 | ADAPT | Genera Command/Query/Handler/Entity/Repository/Controller con `com.fcv` |
| `@traceability-documenter` | `traceability-documenter.instructions.md` | 7 | ADAPT | Actualiza `master-index.json` y genera ficha de trazabilidad del formulario |

### Fase REVISADO — Por formulario VB6

| Agente | Archivo | Paso | Origen | Justificación |
|--------|---------|------|--------|---------------|
| `@security-reviewer` | `security-reviewer.instructions.md` | 8a | ADAPT | Verifica OWASP, Ley 1581/2012, Res. 866/2021 en el código generado |
| `@test-generator` | `test-generator.instructions.md` | 8b | ADAPT | Genera JUnit 5 + tests AuditoriaAcceso obligatorios para HC |

### Fase FRONTEND_LISTO — ⚠️ FUERA DE ALCANCE Fase 1

| Agente | Estado | Nota |
|--------|--------|------|
| `@component-library-agent` | DETENIDO | Next.js frontend es una fase posterior |
| `@frontend-analyzer` | DETENIDO | Idem |
| `@nextjs-generator` | DETENIDO | Idem |

---

## Orden Recomendado de Bounded Contexts

1. **`configuracion`** — Catálogos maestros (CUPs, CIEs, EPS, municipios, tarifas). Sin dependencias de otros módulos. Desbloquea a todos los demás.
2. **`identidad`** — Usuarios, roles, AuditoriaAcceso. Todos los demás BCs dependen de identidad para seguridad.
3. **`admision`** — Paciente, Atencion, Cita, CamaMovimiento. Mayor volumen de formularios (≈ 40).
4. **`historiaclinica`** — Historia clínica, EventoClinico, Certificados. Regulación más estricta (Res. 866/2021). Requiere identidad completo.
5. **`cartera`** — Factura, Cobro, Cartera. Requiere admision completo.

---

## Agentes Descartados y Por Qué

| Agente del Catálogo | Razón para no incluir |
|---------------------|----------------------|
| `api-gateway-generator` | No hay microservicios — arquitectura monolito modular (ADR-002) |
| `event-handler-generator` | No hay motor de mensajería en esta fase (RabbitMQ/Kafka fuera de alcance) |
| `cache-layer-generator` | El caché Redis es simple (TTL fijo de catálogos); no requiere agente dedicado — `backend-generator` lo cubre |
| `kubernetes-agent` | Infraestructura on-premise con Docker Compose; no K8s |
| `iac-agent` | No hay IaC (Terraform/Pulumi); on-premise sin cloud |
| `cloud-agnostic-enforcer` | On-premise total; no hay riesgo de lock-in a cloud |
| `data-migration-agent` | BD SQL Server **compartida** con VB6 durante toda la transición — NO se migran datos, solo DDL aditivo |
| `report-migration-agent` | Reportes (Crystal Reports) están explícitamente **fuera de alcance** de esta fase |
| `react-spa-generator` | Stack frontend es Next.js App Router (cuando llegue la fase 2) |
| `angular-generator` | No aplica — no hay Angular en el stack |
| `mobile-generator` | No hay aplicación móvil |
| `performance-agent` | Equipo SoftwareOne con experiencia; `test-generator` cubre performance básico. Revisar si escala lo requiere. |

---

## Decisiones Clave que Afectan a los Agentes

| Decisión | Impacto en Agentes |
|----------|-------------------|
| **ADR-001**: Strangler Fig | `@migration-orchestrator` mantiene VB6 vivo; no hay corte abrupto |
| **ADR-002**: Monolito Modular + DDD + CQRS | `@backend-generator` siempre genera Commands + Queries separados. `@domain-mapper` define BCs. |
| **ADR-003**: DDL solo aditivo | `@backend-generator` nunca genera `DROP` ni `ALTER ... DROP`. `@security-reviewer` valida esto. |
| **ADR-004**: SP Estrategia A/B/C | `@sp-migration-planner` clasifica cada SP antes de que `@sql-analyzer` lo traduzca. Cap 20% Cat B. HC siempre A. |
| **Resolución 866/2021** | `@backend-generator` registra en `AuditoriaAcceso` antes de devolver HC. `@test-generator` genera test obligatorio de auditoría. `@security-reviewer` valida. |
| **Ley 1581/2012** | `@security-reviewer` verifica que datos sensibles nunca aparezcan en logs. |
| **Redis para catálogos** | `@backend-generator` usa `@Cacheable` en servicios de `configuracion`. TTL 24h. |
