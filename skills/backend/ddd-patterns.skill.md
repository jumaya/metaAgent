# Skill: DDD Patterns — Domain-Driven Design
**Used by:** `@backend-generator` | **Layer:** Backend / Domain

---

## Aggregate Root

The aggregate is the unit of consistency. Only the Aggregate Root is accessible
from outside. Internal entities are only modified through the root.

```java
// Aggregate Root — controls the lifecycle of its child entities
@Entity
@Table(name = "Admisiones")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Admision {                          // ← Aggregate Root

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "AdmisionId")
    private Long id;

    // Child entity — NEVER expose directly outside the aggregate
    @OneToMany(mappedBy = "admision", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<DiagnosticoAdmision> diagnosticos = new ArrayList<>();

    // ✅ Controlled access to child collection — never return the mutable list
    public List<DiagnosticoAdmision> getDiagnosticos() {
        return Collections.unmodifiableList(diagnosticos);
    }

    // ✅ Domain method that enforces aggregate invariant
    public void agregarDiagnostico(String cie10, String descripcion) {
        if (this.estado != EstadoAdmision.ACTIVA) {
            throw new DomainException("Cannot add diagnosis to a closed admission");
        }
        diagnosticos.add(DiagnosticoAdmision.crear(this, cie10, descripcion));
    }
}
```

**Aggregate Rules:**
- One repository per Aggregate Root — never per child entity
- Business invariants are validated INSIDE the aggregate
- Load the complete aggregate before modifying it (`@Transactional`)
- Size limit: if the aggregate grows too large → check if there is a separate bounded context

---

## Value Object (@Embeddable)

Object with no identity of its own, defined by its attributes. Immutable.

```java
// Value Object — no @Id, immutable, compared by value
@Embeddable
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Documento {

    @Column(name = "TipoDoc", nullable = false, length = 5)
    private String tipo;          // "CC", "TI", "CE", "PA"

    @Column(name = "NroDoc", nullable = false, length = 20)
    private String numero;

    // Factory method with invariant validation
    public static Documento of(String tipo, String numero) {
        if (tipo == null || tipo.isBlank())
            throw new DomainException("Document type is required");
        if (numero == null || numero.isBlank())
            throw new DomainException("Document number is required");
        if (!Set.of("CC","TI","CE","PA","RC","NIT").contains(tipo.toUpperCase()))
            throw new DomainException("Invalid document type: " + tipo);

        Documento d = new Documento();
        d.tipo = tipo.toUpperCase();
        d.numero = numero.trim();
        return d;
    }

    // Equality by value (not by reference)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Documento)) return false;
        Documento other = (Documento) o;
        return Objects.equals(tipo, other.tipo) && Objects.equals(numero, other.numero);
    }

    @Override
    public int hashCode() {
        return Objects.hash(tipo, numero);
    }

    @Override
    public String toString() {
        return tipo + "-" + numero;
    }
}

// Usage in entity
@Entity
public class Paciente {
    @Embedded
    private Documento documento;    // ← columns TipoDoc and NroDoc in the table
}
```

---

## Domain Event

Events communicate that something happened in the domain. They are immutable and named in past tense.

```java
// Domain event — immutable record, past-tense name
// PATTERN: {Entity}{PastAction}Event
public record AdmisionCreadaEvent(
    Long admisionId,
    Long pacienteId,
    String numeroPaciente,
    LocalDateTime occurredAt
) {
    // Compact constructor with automatic timestamp
    public AdmisionCreadaEvent(Long admisionId, Long pacienteId, String numeroPaciente) {
        this(admisionId, pacienteId, numeroPaciente, LocalDateTime.now());
    }
}

// Publishing from the CommandHandler (not from the entity)
@Transactional
public Long handle(CrearAdmisionCommand command) {
    Admision admision = Admision.crear(/* ... */);
    admisionRepository.save(admision);

    // Publish AFTER saving, WITHIN the same transaction
    eventPublisher.publishEvent(
        new AdmisionCreadaEvent(admision.getId(), admision.getPacienteId(), command.numeroPaciente())
    );
    return admision.getId();
}

// Event listener (in another bounded context or service)
@Component
public class AdmisionCreadaEventListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onAdmisionCreada(AdmisionCreadaEvent event) {
        // Executes ONLY if the main transaction was successful
        // Ideal for: notifications, updating catalogs, sending to external queues
    }
}
```

---

## Domain vs Infrastructure Repository

```java
// DOMAIN LAYER — pure interface, no Spring or JPA imports
// Defines WHAT the domain needs, not HOW it is implemented
public interface AdmisionRepository {
    Admision save(Admision admision);
    Optional<Admision> findById(Long id);
    Optional<Admision> findByNumeroAdmision(String numero);
    boolean existsByNumeroAdmisionAndFecha(String numero, LocalDate fecha);
}

// INFRASTRUCTURE LAYER — JPA implementation
// Can use Spring Data directly
@Repository
public interface JpaAdmisionRepository
    extends JpaRepository<Admision, Long>, AdmisionRepository {

    // Spring Data generates the implementation automatically
    Optional<Admision> findByNumeroAdmision(String numero);
    boolean existsByNumeroAdmisionAndFecha(String numero, LocalDate fecha);
}
```

---

## Factory Method on the Entity

Use static factory methods instead of public constructors.

```java
// ❌ BAD — public constructor exposes an incomplete object
public Admision(Long pacienteId, String numero) {
    this.pacienteId = pacienteId;
    this.numero = numero;
    // status forgotten → object in invalid state
}

// ✅ GOOD — factory method guarantees invariants from the start
public static Admision crear(Long pacienteId, String numeroAdmision, TipoAdmision tipo) {
    Objects.requireNonNull(pacienteId, "pacienteId is required");
    Objects.requireNonNull(numeroAdmision, "numeroAdmision is required");

    Admision a = new Admision();
    a.pacienteId = pacienteId;
    a.numeroAdmision = numeroAdmision;
    a.tipo = tipo;
    a.estado = EstadoAdmision.ACTIVA;               // ← invariant guaranteed
    a.fechaIngreso = LocalDateTime.now();           // ← automatic timestamp
    return a;
}

// Factory method for reconstitution (from persistence or events)
public static Admision reconstituir(Long id, Long pacienteId, String numero,
                                     EstadoAdmision estado, LocalDateTime fechaIngreso) {
    Admision a = new Admision();
    a.id = id;
    a.pacienteId = pacienteId;
    // ... assign without creation validations
    return a;
}
```

---

## DDD Checklist — Before Generating

```
□ Does each entity have a static factory method instead of a public constructor?
□ Are Value Objects immutable and have equality by value?
□ Are business invariants in the entity, NOT in the handler?
□ Are domain events named in past tense?
□ Are events published AFTER persisting (within the transaction)?
□ Is the domain repository a pure interface with no infrastructure imports?
□ Are child collections exposed as Collections.unmodifiableList()?
□ Is the default constructor protected (AccessLevel.PROTECTED)?
