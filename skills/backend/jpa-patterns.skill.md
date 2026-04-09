# Skill: JPA/Hibernate Patterns — Advanced Persistence
**Used by:** `@backend-generator` | **Layer:** Backend / Infrastructure

---

## Base Entity Configuration (@MappedSuperclass)

```java
// Base class for auditing — all entities extend this
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "FechaCreacion", nullable = false, updatable = false)
    private LocalDateTime creadoEn;

    @LastModifiedDate
    @Column(name = "FechaModificacion")
    private LocalDateTime modificadoEn;

    @CreatedBy
    @Column(name = "CreadoPor", updatable = false, length = 100)
    private String creadoPor;

    @LastModifiedBy
    @Column(name = "ModificadoPor", length = 100)
    private String modificadoPor;

    @Version                                          // ← Optimistic locking
    @Column(name = "Version")
    private Long version;
}

// Enable auditing in Application or configuration class
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

---

## JPA Relationships — Correct Patterns

### @ManyToOne (most common — always LAZY)

```java
// ✅ CORRECT — LAZY avoids N+1 queries
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "MedicoId", nullable = false)
private Medico medicoTratante;

// ❌ INCORRECT — EAGER always loads the full graph
@ManyToOne(fetch = FetchType.EAGER)  // never use EAGER on @ManyToOne
private Medico medicoTratante;
```

### @OneToMany — Bidirectional with orphanRemoval

```java
@OneToMany(mappedBy = "admision",
           cascade = CascadeType.ALL,
           orphanRemoval = true,           // ← automatically remove orphan children
           fetch = FetchType.LAZY)         // ← LAZY always on collections
private List<DiagnosticoAdmision> diagnosticos = new ArrayList<>();

// MANAGEMENT METHODS — maintain bidirectional consistency
public void agregarDiagnostico(DiagnosticoAdmision diagnostico) {
    diagnosticos.add(diagnostico);
    diagnostico.setAdmision(this);         // ← sync the inverse side
}

public void removerDiagnostico(DiagnosticoAdmision diagnostico) {
    diagnosticos.remove(diagnostico);
    diagnostico.setAdmision(null);
}
```

### @ManyToMany — Use with caution

```java
// Prefer explicit intermediate entity over native @ManyToMany
// Allows adding attributes to the relationship (date, status, etc.)

@Entity
@Table(name = "AdmisionProcedimiento")
public class AdmisionProcedimiento {

    @EmbeddedId
    private AdmisionProcedimientoId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("admisionId")
    private Admision admision;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("procedimientoId")
    private Procedimiento procedimiento;

    @Column(name = "FechaRealizacion")
    private LocalDate fechaRealizacion;    // attribute belonging to the relationship
}

@Embeddable
public record AdmisionProcedimientoId(Long admisionId, Long procedimientoId)
    implements Serializable {}
```

---

## N+1 Problem — Detection and Solution

```java
// ❌ N+1 PROBLEM — one query for the list + N queries for each relationship
List<Admision> admisiones = repo.findAll();
admisiones.forEach(a -> a.getMedicoTratante().getNombre()); // N extra queries

// ✅ SOLUTION 1 — JOIN FETCH in JPQL (for small collections)
@Query("SELECT a FROM Admision a JOIN FETCH a.medicoTratante WHERE a.estado = :estado")
List<Admision> findByEstadoConMedico(@Param("estado") EstadoAdmision estado);

// ✅ SOLUTION 2 — @EntityGraph (for configurable cases)
@EntityGraph(attributePaths = {"medicoTratante", "paciente"})
List<Admision> findByFechaIngreso(LocalDate fecha);

// ✅ SOLUTION 3 — DTO Projection (most efficient for read queries)
// Loads ONLY the needed fields, without instantiating the full entity
@Query("""
    SELECT new com.empresa.admision.application.dto.AdmisionSummaryDTO(
        a.id, a.numeroAdmision, p.nombreCompleto, a.estado, a.fechaIngreso
    )
    FROM Admision a
    JOIN a.paciente p
    WHERE a.fechaIngreso BETWEEN :desde AND :hasta
    ORDER BY a.fechaIngreso DESC
    """)
