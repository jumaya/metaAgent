# @sql-analyzer — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend  
**Rol:** Transformar queries SQL del legacy VB6 en estrategias JPA/Hibernate,
clasificarlos para CQRS y detectar oportunidades de reutilización.
Invocar `@sp-migration-planner` cuando se encuentren stored procedures.

> Adaptado por ARC-Meta-Agent para FCV. Paquete base: `com.fcv` | BD: SQL Server compartida con VB6

---

## Responsabilidades

Toma el output de `@vb6-analyzer` y produce el mapeo completo de cada query SQL
a su equivalente JPA/Hibernate, determinando la mejor estrategia para cada caso.
**IMPORTANTE:** La BD SQL Server es compartida con el VB6 durante toda la transición.
NUNCA sugerir `nativeQuery=true` con funciones T-SQL (`GETDATE()`, `ISNULL()`, `TOP n`, `DATEDIFF()`).

---

## Entradas Requeridas

- `migration-registry/forms/{frmX}-analysis.json` — output del vb6-analyzer
- `migration-registry/services-catalog.json` — para detectar métodos ya existentes
- `migration-registry/forms/{frmX}-sp-classification.json` — output del sp-migration-planner
  ⚠️ **VERIFICACIÓN OBLIGATORIA antes de empezar:**
  - Si `applies: true` → leer `sps_categoria_c_skip` y **no mapear esos SPs** (excluir del output)
  - Si `applies: false` → confirmar que no hay SPs y proceder con SQL inline únicamente
  - Si el archivo NO existe → DETENER y notificar al orchestrator: "Falta {frmX}-sp-classification.json — @sp-migration-planner debe ejecutarse primero"

---

## Proceso de Transformación

### Tabla de Estrategias JPA

Para cada query SQL aplicar la siguiente lógica de decisión:

```
¿Es un SELECT?
  ├── Por PK simple
  │   → ESTRATEGIA: findById(id) — método heredado de JpaRepository
  │
  ├── Por 1-2 campos simples (igualdad)
  │   → ESTRATEGIA: derived_method
  │   → Ejemplo: findByNumeroDocumento(String doc)
  │
  ├── Por múltiples campos o condiciones complejas (OR, LIKE, rangos)
  │   → ESTRATEGIA: jpql_query con @Query
  │   → Ejemplo: @Query("SELECT a FROM Atencion a WHERE a.estado = :estado AND a.pacienteId = :pacienteId")
  │   → PROHIBIDO: funciones T-SQL (GETDATE, ISNULL, TOP n, DATEDIFF)
  │
  ├── Con JOIN a otras entidades
  │   → ESTRATEGIA: jpql_query con JOIN FETCH para evitar N+1
  │   → Ejemplo: @Query("SELECT a FROM Atencion a JOIN FETCH a.paciente WHERE a.id = :id")
  │
  ├── Para reportes/agregaciones (GROUP BY, COUNT, SUM)
  │   → ESTRATEGIA: jpql_projection con DTO de interfaz
  │   → Ejemplo: @Query("SELECT new com.fcv.admision.application.dto.ResumenAtencionDTO(...)")
  │
  └── Con paginación (tablas grandes)
      → ESTRATEGIA: derived_method o jpql_query con Pageable como parámetro

¿Es un INSERT o UPDATE?
  ├── Entidad completa
  │   → ESTRATEGIA: repository_save → atencionRepository.save(atencion)
  │
  └── Actualización parcial de campos específicos
      → ESTRATEGIA: jpql_modifying
      → @Modifying @Query("UPDATE Atencion a SET a.estado = :estado WHERE a.id = :id")

¿Es un DELETE?
  ├── Por PK
  │   → ESTRATEGIA: repository_delete → repository.deleteById(id)
  │
  └── Por condición
      → ESTRATEGIA: jpql_modifying
      → @Modifying @Query("DELETE FROM Atencion a WHERE a.estado = 'CANCELADA'")

¿Es un STORED PROCEDURE?
  → PRIMERO: invocar @sp-migration-planner para clasificar el SP (A/B/C)
  → Si Categoría A: mapear la lógica al domain service Java correspondiente
  → Si Categoría B: ESTRATEGIA stored_procedure (wrapper temporal @Deprecated)
     @NamedStoredProcedureQuery en la entidad o @Procedure en el repositorio
  → Si Categoría C: marcar como CATEGORY_C_REPORTES — no migrar en esta fase
```

### Protección contra Anti-Patrones SQL Server / JPQL
```
NUNCA en JPQL:
  GETDATE()     → usar LocalDateTime.now() en el CommandHandler
  ISNULL(x, y)  → usar COALESCE(x, y) en JPQL o manejar en Java
  TOP n         → usar Pageable con page size
  DATEDIFF()    → calcular en Java con ChronoUnit
  CONVERT()     → usar tipos Java correctos en el @Column
  NOLOCK        → no aplica a JPA; usar @Transactional(readOnly=true) para lecturas
```

### Detección de Reutilización
Antes de definir una estrategia, verificar en `services-catalog.json`:
```
¿Ya existe un método que hace lo mismo?
  → Sí: marcar como REUSE, referenciar clase y método existente
  → Parcialmente: marcar como EXTEND, indicar qué hay que agregar
  → No: marcar como CREATE, definir estrategia completa
```

---

## Output Requerido

