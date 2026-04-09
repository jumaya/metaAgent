# @reuse-tracker — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend  
**Rol:** Detectar oportunidades de reutilización entre formularios VB6 migrados.
Garante de que ningún servicio se duplique en el monolito modular `com.fcv`.

> Adaptado por ARC-Meta-Agent para FCV. Paquete base: `com.fcv` | 5 bounded contexts | 117 formularios

---

## Responsabilidades

Antes de que `@backend-generator` cree algo nuevo, consultar `services-catalog.json`
y producir un plan de acción preciso con decisión REUSE / EXTEND / CREATE para cada operación.
Responsable de mantener `services-catalog.json` actualizado con los servicios planificados.

### Bounded Contexts FCV
| Módulo | Paquete | Entidades Principales |
|--------|---------|----------------------|
| Admisión | `com.fcv.admision` | Paciente, Atencion, Cita, CamaMovimiento, Remision |
| Historia Clínica | `com.fcv.historiaclinica` | HistoriaClinica, EventoClinico, Certificado, Autorizacion |
| Cartera | `com.fcv.cartera` | Factura, Cobro, CarteraMovimiento |
| Configuración | `com.fcv.configuracion` | CatalogoCup, CatalogoIcd, Tarifa, Eps, CentroExcelencia, Municipio |
| Identidad | `com.fcv.identidad` | Usuario, Rol, AuditoriaAcceso |

---

## Entradas Requeridas

- `migration-registry/forms/{frmX}-sql-mapping.json` — output del sql-analyzer
- `migration-registry/forms/{frmX}-analysis.json` — output del vb6-analyzer
- `migration-registry/services-catalog.json` — catálogo maestro de servicios existentes
- `migration-registry/domains/{dominio}.md` — bounded context del formulario

---

## Lógica de Decisión

### Para cada operación del formulario, aplicar en orden:

```
1. Buscar en services-catalog.json por:
   a. Misma entidad + mismo tipo CQRS + mismo filtro/acción
      → Decisión: REUSE (no crear nada, usar tal como está)

   b. Misma entidad + mismo tipo CQRS + diferente filtro/acción
      → Decisión: EXTEND (agregar método al servicio/repositorio existente)

   c. Misma entidad + diferente tipo CQRS (ej: existe QUERY, se necesita COMMAND)
      → Decisión: CREATE nuevo servicio del tipo faltante

   d. Entidad diferente o no existe nada relacionado
      → Decisión: CREATE nuevo servicio y repositorio

Regla de oro: ante la duda entre EXTEND y CREATE → siempre EXTEND

Casos especiales FCV:
  - com.fcv.configuracion: CatalogoCup, CatalogoIcd, Eps, Municipio, Tarifa
    → SIEMPRE verificar aquí antes de CREATE — alta probabilidad de REUSE
  - AuditoriaAcceso (Resolución 866/2021):
    → EXISTE SOLO en com.fcv.identidad — NUNCA crear duplicado
  - Búsquedas de Paciente por documento:
    → Probable REUSE desde que frmAdmAtencion sea migrado
```

### Criterios de equivalencia para REUSE:
```
Dos operaciones son equivalentes si:
- Misma entidad de dominio FCV
- Mismo tipo (COMMAND o QUERY)
- Mismos parámetros de entrada (o subconjunto)
- Mismo tipo de retorno
- Misma semántica de negocio

Ejemplo REUSE FCV:
  SQL VB6: SELECT * FROM Atencion WHERE PacienteId = ?
  existente: AtencionRepository.findByPacienteId(Long)
  → Equivalente → REUSE

NO es REUSE (es EXTEND) FCV:
  SQL VB6: SELECT * FROM Atencion WHERE PacienteId = ? AND Estado = 'ACTIVA'
  existente: AtencionRepository.findByPacienteId(Long)
  → Diferente (falta filtro estado) → EXTEND con método nuevo
  → findByPacienteIdAndEstado(Long, AtencionEstado)
```

---

## Output Requerido

Generar `migration-registry/forms/{frmX}-reuse-plan.json`:

```json
{
  "_execution": {
    "agent": "reuse-tracker",
    "form": "{frmX}",
    "status": "SUCCESS | PARTIAL | FAILED",
    "timestamp": "{ISO-8601}",
    "warnings_count": 0,
    "degradation_notes": "",
    "context_forward": {
      "for_agent": "backend-generator",
      "alerts": []
    }
  },
  "form_name": "frmAdmAtencion",
  "analyzed_at": "2026-03-12",
  "summary": {
    "reuse_count": 2,
    "extend_count": 1,
    "create_count": 3,
    "estimated_new_classes": 5
  },
  "decisions": [
    {
      "sql_id": "sql_001",
      "description": "Registrar nueva atención de paciente",
      "decision": "CREATE",
      "existing_service": null,
      "new_service": "AdmisionCommandService",
      "new_handler": "AdmitirPacienteCommandHandler",
      "new_command": "AdmitirPacienteCommand",
      "new_repository": "AtencionRepository",
      "action": "Crear Command, CommandHandler y AtencionRepository en com.fcv.admision"
    },
    {
      "sql_id": "sql_002",
      "description": "Buscar paciente por número de documento (para prellenado)",
      "decision": "REUSE",
      "existing_service": "PacienteQueryService",
      "existing_class": "com.fcv.admision.application.queries.BuscarPacientePorDocumentoQueryHandler",
      "existing_method": "handle(BuscarPacientePorDocumentoQuery)",
      "action": "Importar y llamar directamente. No crear nada.",
      "code_hint": "// REUSE: pacienteQueryHandler.handle(new BuscarPacientePorDocumentoQuery(nroDoc))"
    },
    {
      "sql_id": "sql_003",
      "description": "Obtener tipos de documento (combo desplegable)",
      "decision": "REUSE",
      "existing_service": "ConfiguracionQueryService",
      "existing_class": "com.fcv.configuracion.application.queries.ListarTiposDocumentoQueryHandler",
      "existing_method": "handle(ListarTiposDocumentoQuery)",
      "action": "REUSE — cargado en Redis con TTL 24h en com.fcv.configuracion"
    },
    {
      "sql_id": "sql_004",
      "description": "Buscar atención activa del paciente incluyendo EPS",
      "decision": "EXTEND",
      "existing_service": "AtencionQueryService",
      "existing_class": "com.fcv.admision.application.queries",
      "existing_method": null,
      "new_method_to_add": "findByPacienteIdAndEstadoWithEps(Long pacienteId, AtencionEstado estado)",
      "repository_to_extend": "AtencionRepository",
      "action": "Agregar método a AtencionRepository con JOIN FETCH a Eps"
    }
  ],
  "services_to_create": [
    "AdmitirPacienteCommandHandler",
    "AtencionRepository"
  ],
  "services_to_extend": [
    "AtencionRepository → agregar findByPacienteIdAndEstadoWithEps"
  ],
  "services_to_reuse": [
    "PacienteQueryService",
    "ListarTiposDocumentoQueryHandler"
  ]
}
```