List<AdmisionSummaryDTO> findSummaryByFecha(
    @Param("desde") LocalDate desde,
    @Param("hasta") LocalDate hasta
);
```

---

## Correct Pagination

```java
// In the Repository
public interface AdmisionRepository extends JpaRepository<Admision, Long>,
                                             JpaSpecificationExecutor<Admision> {

    // Pagination with JOIN FETCH — requires separate countQuery to avoid error
    @Query(
        value = "SELECT a FROM Admision a JOIN FETCH a.paciente WHERE a.estado = :estado",
        countQuery = "SELECT COUNT(a) FROM Admision a WHERE a.estado = :estado"
    )
    Page<Admision> findByEstadoConPaciente(@Param("estado") EstadoAdmision estado, Pageable pageable);
}

// In the Controller — Pageable comes from query params automatically
// GET /api/admisiones?page=0&size=20&sort=fechaIngreso,desc
@GetMapping
public ResponseEntity<Page<AdmisionSummaryDTO>> listar(
        @RequestParam(required = false) EstadoAdmision estado,
        @PageableDefault(size = 20, sort = "fechaIngreso", direction = DESC) Pageable pageable) {
    return ResponseEntity.ok(listadoHandler.handle(new ListarAdmisionesQuery(estado, pageable)));
}
```

---

## Enum Handling in the Database

```java
// ✅ CORRECT — store as readable String, not a number
@Enumerated(EnumType.STRING)
@Column(name = "Estado", nullable = false, length = 30)
private EstadoAdmision estado;

// ❌ INCORRECT — EnumType.ORDINAL breaks if the enum order changes
@Enumerated(EnumType.ORDINAL)
private EstadoAdmision estado;

// For legacy enums with short codes in the DB (e.g. "A" = ACTIVE)
@Convert(converter = EstadoAdmisionConverter.class)
@Column(name = "Estado", length = 1)
private EstadoAdmision estado;

@Converter(autoApply = false)
public class EstadoAdmisionConverter implements AttributeConverter<EstadoAdmision, String> {
    @Override public String convertToDatabaseColumn(EstadoAdmision attr) {
        return attr == null ? null : attr.getCodigo();   // "A", "I", "C"
    }
    @Override public EstadoAdmision convertToEntityAttribute(String dbData) {
        return dbData == null ? null : EstadoAdmision.fromCodigo(dbData);
    }
}
```

---

## Native Queries — Justified Cases Only

```java
// Only when JPQL cannot express the query (engine-specific functions, CTEs, etc.)
// ALWAYS include a justification comment

@Query(
    // justification: uses T-SQL recursive CTE for bed hierarchy, not expressible in JPQL
    value = """
        WITH CamasJerarquia AS (
            SELECT CamaId, PisoCodigo, HabitacionNro, 0 as Nivel
            FROM Camas WHERE PisoCodigo = :piso
            UNION ALL
            -- ...
        )
        SELECT * FROM CamasJerarquia
        """,
    nativeQuery = true
)
List<CamaProjection> findJerarquiaCamas(@Param("piso") String piso);

// Projection for native queries — mapping without instantiating the entity
public interface CamaProjection {
    Long getCamaId();
    String getPisoCodigo();
    Integer getHabitacionNro();
}
```

---

## JPA Checklist — Before Generating

```
□ Do all @ManyToOne and @OneToMany relationships use FetchType.LAZY?
□ Do entities with auditing extend BaseEntity?
□ Do @OneToMany have orphanRemoval = true where appropriate?
□ Do enums use @Enumerated(EnumType.STRING)?
□ Do queries with pagination and JOIN FETCH have a separate countQuery?
□ Are identified N+1 queries resolved with @EntityGraph or JOIN FETCH?
□ Do native queries have a justification comment?
□ Are @ManyToMany relationships modeled as explicit intermediate entities?
□ Is @Version used for optimistic locking on concurrent entities?
