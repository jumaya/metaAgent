# Guía de Uso — Agentes de Migración FCV en VS Code + GitHub Copilot
**Proyecto:** Fundación Cardiovascular — Migración Backend VB6 → Java Spring Boot 3.x
**Generado por:** ARC-Meta-Agent | **Fase:** Solo Backend (117 formularios)
**Paquete base:** `com.fcv` | **5 bounded contexts:** admision, historiaclinica, cartera, configuracion, identidad

## Requisitos Previos

1. **VS Code** con la extensión **GitHub Copilot** y **GitHub Copilot Chat** instaladas
2. Cuenta con GitHub Copilot activa (Individual, Business o Enterprise)
3. El repositorio debe tener:
   - `.github/agents/` con los custom agents (`.agent.md`) — aparecen en el dropdown de Copilot
   - `.github/instructions/` con los acelerators (`.instructions.md`) — se invocan con `#file:`
   - `.github/skills/` con los skills especializados — se añaden con `#file:` para enriquecer cada agente

---

## Cómo Funciona el Sistema

GitHub Copilot lee automáticamente `.github/copilot-instructions.md` como contexto global.
Los **custom agents** (`.agent.md` en `.github/agents/`) aparecen directamente en el
dropdown de agentes del Chat de Copilot.
Los **acelerators** (`.instructions.md` en `.github/instructions/`) se invocan
manualmente en el **Copilot Chat** usando `@workspace` y referenciando el archivo con `#file:`.
Los **skills** (`.skill.md` en `.github/skills/`) se añaden junto al acelerator para
aportar conocimiento especializado (patrones DDD, JPA, seguridad OWASP, etc.).

### Abrir Copilot Chat
- **Atajo:** `Ctrl+Alt+I` (Windows/Linux) o `Cmd+Alt+I` (Mac)
- **Menú:** View → Copilot Chat

---

## Comandos de Uso por Agente

### 🎯 Inicio del Proyecto FCV (ejecutar una sola vez)

```bash
# Paso 1 — Escaneo inicial de los 117 formularios VB6
@workspace #file:.github/instructions/migration-orchestrator.instructions.md \
           #file:.github/agents/accelerators/master-index.json
Inicializa el proyecto de migración FCV. Escanea todos los formularios VB6
en la carpeta legacy/forms/ y construye el master-index.json

# Paso 2 — Configurar pipelines en Jenkins on-premise FCV
# (Jenkins ya está instalado — solo configura el proyecto)
@workspace #file:.github/instructions/cicd-agent.instructions.md \
           #file:.github/skills/infra/docker-patterns.skill.md
Configura los pipelines FCV en Jenkins existente. Genera los Jenkinsfiles
para backend CI/CD y el README de configuración del proyecto.

# Paso 3 — Configurar monitoreo FCV
@workspace #file:.github/instructions/monitoring-agent.instructions.md \
           #file:.github/skills/infra/docker-patterns.skill.md
Configura el stack de monitoreo FCV: Prometheus + Grafana + Micrometer.
Genera docker-compose.fcv.yml y los 3 dashboards base (HTTP, métricas de negocio,
progreso de migración). Incluye panel fcv.historiaclinica.accesos (Res. 866/2021).

# Paso 4 — Configurar Docker on-premise
@workspace #file:.github/instructions/docker-agent.instructions.md \
           #file:.github/skills/infra/docker-patterns.skill.md
Genera Dockerfile multi-stage para el backend FCV y docker-compose.fcv.yml
con todos los servicios: API + Redis + Prometheus + Grafana.
```

### 🔄 Migración por Bounded Context FCV (el flujo principal)

```bash
# Migrar un bounded context completo
@workspace #file:.github/instructions/migration-orchestrator.instructions.md \
           #file:.github/agents/accelerators/master-index.json
Migra el bounded context admision. Agrupa formularios frmAdm* 
y migra en orden de dependencias.

# Migrar un formulario específico FCV
@workspace #file:.github/instructions/migration-orchestrator.instructions.md \
           #file:.github/agents/accelerators/master-index.json
Migra el formulario frmAdmAtencion. El archivo está en legacy/forms/frmAdmAtencion.frm

# Pipeline completo para un formulario (manual paso a paso):
# 1. VB6 → 2. SP Plan → 3. SQL → 4. Reuse → 5. Backend → 6. Traza → 7. Security → 8. Tests
```

