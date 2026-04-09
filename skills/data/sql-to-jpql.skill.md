# Skill: SQL Server → JPQL/JPA Transformation
**Used by:** `@sql-analyzer`, `@backend-generator` | **Layer:** Data

---

## Golden Rule

> **NEVER use T-SQL functions in JPQL.** JPQL is database-engine agnostic.
> Always use standard JPA equivalents or Java logic.

---

## T-SQL → JPQL Equivalence Table

| T-SQL (legacy) | JPQL / JPA Equivalent | Notes |
|---|---|---|
| `GETDATE()` | `CURRENT_TIMESTAMP` or Java `LocalDateTime.now()` | Prefer Java for testability |
| `ISNULL(x, y)` | `COALESCE(x, y)` | JPQL supports COALESCE |
| `TOP n` | `Pageable` with `PageRequest.of(0, n)` | Never TOP in JPQL |
| `NOLOCK` | `@Transactional(readOnly=true)` | Isolation level, not a hint |
| `CONVERT(varchar, fecha)` | Java `DateTimeFormatter` in DTO | Format in the app layer |
| `DATEDIFF(day, f1, f2)` | `ChronoUnit.DAYS.between(f1, f2)` | Calculate in Java |
| `DATEADD(day, n, fecha)` | `fecha.plusDays(n)` | LocalDate/LocalDateTime |
| `LEN(campo)` | `LENGTH(campo)` | JPQL uses LENGTH |
| `SUBSTRING(x,i,n)` | `SUBSTRING(x, i, n)` | Same in JPQL |
| `UPPER(x)` / `LOWER(x)` | `UPPER(x)` / `LOWER(x)` | Same in JPQL |
| `LIKE '%texto%'` | `LIKE CONCAT('%',:param,'%')` | Concatenate in JPQL |
| `IS NULL` / `IS NOT NULL` | `IS NULL` / `IS NOT NULL` | Same in JPQL |
| `IN (lista)` | `IN :lista` with `List<>` | Pass Java collection |
| `EXISTS (subquery)` | `EXISTS (subquery)` | Same in JPQL |
| `ROWNUM` / `ROW_NUMBER()` | `Pageable` | Use JPA pagination |

---

## Transformation Patterns

### Simple SELECT with filters → Query Method / JPQL

```sql
-- Legacy T-SQL
SELECT ClienteId, Nombre, NroDoc, Estado
FROM Clientes
WHERE Estado = 'A' AND TipoDocId = @TipoDocId
ORDER BY Nombre
```

```java
// ✅ Option 1 — Query Method (for simple filters)
List<Cliente> findByEstadoAndTipoDocumentoIdOrderByNombreAsc(
    EstadoCliente estado, Long tipoDocumentoId
);

// ✅ Option 2 — Named JPQL (for more control)
@Query("SELECT c FROM Cliente c WHERE c.estado = :estado AND c.tipoDocumento.id = :tipoDocId ORDER BY c.nombre")
List<ClienteSummaryDTO> findActivosPorTipoDoc(
    @Param("estado") EstadoCliente estado,
    @Param("tipoDocId") Long tipoDocId
);
```

### SELECT with JOIN → JPQL with JOIN FETCH or DTO Projection

```sql
-- Legacy T-SQL
SELECT c.ClienteId, c.Nombre, td.Descripcion as TipoDoc, ci.NombreCiudad
FROM Clientes c
INNER JOIN TiposDocumento td ON c.TipoDocId = td.TipoDocId
LEFT JOIN Ciudades ci ON c.CiudadId = ci.CiudadId
WHERE c.Estado = 'A'
```

```java
// ✅ DTO Projection — efficient, only the needed fields
@Query("""
    SELECT new com.empresa.cliente.dto.ClienteResponseDTO(
        c.id, c.nombre, td.descripcion, ci.nombre
    )
    FROM Cliente c
    JOIN c.tipoDocumento td
    LEFT JOIN c.ciudad ci
    WHERE c.estado = :estado
    """)
List<ClienteResponseDTO> findActivosConDetalles(@Param("estado") EstadoCliente estado);
```

### SELECT with ISNULL / COALESCE

```sql
-- Legacy T-SQL
SELECT ClienteId, ISNULL(Telefono, 'No phone') as Telefono
FROM Clientes
```