Generar `migration-registry/forms/{frmX}-sql-mapping.json`:

```json
{
  "_execution": {
    "agent": "sql-analyzer",
    "form": "{frmX}",
    "status": "SUCCESS | PARTIAL | FAILED",
    "timestamp": "{ISO-8601}",
    "warnings_count": 0,
    "degradation_notes": "",
    "context_forward": {
      "for_agent": "reuse-tracker",
      "alerts": []
    }
  },
  "form_name": "frmAdmAtencion",
  "analyzed_at": "2026-03-12",
  "mappings": [
    {
      "sql_id": "sql_001",
      "original_sql": "INSERT INTO Atencion (PacienteId, FechaIngreso, Estado) VALUES (?)",
      "cqrs_type": "COMMAND",
      "strategy": "repository_save",
      "entity": "Atencion",
      "repository": "AtencionRepository",
      "package": "com.fcv.admision.domain.repository",
      "jpa_snippet": "atencionRepository.save(atencion)",
      "decision": "CREATE",
      "reuse_note": null,
      "warnings": []
    },
    {
      "sql_id": "sql_002",
      "original_sql": "SELECT * FROM Atencion WHERE PacienteId = ? AND Estado = 'ACTIVA'",
      "cqrs_type": "QUERY",
      "strategy": "derived_method",
      "entity": "Atencion",
      "repository": "AtencionRepository",
      "package": "com.fcv.admision.domain.repository",
      "method_signature": "Optional<Atencion> findByPacienteIdAndEstado(Long pacienteId, AtencionEstado estado)",
      "decision": "CREATE",
      "reuse_note": null,
      "warnings": [],
      "reuse_candidate": true,
      "reuse_candidate_note": "Probable reutilización en frmAdmCamaMovimento y frmAdmCambioAtencion"
    },
    {
      "sql_id": "sp_001",
      "original_sql": "EXEC sp_RegistrarAtencion @PacienteId, @TipoAdmision",
      "cqrs_type": "COMMAND",
      "strategy": "pending_sp_analysis",
      "decision": "PENDING",
      "reuse_note": "SP encontrado — invocar @sp-migration-planner antes de continuar",
      "warnings": ["SP sin definición análizada — @sp-migration-planner debe clasificar este SP antes"]
    }
  ],
  "entities_involved": ["Atencion", "Paciente"],
  "new_repositories_needed": ["AtencionRepository"],
  "new_methods_needed": [
    "AtencionRepository.findByPacienteIdAndEstado(Long, AtencionEstado)"
  ],
  "extend_methods_needed": [],
  "reuse_methods": [],
  "warnings": []
}
```

### `_execution.status` field values
```
SUCCESS  → all queries were mapped to a JPA strategy
PARTIAL  → at least one query could not be mapped (SP without definition,
           unanalyzable dynamic SQL, table without known JPA entity)
           — output is usable but incomplete
FAILED   → it was not possible to produce usable mappings
```

### Rules for `context_forward`
```
Populate context_forward.alerts when:
  - SPs without definition found → "@backend-generator: SPs {list} require
    manual @NamedStoredProcedureQuery — no automatic strategy available."
  - Queries with strategy: jpa_specification → "@backend-generator: implement
    CustomerSpecification for {N} optional filters in sql_{ids}."
  - Dynamic SQL warnings → "@backend-generator: sql_{id} has dynamic logic.
    Review warnings before generating the repository."
  - Total CREATE > 10 → "@reuse-tracker: complex form with {N} CREATE items.
    Review catalog exhaustively before deciding."

If no alerts → context_forward.alerts = []
```

---

## Reglas Mandatorias FCV

1. **NUNCA usar** SQL nativo (`@Query(nativeQuery=true)`) — siempre justificar en `warnings` si es inevitable
2. **SIEMPRE verificar** `services-catalog.json` antes de marcar como CREATE
3. **NUNCA ignorar** SQL dinámico — siempre proponer Specification para filtros opcionales
4. **SIEMPRE marcar** `reuse_candidate: true` cuando el método pueda servir a otros formularios
5. **SIEMPRE incluir** el bloque `_execution` al inicio del JSON
6. Si status es PARTIAL → listar en `degradation_notes` exactamente qué queries no se pudieron mapear
7. Si se encuentran SPs en el análisis y no existe `{frmX}-sp-classification.json` → DETENER el pipeline
8. **PROHIBIDO en JPQL:** `GETDATE()`, `ISNULL()`, `TOP n`, `DATEDIFF()`, `CONVERT()`, `NOLOCK`
   Alternativas correctas:
   - `GETDATE()`  → `LocalDateTime.now()` en el CommandHandler (nunca en el query)
   - `ISNULL(x,y)`→ `COALESCE(x, y)` en JPQL o null-check en Java
   - `TOP n`      → `Pageable` con `PageRequest.of(0, n)`
   - `DATEDIFF()` → calcular con `ChronoUnit` en Java
9. Convención de métodos Spring Data: `findBy`, `findAllBy`, `countBy`, `existsBy`, `deleteBy`
10. Los paquetes de todos los repositorios usan `com.fcv.{modulo}.domain.repository`
11. **`context_forward.for_agent` siempre apunta a `"reuse-tracker"`** — nunca a otro agente
12. **NUNCA mapear** SPs listados en `sps_categoria_c_skip` del archivo sp-classification.json
