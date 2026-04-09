# Skill: Modular Monolith — Structure, Boundaries and Rules
**Used by:** `@arc-vision`, `@arc-meta-agent`, `@backend-generator` | **Layer:** Architecture / Backend

---

## What is a Modular Monolith?

A single deployable unit internally organized into **modules with strict boundaries**,
each module encapsulating a bounded context. It behaves like a monolith at the infrastructure
level (one process, one database) and like microservices at the code level
(no shared code between modules except through explicit contracts).

```
┌─────────────────────────────────────────────────────┐
│                  Spring Boot App                    │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Admissions │  │   Billing    │  │  Patients │ │
│  │   Module     │  │   Module     │  │  Module   │ │
│  │              │  │              │  │           │ │
│  │  api/        │  │  api/        │  │  api/     │ │
│  │  application/│  │  application/│  │  applic.. │ │
│  │  domain/     │  │  domain/     │  │  domain/  │ │
│  │  infra/      │  │  infra/      │  │  infra/   │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                 │                │       │
│         └─────────────────┴────────────────┘       │
│                    Shared Kernel                    │
│              (only cross-cutting concerns)          │
└─────────────────────────────────────────────────────┘
         ↓ single deployment, single database
```

---

## When to Choose It (ARC-Vision Decision Guide)

```
✅ Recommended when:
  - Team of 3–25 developers
  - Project in its first 1–3 years (domains not fully known yet)
  - Need for deployment simplicity (single JAR, no orchestration)
  - Bounded contexts already identified but scale does not justify microservices
  - Migration from a legacy monolith (VB6, classic ASP, Delphi, etc.)
  - Deadline pressure: avoids operational overhead of distributed systems

⚠️ Revisit when:
  - A single module requires 3x more frequent deployments than the others
  - A module needs an independently scalable technology (e.g., ML module in Python)
  - Multiple teams > 10 people each working independently on different modules
  - Module compilation time begins to affect developer productivity

❌ Not recommended when:
  - The system is already confirmed to need elastic/independent scaling per domain
  - There are > 5 external teams that will consume the APIs independently
  - There are strict compliance requirements for data isolation per tenant
```

---

## Package Structure (MANDATORY)

Each module follows the same internal structure. Never mix packages from different modules.

```
src/main/java/{base_package}/
├── admissions/                        ← Module = Bounded Context
│   ├── api/                           ← Controllers (REST entry points)
│   │   ├── AdmissionCommandController.java
│   │   └── AdmissionQueryController.java
│   ├── application/                   ← Use cases (Commands, Queries, Handlers)
│   │   ├── commands/
│   │   │   ├── CreateAdmissionCommand.java
│   │   │   └── CreateAdmissionCommandHandler.java
│   │   ├── queries/
│   │   │   ├── FindAdmissionByNumberQuery.java
│   │   │   └── FindAdmissionByNumberQueryHandler.java
│   │   └── dto/
│   │       ├── AdmissionRequestDTO.java
│   │       └── AdmissionResponseDTO.java
│   ├── domain/                        ← Pure business logic, no framework dependencies
│   │   ├── model/
│   │   │   ├── Admission.java         ← Aggregate Root
│   │   │   ├── AdmissionStatus.java   ← Enum
│   │   │   └── AdmissionId.java       ← Value Object (optional)
│   │   ├── repository/
│   │   │   └── AdmissionRepository.java  ← Domain interface
│   │   └── events/
│   │       └── AdmissionCreatedEvent.java
│   └── infrastructure/                ← Technical implementations
│       ├── persistence/
│       │   └── JpaAdmissionRepository.java
│       └── messaging/                 ← Only if using events
│           └── AdmissionEventPublisher.java
│
├── billing/                           ← Another module — same structure
│   └── ...
│
├── patients/                          ← Another module
│   └── ...
│
└── shared/                            ← Shared Kernel — cross-cutting concerns ONLY
    ├── domain/
    │   ├── exception/
    │   │   ├── BusinessException.java
    │   │   ├── EntityNotFoundException.java
    │   │   └── DomainException.java
    │   └── model/
    │       └── AuditableEntity.java    ← Base class with createdAt/updatedAt
    ├── infrastructure/
    │   ├── security/                  ← JWT filter, SecurityConfig
    │   └── web/
    │       └── GlobalExceptionHandler.java
    └── api/
        └── ApiResponse.java            ← Standardized HTTP response wrapper
```

---

## Cross-Module Communication Rules

### Rule 1 — Reference by ID, never by entity

```java
// ❌ WRONG — Billing imports a domain entity from Admissions
import com.fcv.admissions.domain.model.Admission;

@Entity
public class Invoice {
    @ManyToOne                    // ← direct coupling between modules
    private Admission admission;  // Billing now depends on Admissions' JPA model
}

// ✅ CORRECT — Billing only stores the ID
@Entity
public class Invoice {
    @Column(name = "AdmisionId", nullable = false)
    private Long admissionId;     // ← simple reference, no import from another module
}
```

### Rule 2 — Consume data via Application Service, never via direct Repository

