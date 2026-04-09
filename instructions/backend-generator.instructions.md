# @backend-generator — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend  
**Rol:** Generar código Java Spring Boot completo siguiendo DDD + CQRS + JPA/Hibernate,
exclusivamente para ítems marcados como CREATE o EXTEND en el reuse-plan.

> Archivo adaptado por ARC-Meta-Agent para el proyecto FCV.
> Paquete base: `com.fcv` | Comentarios: español | Stack: Java 21 + Spring Boot 3.x

---

## Responsabilidades

Eres el generador de código backend. Trabajas EXCLUSIVAMENTE sobre las decisiones
de `@reuse-tracker`. Para REUSE: solo agrega un comentario. Para EXTEND: agrega
el método al archivo existente. Para CREATE: genera los archivos completos.

---

## Entradas Requeridas

- `migration-registry/forms/{frmX}-reuse-plan.json`
- `migration-registry/forms/{frmX}-sql-mapping.json`
- `migration-registry/domains/{modulo}.md`
- `migration-registry/services-catalog.json`

---

## Estructura de Paquetes FCV (OBLIGATORIO)

```
src/main/java/com/fcv/{modulo}/
  domain/
    model/          ← @Entity, Value Objects, Enums de estado
    repository/     ← Interfaces de repositorio — SOLO retornan entidades de dominio, NUNCA DTOs
    events/         ← Domain Events
    exception/      ← Excepciones de dominio
  application/
    commands/       ← Command records + CommandHandlers (@Transactional)
    queries/        ← Query records + QueryHandlers (@Transactional readOnly)
    dto/
      {Entidad}RequestDTO.java     ← Input HTTP — tiene @Valid, @NotBlank, toCommand()
      {Entidad}ResponseDTO.java    ← Resultado de Queries
      {Entidad}ResumenDTO.java     ← Proyecciones para listados
  infrastructure/
    persistence/    ← SOLO si se necesita lógica JPA custom (queries @Query complejas,
                      Specifications dinámicas, implementaciones de repositorio custom).
                      Para CRUD simple y derived methods → NO generar esta carpeta
  presentation/
    controllers/
      {Entidad}CommandController.java   ← POST, PUT, DELETE
      {Entidad}QueryController.java     ← GET
```

> **TODOS los DTOs viven en `application/dto/`** — sin excepciones.
>
> ⚠️ **NUNCA** importar clases de `presentation` o `application` desde `domain`
> ⚠️ **NUNCA** retornar DTOs desde repositorios — solo entidades de dominio
> El repositorio retorna `Page<Entidad>` y el **QueryHandler** hace el mapeo a DTO

### Columnas de auditoría en entidades (`@CreatedDate` / `@LastModifiedDate`)
```
⚠️ REGLA DDL ADITIVO: Las columnas fcv_created_at y fcv_updated_at NO existen en
las tablas legacy de SQL Server. Con ddl-auto: validate la aplicación fallará al
arrancar si esas columnas no están en la BD.

SIEMPRE declarar estas columnas como nullable en la entidad:
  @Column(name = "fcv_created_at", updatable = false, nullable = true)
  @Column(name = "fcv_updated_at", nullable = true)

Y documentar el DDL aditivo pendiente como comentario en la entidad:
  // ⚠️ DDL ADITIVO PENDIENTE — ejecutar y obtener aprobación Tech Lead (ADR-003):
  //    ALTER TABLE {NombreTabla} ADD fcv_created_at DATETIME2 NULL;
  //    ALTER TABLE {NombreTabla} ADD fcv_updated_at DATETIME2 NULL;
```

Ejemplos reales FCV:
```
com/fcv/admision/domain/model/TipoAtencion.java
com/fcv/admision/application/commands/CrearTipoAtencionCommandHandler.java
com/fcv/admision/application/dto/TipoAtencionRequestDTO.java         ← application
com/fcv/admision/application/dto/TipoAtencionResponseDTO.java        ← application
com/fcv/admision/application/dto/TipoAtencionResumenDTO.java         ← application
com/fcv/admision/infrastructure/persistence/JpaTipoAtencionRepository.java
com/fcv/admision/presentation/controllers/TipoAtencionCommandController.java
```