**Flujo FCV por formulario:**
```
frmAdmAtencion.frm
  ↓
@vb6-analyzer      → frmAdmAtencion-analysis.json
  ↓
@sp-migration-planner → frmAdmAtencion-sp-classification.json  (ADR-004: A/B/C)
  ↓
@sql-analyzer      → frmAdmAtencion-sql-mapping.json
  ↓
@reuse-tracker     → frmAdmAtencion-reuse-plan.json
  ↓
@backend-generator → com.fcv.admision.application.commands.AdmitirPacienteCommandHandler
  ↓
@traceability-documenter → migration-registry/forms/frmAdmAtencion.md
  ↓
@security-reviewer → verifica Ley 1581/2012 + Res. 866/2021
  ↓
@test-generator    → AdmitirPacienteCommandHandlerTest.java
  ↓
Estado: BACKEND_LISTO → COMPLETADO
```

### 📊 Control y Monitoreo

```bash
# Ver estado completo (incluye step counts y bloqueados)
@workspace #file:.github/instructions/migration-orchestrator.instructions.md \
           #file:.github/agents/accelerators/master-index.json
Estado de la migración

# Desbloquear un agente que superó su step limit
@workspace #file:.github/instructions/migration-orchestrator.instructions.md \
           #file:.github/agents/accelerators/master-index.json
Desbloquea frmClientes::sql-analyzer

# Ver qué agentes están registrados y sus step limits
@workspace #file:.github/agents/accelerators/agent-registry.json
Muéstrame los step limits de todos los agentes del pipeline
```

### 🔍 Usar Agentes Individualmente — FCV (modo avanzado)

Puedes invocar agentes directamente si quieres más control:

```bash
# Solo análisis VB6
@workspace #file:.github/instructions/vb6-analyzer.instructions.md
Analiza este formulario: #file:legacy/forms/frmAdmAtencion.frm

# Solo clasificación de stored procedures (ADR-004 A/B/C)
@workspace #file:.github/instructions/sp-migration-planner.instructions.md \
           #file:.github/skills/data/sql-to-jpql.skill.md
Clasifica los SPs de frmAdmAtencion según estrategia ADR-004.
Análisis: #file:migration-registry/forms/frmAdmAtencion-analysis.json

# Solo análisis SQL → JPQL (con skill especializado)
@workspace #file:.github/instructions/sql-analyzer.instructions.md \
           #file:.github/skills/data/sql-to-jpql.skill.md
Transforma las queries de #file:migration-registry/forms/frmAdmAtencion-analysis.json
Catálogo de SPs clasificados: #file:migration-registry/forms/frmAdmAtencion-sp-classification.json

# Solo detección de reutilización entre BCs
@workspace #file:.github/instructions/reuse-tracker.instructions.md
Detecta qué puede reutilizarse para frmAdmAtencion.
SQL mapping: #file:migration-registry/forms/frmAdmAtencion-sql-mapping.json
Catálogo de servicios: #file:.github/agents/accelerators/services-catalog.json

# Solo generación de backend Java FCV (con skills DDD + JPA + CQRS)
@workspace #file:.github/instructions/backend-generator.instructions.md \
           #file:.github/skills/backend/ddd-patterns.skill.md \
           #file:.github/skills/backend/cqrs-patterns.skill.md \
           #file:.github/skills/backend/jpa-patterns.skill.md \
           #file:.github/skills/backend/exception-handling.skill.md
Genera el código Java com.fcv.admision para frmAdmAtencion.
Plan de reutilización: #file:migration-registry/forms/frmAdmAtencion-reuse-plan.json
Dominio: #file:migration-registry/domains/admision.md

# Solo mapeo de dominio DDD
@workspace #file:.github/instructions/domain-mapper.instructions.md \
           #file:.github/skills/backend/ddd-patterns.skill.md \
           #file:.github/skills/architecture/modular-monolith-patterns.skill.md
Mapea el dominio de frmAdmAtencion al bounded context admision de FCV.
Análisis: #file:migration-registry/forms/frmAdmAtencion-analysis.json
BC existente: #file:migration-registry/domains/admision.md

# Revisión de seguridad (Ley 1581/2012 + Resolución 866/2021)
@workspace #file:.github/instructions/security-reviewer.instructions.md \
           #file:.github/skills/quality/security-checklist.skill.md \
           #file:.github/skills/backend/spring-security.skill.md
Revisa la seguridad del código generado para frmAdmAtencion.
Trazabilidad: #file:migration-registry/forms/frmAdmAtencion.md

# Generación de tests unitarios FCV
@workspace #file:.github/instructions/test-generator.instructions.md \
           #file:.github/skills/quality/test-patterns.skill.md
Genera los tests para AdmitirPacienteCommandHandler de FCV.
Análisis original: #file:migration-registry/forms/frmAdmAtencion-analysis.json

# Documentación de trazabilidad
@workspace #file:.github/instructions/traceability-documenter.instructions.md
Documenta la trazabilidad de frmAdmAtencion → com.fcv.admision.
Análisis: #file:migration-registry/forms/frmAdmAtencion-analysis.json
```