```java
// ✅ Option 1 — COALESCE in JPQL
@Query("SELECT c.id, COALESCE(c.telefono, 'No phone') FROM Cliente c")

// ✅ Option 2 (recommended) — handle in DTO, not in query
public record ClienteDTO(Long id, String telefono) {
    public String getTelefonoDisplay() {
        return telefono != null ? telefono : "No phone";
    }
}
```

### SELECT with dates and GETDATE()

```sql
-- Legacy T-SQL
SELECT * FROM Admisiones
WHERE FechaIngreso >= DATEADD(day, -30, GETDATE())
  AND FechaIngreso <= GETDATE()
```

```java
// ✅ Calculate dates in Java, pass as parameters
@Transactional(readOnly = true)
public List<AdmisionSummaryDTO> findLast30Days() {
    LocalDate until = LocalDate.now();
    LocalDate from = until.minusDays(30);
    return admisionRepository.findByFechaIngresoBetween(from, until);
}

// Repository — no GETDATE() or DATEADD in the query
List<Admision> findByFechaIngresoBetween(LocalDate desde, LocalDate hasta);
```

### SELECT with pagination TOP/ROWNUM

```sql
-- Legacy T-SQL (Top 10)
SELECT TOP 10 * FROM Pacientes ORDER BY FechaCreacion DESC

-- Legacy Oracle
SELECT * FROM Pacientes WHERE ROWNUM <= 10 ORDER BY FechaCreacion DESC
```

```java
// ✅ Use Pageable
Page<Paciente> findAll(Pageable pageable);

// Call: PageRequest.of(0, 10, Sort.by("creadoEn").descending())
```

### INSERT / UPDATE → CommandHandler + save()

```sql
-- Legacy T-SQL
INSERT INTO Pacientes (NroDoc, Nombre, Estado, FechaCreacion)
VALUES (@NroDoc, @Nombre, 'A', GETDATE())

UPDATE Pacientes SET Nombre = @Nombre, FechaModificacion = GETDATE()
WHERE PacienteId = @Id
```

```java
// ✅ INSERT → save() with a new entity
Paciente paciente = Paciente.crear(command.documentoNumero(), command.nombre());
pacienteRepository.save(paciente);
// FechaCreacion → @CreatedDate in BaseEntity
// FechaModificacion → @LastModifiedDate in BaseEntity

// ✅ UPDATE → load entity + modify + save()
Paciente paciente = pacienteRepository.findById(command.id())
    .orElseThrow(() -> new EntityNotFoundException("Paciente", command.id()));
paciente.actualizarNombre(command.nombre());   // domain method
pacienteRepository.save(paciente);
// FechaModificacion updated automatically by @LastModifiedDate
```

### Stored Procedures → A/B/C Classification

```
Category A — Migrate to Java service:
  Criteria: clear business logic, no complex cursors
  Action: extract logic to CommandHandler or domain method

  Legacy SP example:
  CREATE PROC sp_ValidarDisponibilidadCama @HabitacionId INT AS BEGIN
    IF (SELECT COUNT(*) FROM Camas WHERE HabitacionId=@HabitacionId AND Estado='D') > 0
      RETURN 1
    RETURN 0
  END

  ✅ In Java:
  boolean existsByHabitacionIdAndEstado(Long habitacionId, EstadoCama estado);
  // CommandHandler: if (!camaRepo.existsByHabitacionIdAndEstado(id, DISPONIBLE)) throw ...

Category B — Replace with JPA/JPQL:
  Criteria: SP is simple CRUD or a direct query with no logic
  Action: Query Method or @Query in the repository

Category C — Wrap with @Procedure:
  Criteria: complex SP, critical, with cursors or logic hard to migrate
  Action: call the original SP from Java until it can be migrated

  @Procedure(procedureName = "sp_GenerarNumeroAdmision")
  String generarNumeroAdmision(@Param("sede") String sede);
```

---

## SQL → JPA Checklist — Before Generating

```
□ Were all T-SQL functions removed from JPQL (GETDATE, ISNULL, TOP)?
□ Are date calculations done in Java (LocalDate.now(), plusDays(), etc.)?
□ Does pagination use Pageable instead of TOP/ROWNUM?
□ Do INSERT/UPDATE go through save() with @CreatedDate/@LastModifiedDate?
□ Are complex SPs classified as A/B/C?
□ Do Category C SPs use @Procedure in the repository?
□ Do LIKE clauses use CONCAT('%',:param,'%') in JPQL?
□ Do optional filters use Specification instead of conditional queries?