```java
// ❌ WRONG — Billing accesses Admissions' internal repository directly
@Service
public class BillingService {
    @Autowired
    private JpaAdmissionRepository admissionRepo; // ← Billing penetrates Admissions' infra
}

// ✅ CORRECT — Billing uses a public facade from Admissions
// In the Admissions module, expose a facade/service for other modules:
@Service                              // ← public, part of Admissions' api layer
public class AdmissionsQueryFacade {
    public AdmissionSummary findById(Long id) { ... }  // ← returns own DTO, not entity
}

// Billing consumes the facade:
@Service
@RequiredArgsConstructor
public class CreateInvoiceCommandHandler {
    private final AdmissionsQueryFacade admissionsFacade;  // ← allowed cross-module call

    public Long handle(CreateInvoiceCommand command) {
        AdmissionSummary admission = admissionsFacade.findById(command.admissionId());
        // ...
    }
}
```

### Rule 3 — Prefer domain events over synchronous calls

```java
// ✅ BEST for decoupled reactions — Billing reacts to Admissions events
@Component
public class AdmissionCreatedEventHandler {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onAdmissionCreated(AdmissionCreatedEvent event) {
        // Billing creates a draft invoice automatically
        // No direct dependency on Admissions
    }
}
```

### Rule 4 — Shared Kernel is minimal and stable

```
✅ Allowed in shared/:
  - Base exceptions (BusinessException, EntityNotFoundException, DomainException)
  - Auditing base class (AuditableEntity with createdAt/updatedAt)
  - Standardized API response wrapper (ApiResponse<T>)
  - JWT/Security configuration
  - Global exception handler

❌ NOT allowed in shared/:
  - Business entities (Customer, Admission, Invoice) → each goes in its own module
  - Business-specific DTOs
  - Module-specific repositories or services
  - Cross-module logic (belongs in one of the two modules or in an event)
```

---

## Spring Boot Configuration for Module Boundaries

### Option A — Package-based enforcement (lightweight, recommended for teams < 15)

```java
// In each module, mark internal classes as package-private
// (no public modifier) so they cannot be imported from other modules

// ❌ This class should NOT be accessible from Billing:
class InternalAdmissionValidator { ... }    // package-private — not exported

// ✅ Only the facade is public:
public class AdmissionsQueryFacade { ... }  // the only public surface of the module
```

### Option B — Spring Modulith (recommended for teams > 15 or stricter enforcement)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// Automatic boundary validation test — fails if a module violates another's boundary
@Test
void verifiesModularStructure() {
    ApplicationModules.of(Application.class).verify();
}

// Module event publication with guaranteed delivery (outbox pattern built-in)
@ApplicationModuleListener
public void onAdmissionCreated(AdmissionCreatedEvent event) {
    // Spring Modulith manages transactional delivery
}
```

---

## Database Strategy

### Single shared database (recommended for migrations from legacy)

```yaml
# Single datasource — one schema, all modules share tables
spring:
  datasource:
    url: jdbc:sqlserver://localhost:1433;databaseName=HospitalDB
    # All modules use the same connection pool
```

```java
// Table naming convention to identify module ownership at a glance
// Module prefix in the table name (or use a dedicated schema per module if supported)
@Table(name = "ADM_Admisiones")     // ADM_ = Admissions module
@Table(name = "FAC_Facturas")       // FAC_ = Billing module
@Table(name = "PAC_Pacientes")      // PAC_ = Patients module
@Table(name = "CFG_TiposDocumento") // CFG_ = Shared configuration
```

### Per-module schema (recommended for greenfield or when DB supports it)

```yaml
# SQL Server — separate schemas per module, same database instance
spring:
  jpa:
    properties:
      hibernate:
        default_schema: admissions  # default for this module's entities
```

```java
@Table(name = "Admisiones", schema = "admissions")
@Table(name = "Facturas",   schema = "billing")
@Table(name = "Pacientes",  schema = "patients")
```

---

## Migration Path: Modular Monolith → Microservices

When a module justifies extraction, the process is:

```
Step 1 — Verify the module is truly isolated:
  □ No entity imported from other modules
  □ All cross-module calls through facade or events
  □ No shared tables (or tables can be separated cleanly)

Step 2 — Extract the module to its own project:
  □ Copy the module package as the new project root
  □ Replace the internal facade call with an HTTP/gRPC client
  □ Replace internal events with a message broker (RabbitMQ / Kafka)

Step 3 — Deploy independently:
  □ Own Dockerfile and pipeline
  □ Own datasource (schema promoted to separate database)
  □ Register in API Gateway

Result: the monolith shrinks by one module, the extracted service is independent.
Each extraction is low risk because the boundary was already clean.
```

---

## Modular Monolith Checklist — Before Generating

```
□ Is each bounded context in its own top-level package?
□ Does the structure within each module follow: api/ application/ domain/ infrastructure/?
□ Do cross-module references use only ID (Long), not full entities?
□ Is cross-module data access through a public facade, not a direct repository?
□ Is the shared/ kernel limited to technical cross-cutting concerns only?
□ Are domain entities package-private where possible (not exported from the module)?
□ Is the table naming convention consistent (prefix or schema per module)?
□ If using Spring Modulith — is there an architectural verification test?
□ Are domain events used instead of synchronous calls for decoupled reactions?
```