---

## Flujo Completo FCV Paso a Paso

### Semana 1 — Setup del Proyecto FCV

```
Día 1:
  1. Clonar el repositorio con los agentes
  2. Abrir Copilot Chat → ejecutar escaneo inicial de 117 formularios
  3. Revisar master-index.json generado → ajustar bounded contexts si es necesario
  4. Aprobar el plan de migración
  5. Ejecutar cicd-agent → configura el proyecto en Jenkins on-premise existente

Día 2:
  6. Ejecutar docker-agent → genera Dockerfile + docker-compose.fcv.yml
  7. Ejecutar monitoring-agent → levanta stack Prometheus + Grafana
  8. Verificar dashboard fcv.historiaclinica.accesos (Res. 866/2021)

Día 3+:
  9. Comenzar migración bounded context por bounded context
  10. Orden recomendado: configuracion → identidad → admision → historiaclinica → cartera
```

### Por Cada Bounded Context FCV

```
1. Invocar migration-orchestrator con el nombre del bounded context
2. Revisar el plan presentado (orden de formularios, dependencias)
3. Aprobar con "sí, procede"
4. Los agentes trabajan formulario por formulario automáticamente
   (vb6-analyzer → sp-migration-planner → sql-analyzer → reuse-tracker
    → backend-generator → traceability-documenter → security-reviewer → test-generator)
5. Revisar código Java generado en com.fcv.{modulo}/
6. Corregir hallazgos de security-reviewer
   (especial atención a Ley 1581/2012 y Resolución 866/2021 en historiaclinica)
7. Confirmar estado BACKEND_LISTO → COMPLETADO en master-index.json
```

### Orden de Bounded Contexts Recomendado FCV

```
1. configuracion   → Catálogos base: CUPs, CIEs, EPS, municipios, tarifas
                     (sin dependencias — base para todos los demás BC)
2. identidad       → Usuarios, Roles, AuditoriaAcceso
                     (requerido por historiaclinica para cumplimiento Res. 866/2021)
3. admision        → Atencion, Paciente, Cita, CamaMovimiento, Remision
                     (BC central del sistema hospitalario)
4. historiaclinica → HistoriaClinica, EventoClinico, Certificado, Autorizacion
                     (CRÍTICO: máxima regulación — requiere identidad y admision)
5. cartera         → Factura, Cobro, CarteraMovimiento
                     (requiere admision para referencia a Atencion)
```


---

## Tips y Buenas Prácticas FCV

### Proveer contexto suficiente
Copilot trabaja mejor cuando le das múltiples archivos como contexto:
```bash
# Bien — con contexto completo + skills
@workspace #file:.github/instructions/backend-generator.instructions.md \
           #file:.github/skills/backend/ddd-patterns.skill.md \
           #file:.github/skills/backend/jpa-patterns.skill.md \
           #file:migration-registry/forms/frmAdmAtencion-reuse-plan.json \
           #file:migration-registry/forms/frmAdmAtencion-sql-mapping.json \
           #file:migration-registry/domains/admision.md
Genera el código backend para frmAdmAtencion

# Mal — sin contexto suficiente
@workspace Genera el backend para frmAdmAtencion
```

### Verificar el registro antes de cada sesión
```bash
# Revisar estado actual antes de continuar
@workspace #file:.github/instructions/migration-orchestrator.instructions.md
Dame un reporte del estado actual de la migración FCV.
Registro: #file:.github/agents/accelerators/master-index.json
```

### sp-migration-planner — cuándo usarlo directamente
```bash
# Si el vb6-analyzer reporta muchos SPs UNKNOWN o complejos
@workspace #file:.github/instructions/sp-migration-planner.instructions.md \
           #file:.github/skills/data/sql-to-jpql.skill.md
Reclasifica manualmente los SPs de frmAdmAtencionCoc que contienen
lógica de cierre de cuenta compleja.
Análisis: #file:migration-registry/forms/frmAdmAtencionCoc-analysis.json
```

