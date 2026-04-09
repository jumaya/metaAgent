# Orden de Ejecución de Agentes — FCV Backend

**Proyecto:** Fundación Cardiovascular — Migración VB6 → Java 21 Spring Boot  
**Alcance:** Solo backend `com.fcv` — Fase 1  
**Patrón:** Strangler Fig incremental, formulario por formulario

---

## Flujo Principal — Pipeline por Formulario VB6

```mermaid
graph TD
    START([Inicio: formulario VB6 seleccionado]) --> VB6

    VB6["@vb6-analyzer\nLee .frm y extrae\nlógica + SQL inline\n→ {form}-analysis.json"]
    SP["@sp-migration-planner\nClasifica SPs en A/B/C\nHC siempre A · cap 20% B\n→ {form}-sp-classification.json"]
    SQL["@sql-analyzer\nTraduce SQL/SPs á JPQL\no Spring Data methods\n→ {form}-sql-analysis.json"]
    REUSE["@reuse-tracker\nVerifica services-catalog.json\n¿REUSE / EXTEND / CREATE?\n→ decision incluida en contexto"]
    BE["@backend-generator\nGenera Command + Query +\nHandler + Entity + Repository\n+ Controller com.fcv\n→ código Java"]
    TRACE["@traceability-documenter\nActualiza master-index.json\nGenera ficha de trazabilidad\n→ {form}-traceability.md"]
    SEC["@security-reviewer\nOWASP · Ley 1581/2012\nRes. 866/2021 · roles FCV\n→ security-report.md"]
    TEST["@test-generator\nJUnit 5 + AuditoriaAcceso\nobligatorio para HC\n→ tests Java"]
    END([Estado: REVISADO → COMPLETADO])

    VB6 --> SP
    SP --> SQL
    SQL --> REUSE
    REUSE --> BE
    BE --> TRACE
    TRACE --> SEC
    SEC --> TEST
    TEST --> END

    style VB6 fill:#f0f4ff,stroke:#4a6cf7
    style SP fill:#fff4e6,stroke:#f97316
    style SQL fill:#fff4e6,stroke:#f97316
    style REUSE fill:#fff4e6,stroke:#f97316
    style BE fill:#e6f4ea,stroke:#22c55e
    style TRACE fill:#e6f4ea,stroke:#22c55e
    style SEC fill:#fef2f2,stroke:#ef4444
    style TEST fill:#fef2f2,stroke:#ef4444
    style END fill:#f0fdf4,stroke:#16a34a
```

---

## Flujo de Agentes de Infraestructura (una sola vez, al inicio)

```mermaid
graph LR
    A([Inicio del proyecto]) --> ORCH

    ORCH["@migration-orchestrator\nConfigura registros · define\nordenprioridad BCs\nVerifica stato VB6 en producción"]
    DOMAIN["@domain-mapper\nDefine los 5 BCs FCV\nconfiguracion · identidad\nadmision · historiaclinica · cartera"]
    CICD["@cicd-agent\nConfigura Jenkins on-premise\ncredenciales fcv- · pipeline\nbackend-only · SonarQube"]
    MON["@monitoring-agent\nDashboards Grafana fcv.*\nalertas p95>800ms\nHC accesos>100/min"]
    DOCKER["@docker-agent\nDockerfile multi-stage JRE\ndocker-compose.fcv.yml\nAPI+Redis+Prometheus+Grafana"]

    ORCH --> DOMAIN
    DOMAIN --> CICD
    CICD --> MON
    CICD --> DOCKER
    MON --> READY
    DOCKER --> READY

    READY([Infraestructura lista\nIniciar pipeline por formulario])

    style ORCH fill:#fdf4ff,stroke:#a855f7
    style DOMAIN fill:#fdf4ff,stroke:#a855f7
    style CICD fill:#f0f9ff,stroke:#0ea5e9
    style MON fill:#f0f9ff,stroke:#0ea5e9
    style DOCKER fill:#f0f9ff,stroke:#0ea5e9
    style READY fill:#f0fdf4,stroke:#16a34a
```

---

## Flujo de Estados del Master-Index

```mermaid
stateDiagram-v2
    [*] --> PENDIENTE : Formulario identificado en VB6

    PENDIENTE --> ANALIZANDO : @migration-orchestrator selecciona formulario\n@vb6-analyzer comienza análisis

    ANALIZANDO --> BACKEND_LISTO : @backend-generator completa código\n@traceability-documenter actualiza registro

    BACKEND_LISTO --> REVISADO : @security-reviewer + @test-generator completan\ntodos los checks sin hallazgos críticos

    REVISADO --> COMPLETADO : Tech Lead aprueba Pull Request\nCódigo mergeado a main

    REVISADO --> ERROR : security-reviewer encuentra hallazgo crítico\no test-generator falla con 0% cobertura HC

    BACKEND_LISTO --> BLOQUEADO : Cap 20% categoría B de SPs superado\no DDL no aditivo detectado

    BLOQUEADO --> ANALIZANDO : Tech Lead resuelve bloqueo

    ERROR --> ANALIZANDO : Desarrollador corrige y reinicia pipeline
```

