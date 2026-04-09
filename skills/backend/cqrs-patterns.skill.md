# Skill: CQRS Patterns — Command Query Responsibility Segregation
**Used by:** `@backend-generator` | **Layer:** Backend / Application

---

## Core Principle

**Commands** change state → return ID or void, never the full entity.
**Queries** read state → return DTOs, never domain entities.
**Handlers** orchestrate → contain no business logic (that belongs in the entity).

```
Controller → Command/Query → Handler → Repository/Entity → DTO (only in query)
```

---

## Command — Input Data for Writes

```java
// Immutable record — exact data the operation needs
// Name: {Verb}{Entity}Command — verb in infinitive
// CREATED by: @backend-generator | Origin: {frmX}.frm | sql_id: {id}
public record CrearAdmisionCommand(
    @NotNull Long pacienteId,
    @NotBlank @Size(max = 20) String numeroPaciente,
    @NotNull Long tipoAdmisionId,
    @NotNull Long medicoTratanteId,
    @NotNull @FutureOrPresent LocalDate fechaIngreso,
    String observaciones                              // nullable — optional field
) {}

// Update command — includes the resource ID
public record ActualizarDiagnosticoCommand(
    @NotNull Long admisionId,                        // ← ID of the resource to modify
    @NotBlank String codigoCie10,
    @NotBlank @Size(max = 500) String descripcion
) {}
```

**Command Rules:**
- Always `record` (immutable by design)
- Bean Validation annotations on every field
- Never include the full entity — only the data needed for the operation
- Resource ID goes in the command for update/delete operations

---

## CommandHandler — Orchestrates Writes

```java
// CREATED by: @backend-generator | Origin: {frmX}.frm
@Service
@RequiredArgsConstructor
public class CrearAdmisionCommandHandler {

    // Constructor injection (Lombok @RequiredArgsConstructor)
    private final AdmisionRepository admisionRepository;
    private final PacienteRepository pacienteRepository;
    private final MedicoRepository medicoRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional                                   // ← ALWAYS on CommandHandler
    public Long handle(CrearAdmisionCommand command) {

        // 1. Business validations (those Bean Validation cannot handle)
        if (admisionRepository.existsByPacienteYFecha(command.pacienteId(), command.fechaIngreso())) {
            throw new BusinessException("Patient already has an active admission on that date");
        }

        // 2. Load required dependencies
        Paciente paciente = pacienteRepository.findById(command.pacienteId())
            .orElseThrow(() -> new EntityNotFoundException("Paciente", command.pacienteId()));

        Medico medico = medicoRepository.findById(command.medicoTratanteId())
            .orElseThrow(() -> new EntityNotFoundException("Medico", command.medicoTratanteId()));

        // 3. Delegate creation to the domain (entity factory method)
        Admision admision = Admision.crear(paciente, medico, command.fechaIngreso());

        // 4. Persist
        admisionRepository.save(admision);

        // 5. Publish domain event
        eventPublisher.publishEvent(new AdmisionCreadaEvent(admision.getId(), command.pacienteId()));

        // 6. Return only the ID — never the entity
        return admision.getId();
    }
}
```

**CommandHandler Rules:**
- `@Transactional` always present
- Never return the entity — only `Long id` or `void`
- Business validations BEFORE loading dependencies (fail fast)
- Publish events WITHIN the transaction (rolled back if it fails)
- One handler per command — do not mix responsibilities

---

## Query — Search Parameters

```java
// Simple query — single search field
public record BuscarAdmisionPorNumeroQuery(
    @NotBlank String numeroAdmision
) {}

// Multi-filter query — for listings
public record ListarAdmisionesPorFechaQuery(
    LocalDate fechaDesde,                            // nullable — optional filters
    LocalDate fechaHasta,
    Long medicoId,                                   // nullable
    EstadoAdmision estado,                           // nullable
    Pageable pageable                                // pagination always present in listings
) {}
```

---

## QueryHandler — Reads Without Modifying

```java
// CREATED by: @backend-generator | Origin: {frmX}.frm | sql_id: {id}
@Service
@RequiredArgsConstructor
public class BuscarAdmisionPorNumeroQueryHandler {

    private final AdmisionRepository admisionRepository;

    @Transactional(readOnly = true)                  // ← ALWAYS readOnly on QueryHandler
    public AdmisionResponseDTO handle(BuscarAdmisionPorNumeroQuery query) {
        return admisionRepository.findByNumeroAdmision(query.numeroAdmision())
            .map(AdmisionResponseDTO::from)          // ← mapping in the DTO, not here
            .orElseThrow(() -> new EntityNotFoundException(
                "Admision", query.numeroAdmision()
            ));
    }
}

// QueryHandler with filters and pagination
@Service
@RequiredArgsConstructor
public class ListarAdmisionesPorFechaQueryHandler {

    private final AdmisionRepository admisionRepository;

    @Transactional(readOnly = true)
    public Page<AdmisionSummaryDTO> handle(ListarAdmisionesPorFechaQuery query) {
        Specification<Admision> spec = AdmisionSpecs.porFiltros(
            query.fechaDesde(), query.fechaHasta(), query.medicoId(), query.estado()
        );
        return admisionRepository.findAll(spec, query.pageable())
            .map(AdmisionSummaryDTO::from);
    }
}
```