### Regla de oro: DDL Additive Only
```
⚠️ La BD SQL Server es compartida con el sistema VB6 en producción.
NINGÚN agente debe generar:
  ❌ DROP TABLE, ALTER TABLE DROP COLUMN, ALTER TABLE ALTER COLUMN
  ✅ Solo: CREATE TABLE, ALTER TABLE ADD COLUMN (con DEFAULT o nullable)
Si ves DDL destructivo en el código generado → rechazar y reportar al Tech Lead.
```

### Cuando Copilot no sigue las instrucciones
```bash
# Recordarle las reglas FCV específicas
@workspace El código generado no sigue las instrucciones de FCV.
#file:.github/instructions/backend-generator.instructions.md
#file:.github/copilot-instructions.md
Por favor revisa estas reglas y corrige: [describir el problema]
```

### Guardar el progreso
Los archivos JSON del registro son la fuente de verdad.
Commitear frecuentemente:
```bash
git add migration-registry/
git commit -m "chore: migración BC admision completada (23 formularios frmAdm*)"
```

---

## Estructura Final del Repositorio FCV

```
.
├── .github/
│   ├── copilot-instructions.md              ← Contexto global FCV (carga automática)
│   ├── agents/
│   │   ├── arc-vision.agent.md
│   │   ├── arc-meta-agent.agent.md
│   │   └── accelerators/
│   │       ├── master-index.json            ← Registro maestro (117 formularios)
│   │       ├── services-catalog.json        ← Catálogo de servicios com.fcv creados
│   │       └── USAGE-GUIDE.md
│   ├── instructions/                        ← Acelerators FCV personalizados
│   │   ├── backend-generator.instructions.md       ✅
│   │   ├── cicd-agent.instructions.md              ✅ Jenkins on-premise, solo backend
│   │   ├── docker-agent.instructions.md            ✅ NUEVO
│   │   ├── domain-mapper.instructions.md           ✅
│   │   ├── migration-orchestrator.instructions.md  ✅ solo backend
│   │   ├── monitoring-agent.instructions.md        ✅ métricas FCV
│   │   ├── reuse-tracker.instructions.md           ✅
│   │   ├── security-reviewer.instructions.md       ✅ Ley 1581 + Res. 866/2021
│   │   ├── sp-migration-planner.instructions.md    ✅ NUEVO — ADR-004
│   │   ├── sql-analyzer.instructions.md            ✅
│   │   ├── test-generator.instructions.md          ✅ AuditoriaAcceso tests
│   │   ├── traceability-documenter.instructions.md ✅
│   │   └── vb6-analyzer.instructions.md            ✅
│   └── skills/
│       ├── backend/ (ddd, cqrs, jpa, spring-security, exception-handling)
│       ├── data/ (sql-to-jpql)
│       ├── quality/ (test-patterns, security-checklist)
│       ├── infra/ (docker-patterns)
│       └── architecture/ (adr-patterns, modular-monolith-patterns)
├── legacy/                                  ← Código fuente VB6 (solo lectura)
│   ├── forms/                               ← 117 formularios .frm
│   ├── modules/
│   └── classes/
├── migration-registry/
│   ├── domains/ (admision, historiaclinica, cartera, configuracion, identidad)
│   └── forms/
│       ├── {frmX}-analysis.json
│       ├── {frmX}-sp-classification.json    ← NUEVO (sp-migration-planner ADR-004)
│       ├── {frmX}-sql-mapping.json
│       ├── {frmX}-reuse-plan.json
│       └── {frmX}.md
├── src/main/java/com/fcv/               ← Backend Spring Boot generado
│   ├── admision/ (domain/model, application/commands, application/queries, infrastructure/rest)
│   ├── historiaclinica/
│   ├── cartera/
│   ├── configuracion/
│   └── identidad/
├── docs/ (tech-spec, architecture.mermaid, executive-summary, ADRs ADR-001..ADR-004)
├── jenkins/
│   ├── setup/ (required-plugins.txt, README-fcv-setup.md)
│   └── backend/ (Jenkinsfile.ci, Jenkinsfile.cd)
├── monitoring/
│   ├── prometheus/ (prometheus.yml, alerts.yml)
│   └── grafana/dashboards/ (backend-overview, business-metrics*, migration-progress)
│       * business-metrics incluye panel fcv.historiaclinica.accesos (Res. 866/2021)
├── Dockerfile
├── docker-compose.fcv.yml
├── docker-compose.dev.yml
└── .env.example
```
│   ├── agents/                              ← Custom agents (dropdown de Copilot)
│   │   ├── arc-vision.agent.md              ← Meta-agente arquitectura
│   │   ├── arc-meta-agent.agent.md          ← Meta-agente generador de ecosistema
```