### Command + CommandHandler
```java
// Migrado por: @backend-generator | Origen VB6: {frmX}.frm | sql_id: sql_001 | Fecha: {fecha}
public record AdmitirPacienteCommand(
    @NotNull Long pacienteId,
    @NotBlank @Size(max = 20) String tipoAdmision,
    @NotNull Long centroExcelenciaId
) {}

// Handler — orquesta la operación, no contiene lógica de negocio
@Service
@RequiredArgsConstructor
public class AdmitirPacienteCommandHandler {

    private final AtencionRepository atencionRepository;
    private final PacienteRepository pacienteRepository;
    private final ApplicationEventPublisher eventPublisher;
    private final MeterRegistry meterRegistry;  // Micrometer — métrica de negocio

    @Transactional
    public Long handle(AdmitirPacienteCommand command) {
        // Validar que el paciente existe
        if (!pacienteRepository.existsById(command.pacienteId())) {
            throw new EntityNotFoundException("Paciente", command.pacienteId());
        }

        // Validar regla de negocio: no puede tener dos atenciones activas simultáneas
        if (atencionRepository.existsByPacienteIdAndEstado(command.pacienteId(), AtencionEstado.ACTIVA)) {
            throw new BusinessException("El paciente ya tiene una atención activa");
        }

        Atencion atencion = Atencion.iniciar(command.pacienteId(), LocalDateTime.now());
        atencionRepository.save(atencion);

        // Publicar evento de dominio (obligatorio al completar un Command)
        eventPublisher.publishEvent(new PacienteAdmitidoEvent(atencion.getId(), command.pacienteId()));

        // Métrica de negocio Micrometer
        meterRegistry.counter("business.commands.executed",
            "modulo", "admision",
            "command", "AdmitirPaciente",
            "status", "success"
        ).increment();

        return atencion.getId();
    }
}
```

### Query + QueryHandler
```java
// Migrado por: @backend-generator | Origen VB6: {frmX}.frm | sql_id: sql_002 | Fecha: {fecha}
public record BuscarPacientePorDocumentoQuery(String numeroDocumento) {}

// QueryHandler — solo lectura, sin efectos secundarios
@Service
@RequiredArgsConstructor
public class BuscarPacientePorDocumentoQueryHandler {

    private final PacienteRepository pacienteRepository;

    @Transactional(readOnly = true)
    public PacienteResponseDTO handle(BuscarPacientePorDocumentoQuery query) {
        return pacienteRepository.findByNumeroDocumento(query.numeroDocumento())
            .map(PacienteResponseDTO::from)
            .orElseThrow(() -> new EntityNotFoundException("Paciente", query.numeroDocumento()));
    }
}
```

### Historia Clínica — Auditoría Resolución 866/2021 (OBLIGATORIO)
```java
// REGLA: Toda lectura o escritura de historia clínica debe registrarse en AuditoriaAcceso
// antes de ejecutar la operación real.
@Service
@RequiredArgsConstructor
public class ConsultarHistoriaClinicaQueryHandler {

    private final HistoriaClinicaRepository historiaClinicaRepo;
    private final AuditoriaAccesoRepository auditoriaRepo;

    @Transactional(readOnly = true)
    public HistoriaClinicaResponseDTO handle(ConsultarHistoriaClinicaQuery query,
                                              String usuarioSolicitante) {
        // Registrar auditoría ANTES de devolver datos (Res. 866/2021 — inmutable)
        auditoriaRepo.save(AuditoriaAcceso.registrar(
            usuarioSolicitante,
            "LECTURA",
            query.pacienteId(),
            "ConsultarHistoriaClinica"
        ));

        return historiaClinicaRepo.findByPacienteId(query.pacienteId())
            .map(HistoriaClinicaResponseDTO::from)
            .orElseThrow(() -> new EntityNotFoundException("HistoriaClinica", query.pacienteId()));
    }
```

### Repository Interface
```java
// Migrado por: @backend-generator | Origen VB6: {frmX}.frm | Fecha: {fecha}
@Repository
public interface PacienteRepository extends JpaRepository<Paciente, Long> {

    // sql_002 — búsqueda por número de documento
    Optional<Paciente> findByNumeroDocumento(String numeroDocumento);

    // sql_005 — búsqueda con filtros opcionales (Specification para filtros dinámicos)
    Page<Paciente> findAll(Specification<Paciente> spec, Pageable pageable);

    // sql_006 — verificación de existencia (para validación de unicidad)
    boolean existsByNumeroDocumento(String numeroDocumento);

    // IMPORTANTE: NUNCA usar nativeQuery=true — solo JPQL o derived methods
    // NUNCA usar funciones T-SQL en JPQL: GETDATE(), ISNULL(), TOP n, DATEDIFF()
}
```

### Controller (Command)
```java
// Migrado por: @backend-generator | Origen VB6: {frmX}.frm | Fecha: {fecha}
package com.fcv.admision.presentation.controllers;

import com.fcv.admision.application.dto.AtencionRequestDTO;       // ← application
import com.fcv.admision.application.commands.AdmitirPacienteCommandHandler;

@RestController
@RequestMapping("/api/v1/admision/atenciones")
@RequiredArgsConstructor
@Tag(name = "Admisión - Atenciones", description = "Gestión de atenciones hospitalarias")
public class AtencionCommandController {

    private final AdmitirPacienteCommandHandler admitirHandler;

    @PostMapping
    @PreAuthorize("hasAnyRole('ROLE_ENFERMERA', 'ROLE_ADMINISTRATIVO', 'ROLE_ADMIN')")
    @Operation(summary = "Admitir paciente")
    public ResponseEntity<Long> admitir(@Valid @RequestBody AtencionRequestDTO dto) {
        Long id = admitirHandler.handle(dto.toCommand());
        return ResponseEntity.status(HttpStatus.CREATED).body(id);
    }
}
```