---

## Dependencias entre Agentes

| Agente | Requiere que estén completos | Puede correr en paralelo con |
|--------|------------------------------|------------------------------|
| `@vb6-analyzer` | — (primer paso) | Otros formularios del mismo BC |
| `@sp-migration-planner` | `@vb6-analyzer` del mismo formulario | — |
| `@sql-analyzer` | `@sp-migration-planner` del mismo formulario | — |
| `@domain-mapper` | `@vb6-analyzer` (para confirmar BC) | `@sql-analyzer` |
| `@reuse-tracker` | `@sql-analyzer` del mismo formulario | — |
| `@backend-generator` | `@reuse-tracker` + `@domain-mapper` completos | — |
| `@traceability-documenter` | `@backend-generator` del mismo formulario | — |
| `@security-reviewer` | `@traceability-documenter` del mismo formulario | `@test-generator` |
| `@test-generator` | `@backend-generator` del mismo formulario | `@security-reviewer` |
| `@cicd-agent` | `@domain-mapper` (estructura de módulos definida) | `@monitoring-agent`, `@docker-agent` |
| `@monitoring-agent` | `@domain-mapper` (BCs definidos para dashboards) | `@cicd-agent`, `@docker-agent` |
| `@docker-agent` | `architecture-context.json` disponible | `@cicd-agent`, `@monitoring-agent` |

---

## Orden Recomendado de Bounded Contexts

Ejecutar formularios en este orden para minimizar dependencias entre módulos:

```
1. configuracion   → Catálogos maestros. Sin dependencias. Habilita CUPs, CIEs, EPS para los demás.
2. identidad       → Usuarios, roles, AuditoriaAcceso. Todos los demás BCs dependen de identidad.
3. admision        → Paciente, Atencion, Cita, CamaMovimiento. ~40 formularios, mayor volumen.
4. historiaclinica → HC, EventoClinico, Certificados. Regulación más estricta (Res. 866/2021).
5. cartera         → Factura, Cobro. Requiere admision completo.
```

---

## Reglas de Ejecución Críticas

| Regla | Detalle |
|-------|---------|
| **HC siempre A** | Formularios `frmHC*` → todos sus SPs son Categoría A. `@sp-migration-planner` no puede clasificarlos como B o C. |
| **Cap 20% Cat B** | Si el módulo supera el 20% de SPs en Categoría B → BLOQUEADO, requiere decisión del Tech Lead. |
| **DDL solo aditivo** | `@backend-generator` nunca genera `DROP TABLE`, `ALTER TABLE DROP COLUMN`, ni `ALTER COLUMN`. Si es necesario → BLOQUEADO. |
| **AuditoriaAcceso antes de devolver HC** | Todo endpoint que devuelva datos de historia clínica debe registrar en `AuditoriaAcceso` ANTES de retornar la respuesta. `@security-reviewer` valida esto. |
| **Consultar services-catalog.json primero** | `@reuse-tracker` siempre verifica `migration-registry/services-catalog.json` antes de generar. Si el servicio existe → REUSE, no CREATE. |
| **Domain Event al completar Command** | `@backend-generator` publica un Domain Event al final de cada CommandHandler exitoso. |
| **Tests de AuditoriaAcceso obligatorios** | Para cualquier formulario del módulo `historiaclinica`, `@test-generator` DEBE generar el test de verificación de auditoría. Sin este test → no se puede marcar REVISADO. |

---

## Skills Activos por Fase

```
ANALIZANDO:
  > backend/ddd-patterns.skill.md      → @vb6-analyzer, @domain-mapper
  > data/sql-to-jpql.skill.md          → @sp-migration-planner, @sql-analyzer  
  > backend/jpa-patterns.skill.md      → @sp-migration-planner, @sql-analyzer

BACKEND_LISTO:
  > backend/ddd-patterns.skill.md      → @backend-generator
  > backend/cqrs-patterns.skill.md     → @backend-generator
  > backend/jpa-patterns.skill.md      → @backend-generator
  > backend/exception-handling.skill.md → @backend-generator
  > architecture/modular-monolith-patterns.skill.md → @backend-generator, @reuse-tracker
  > architecture/adr-patterns.skill.md → @domain-mapper

REVISADO:
  > quality/security-checklist.skill.md → @security-reviewer
  > backend/spring-security.skill.md    → @security-reviewer
  > quality/test-patterns.skill.md      → @test-generator

INFRAESTRUCTURA:
  > infra/docker-patterns.skill.md     → @docker-agent, @cicd-agent
```