---

## Specification — Dynamic JPA Filters

```java
// For queries with optional filters — avoids multiple queries per combination
public class AdmisionSpecs {

    // Static method that builds the Specification based on present filters
    public static Specification<Admision> porFiltros(
            LocalDate fechaDesde, LocalDate fechaHasta,
            Long medicoId, EstadoAdmision estado) {

        return Specification
            .where(desdeFecha(fechaDesde))
            .and(hastaFecha(fechaHasta))
            .and(porMedico(medicoId))
            .and(porEstado(estado));
    }

    // Each predicate handles the null case (filter not applied)
    private static Specification<Admision> desdeFecha(LocalDate desde) {
        return (root, query, cb) ->
            desde == null ? null : cb.greaterThanOrEqualTo(root.get("fechaIngreso"), desde);
    }

    private static Specification<Admision> hastaFecha(LocalDate hasta) {
        return (root, query, cb) ->
            hasta == null ? null : cb.lessThanOrEqualTo(root.get("fechaIngreso"), hasta);
    }

    private static Specification<Admision> porMedico(Long medicoId) {
        return (root, query, cb) ->
            medicoId == null ? null : cb.equal(root.get("medicoTratante").get("id"), medicoId);
    }

    private static Specification<Admision> porEstado(EstadoAdmision estado) {
        return (root, query, cb) ->
            estado == null ? null : cb.equal(root.get("estado"), estado);
    }
}
```

---

## DTO — Mapping from Entity

```java
// ResponseDTO — full projection for GET /{id}
// NEVER expose the entity directly from the Controller
public record AdmisionResponseDTO(
    Long id,
    String numeroAdmision,
    String nombrePaciente,
    String documentoPaciente,
    String nombreMedico,
    EstadoAdmision estado,
    LocalDate fechaIngreso,
    LocalDateTime creadoEn
) {
    // Static factory method — the DTO knows how to build itself from the entity
    public static AdmisionResponseDTO from(Admision admision) {
        return new AdmisionResponseDTO(
            admision.getId(),
            admision.getNumeroAdmision(),
            admision.getPaciente().getNombreCompleto(),
            admision.getPaciente().getDocumento().toString(),
            admision.getMedicoTratante().getNombreCompleto(),
            admision.getEstado(),
            admision.getFechaIngreso().toLocalDate(),
            admision.getCreadoEn()
        );
    }
}

// SummaryDTO — reduced projection for listings/tables
public record AdmisionSummaryDTO(
    Long id,
    String numeroAdmision,
    String nombrePaciente,
    EstadoAdmision estado,
    LocalDate fechaIngreso
) {
    public static AdmisionSummaryDTO from(Admision admision) {
        return new AdmisionSummaryDTO(
            admision.getId(),
            admision.getNumeroAdmision(),
            admision.getPaciente().getNombreCompleto(),
            admision.getEstado(),
            admision.getFechaIngreso().toLocalDate()
        );
    }
}

// RequestDTO — data coming from frontend to Controller
public record AdmisionRequestDTO(
    @NotNull Long pacienteId,
    @NotNull Long tipoAdmisionId,
    @NotNull Long medicoTratanteId,
    @NotNull LocalDate fechaIngreso,
    String observaciones
) {
    // Conversion to Command — the DTO knows how to build the command
    public CrearAdmisionCommand toCommand() {
        return new CrearAdmisionCommand(
            pacienteId, null, tipoAdmisionId, medicoTratanteId, fechaIngreso, observaciones
        );
    }
}
```

---

## CQRS Checklist — Before Generating

```
□ Is each Command an immutable record with Bean Validation annotations?
□ Does each CommandHandler have @Transactional?
□ Do CommandHandlers return only ID or void (never the entity)?
□ Does each QueryHandler have @Transactional(readOnly = true)?
□ Do QueryHandlers return only DTOs (never entities)?
□ Do optional filters in listings use Specification?
□ Is the entity → DTO mapping in the DTO (from() method), not in the handler?
□ Does the RequestDTO have the toCommand() method?
□ Are events published within the CommandHandler transaction?