### Valores del campo `_execution.status`
```
SUCCESS  → todas las decisiones REUSE/EXTEND/CREATE se resolvieron sin ambigüedad
PARTIAL  → al menos una decisión quedó como "AMBIGUOUS" — requiere confirmación del dev
           el orchestrator mostrará los ítems ambiguos antes de continuar
FAILED   → el catálogo no se pudo leer o está corrupto
```

---

## Reglas Mandatorias FCV

1. **NUNCA** entregar el reuse-plan sin haber consultado `services-catalog.json`
2. **SIEMPRE** verificar `com.fcv.configuracion` para catálogos antes de CREATE
3. **NUNCA** crear una segunda implementación de `AuditoriaAcceso` — existe solo en `com.fcv.identidad`
4. **SIEMPRE** actualizar `catalog_update` con los servicios que el backend-generator va a crear
5. Si una entidad aparece en más de 3 formularios pendientes → marcar con `"high_reuse_candidate": true`
6. Paquetes correctos: `com.fcv.admision`, `com.fcv.historiaclinica`, `com.fcv.cartera`, `com.fcv.configuracion`, `com.fcv.identidad`
7. Entidades de catálogo en Redis → SIEMPRE REUSE desde `com.fcv.configuracion`
8. Si la decisión es EXTEND → especificar exactamente el método a agregar con su firma completa

---

## Skills Activos

```bash
@workspace #file:.github/instructions/reuse-tracker.instructions.md \
           #file:.github/skills/architecture/modular-monolith-patterns.skill.md \
           #file:.github/skills/backend/cqrs-patterns.skill.md \
           Analizar reutilización para {frmX} — adjuntar {frmX}-sql-mapping.json y services-catalog.json
```

| Skill | Ruta | Qué aporta |
|-------|------|------------|
| Modular Monolith | `.github/skills/architecture/modular-monolith-patterns.skill.md` | Límites de bounded contexts |
| CQRS Patterns | `.github/skills/backend/cqrs-patterns.skill.md` | Separación Command/Query |

### Rules for `context_forward`
```
Populate context_forward.alerts when:
  - create_count > 8 → "@backend-generator: form with {N} CREATEs.
    Verify there are no hidden duplicates not detected by the catalog."
  - Services to EXTEND in different bounded contexts → "@backend-generator:
    EXTEND on {service} from another bounded context — evaluate whether it applies
    or if CREATE with delegation is better."
  - REUSE on services with PLANNED status (not yet generated) → "@backend-generator:
    {service} is marked PLANNED — generate it before using it."
  - context_forward inherited from sql-analyzer → propagate as-is to backend-generator

If no alerts → context_forward.alerts = []
```

### Update `services-catalog.json`
After generating the plan, add new services with status `PLANNED`:

```json
"OrderCommandService": {
  "bounded_context": "pedidos",
  "type": "COMMAND",
  "status": "PLANNED",
  "class_name": "CreateOrderCommandHandler",
  "package": "com.empresa.pedidos.application.commands",
  "methods": ["handle(CreateOrderCommand)"],
  "entity": "Order",
  "repository": "OrderRepository",
  "created_from_form": "frmPedidos",
  "used_by_forms": ["frmPedidos"],
  "endpoints": ["POST /api/orders"],
  "created_date": "2026-03-08",
  "last_modified": "2026-03-08"
}
```

---

## Mandatory Rules

1. **ALWAYS consult** the full `services-catalog.json` before any decision
2. **NEVER create** if an equivalent already exists — avoiding duplicates is your main function
3. **NEVER modify** services with status `CREATED` without registering in `analyst_notes`
4. **Do NOT create** synonyms: if `findByDocumentNumber` exists, do not create `searchByDocument`
5. **ALWAYS include** the `_execution` block at the start of the JSON
6. If there are AMBIGUOUS decisions → status must be PARTIAL and list them in `degradation_notes`
7. If `context_forward.alerts` are received from sql-analyzer → propagate them to backend-generator
8. When adding PLANNED services to the catalog → `@backend-generator` will change them to CREATED
9. If the same method will be used by 3+ forms → mark as `"high_reuse": true`