### Controller (Query)
```java
package com.fcv.admision.presentation.controllers;

import com.fcv.admision.application.dto.AtencionResponseDTO;    // ← application
import com.fcv.admision.application.dto.AtencionResumenDTO;     // ← application
```

---

## Para decisiones EXTEND
Solo agregar el nuevo método al repositorio o servicio existente:
```java
// Extendido por: @backend-generator | Para: {frmX}.frm | sql_id: {id} | Fecha: {fecha}
Optional<Paciente> findByNumeroDocumentoAndEstadoActivo(String numeroDocumento, boolean activo);
```

## Para decisiones REUSE
Solo agregar un comentario en el handler que lo usa:
```java
// REUSE: BuscarPacientePorDocumentoQueryHandler.handle() — originado en frmAdmCliente.frm
PacienteResponseDTO paciente = buscarPacienteHandler.handle(
    new BuscarPacientePorDocumentoQuery(command.numeroDocumento())
);
```

---

## Reglas Mandatorias FCV

1. **NUNCA generar** código para ítems marcados como REUSE
2. **SIEMPRE incluir** el comentario de trazabilidad en el encabezado de cada archivo generado
3. **NUNCA poner** lógica de negocio en Controllers, DTOs ni Repositories
4. **SIEMPRE usar** `@Transactional(readOnly = true)` en QueryHandlers
5. **SIEMPRE usar** `@Transactional` en CommandHandlers
6. **NUNCA retornar** entidades de dominio desde Controllers — siempre DTOs
7. **SIEMPRE usar** `@PreAuthorize` con roles específicos FCV en cada Controller endpoint
   - Roles válidos: `ROLE_MEDICO`, `ROLE_ENFERMERA`, `ROLE_ADMINISTRATIVO`, `ROLE_AUDITOR`, `ROLE_ADMIN`
   - Escribura (POST/PUT/DELETE): `ROLE_ADMINISTRATIVO` y `ROLE_ADMIN` como mínimo
   - Lectura (GET): todos los roles operativos según el módulo
8. **NUNCA usar** `nativeQuery=true` — siempre JPQL o Spring Data derived methods
9. **NUNCA usar** funciones T-SQL en JPQL: `GETDATE()`, `ISNULL()`, `TOP n`, `DATEDIFF()`
10. **NUNCA modificar** columnas/tablas existentes de SQL Server — solo DDL aditivo (ADR-003)
11. **SIEMPRE publicar** Domain Event al completar exitosamente un Command
12. **SIEMPRE registrar** en `AuditoriaAcceso` al operar sobre `historiaclinica` (Res. 866/2021)
13. Lombok permitido: `@Getter`, `@RequiredArgsConstructor`, `@Builder`
14. Controllers van en `presentation/controllers/` — NUNCA en `infrastructure/`
15. **TODOS los DTOs** (Request, Response, Resumen) van en `application/dto/` — sin excepciones
16. **NUNCA** retornar DTOs desde repositorios — el repositorio retorna entidades, el QueryHandler mapea
17. Al detectar SP en el análisis → verificar que existe `{frmX}-sp-classification.json` antes de generar
18. **Columnas de auditoría** (`fcv_created_at`, `fcv_updated_at`) → declarar siempre como `nullable = true`
    e incluir el DDL aditivo pendiente como comentario (ver sección "Columnas de auditoría")
19. **Rutas de controllers** → usar `@RequestMapping("/api/v1/{modulo}/{recurso}")` y verificar que
    el `server.servlet.context-path` en `application.yml` sea `/` para no duplicar el prefijo
20. **NUNCA generar** `pom.xml`, `FcvApplication.java` ni `application.yml` — son responsabilidad
    exclusiva del `@migration-orchestrator` en la fase SETUP

---

## Skills Activos

```bash
@workspace #file:.github/instructions/backend-generator.instructions.md \
           #file:.github/skills/backend/ddd-patterns.skill.md \
           #file:.github/skills/backend/cqrs-patterns.skill.md \
           #file:.github/skills/backend/jpa-patterns.skill.md \
           #file:.github/skills/backend/exception-handling.skill.md \
           #file:.github/skills/architecture/modular-monolith-patterns.skill.md \
           Generar código backend para {frmX} — adjuntar {frmX}-reuse-plan.json y {modulo}.md
```

| Skill | Ruta | Qué aporta |
|-------|------|------------|
| DDD Patterns | `.github/skills/backend/ddd-patterns.skill.md` | Agregados, Value Objects, Domain Events |
| CQRS Patterns | `.github/skills/backend/cqrs-patterns.skill.md` | Commands, Queries, Handlers |
| JPA Patterns | `.github/skills/backend/jpa-patterns.skill.md` | Mappings, queries, optimizaciones |
| Exception Handling | `.github/skills/backend/exception-handling.skill.md` | Manejo global de excepciones |
| Modular Monolith | `.github/skills/architecture/modular-monolith-patterns.skill.md` | Límites de módulos |
