# Skill: Global Exception Handling
**Used by:** `@backend-generator` | **Layer:** Backend / Infrastructure

---

## Domain Exception Hierarchy

```java
// Base domain exception — all others extend this
public class DomainException extends RuntimeException {
    private final String errorCode;

    public DomainException(String message) {
        super(message);
        this.errorCode = "DOMAIN_ERROR";
    }

    public DomainException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// Specific exceptions — clear semantics for each case
public class EntityNotFoundException extends DomainException {
    public EntityNotFoundException(String entity, Object id) {
        super("NOT_FOUND", entity + " not found with id: " + id);
    }
}

public class BusinessException extends DomainException {
    public BusinessException(String message) {
        super("BUSINESS_RULE_VIOLATION", message);
    }
}

public class DuplicateEntityException extends DomainException {
    public DuplicateEntityException(String message) {
        super("DUPLICATE_ENTITY", message);
    }
}

public class InvalidStateException extends DomainException {
    public InvalidStateException(String message) {
        super("INVALID_STATE", message);
    }
}
```

---

## @RestControllerAdvice — Global Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // Standardized error response structure
    public record ErrorResponse(
        String errorCode,
        String message,
        String path,
        LocalDateTime timestamp,
        List<FieldError> fieldErrors    // only for validation errors
    ) {
        public record FieldError(String field, String message) {}

        // Constructor without fieldErrors
        public ErrorResponse(String errorCode, String message, String path) {
            this(errorCode, message, path, LocalDateTime.now(), null);
        }
    }

    // 404 — Entity not found
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            EntityNotFoundException ex, HttpServletRequest req) {
        log.warn("Entity not found: {} | path: {}", ex.getMessage(), req.getRequestURI());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), req.getRequestURI()));
    }

    // 422 — Business rule violated
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(
            BusinessException ex, HttpServletRequest req) {
        log.warn("Business rule violated: {} | path: {}", ex.getMessage(), req.getRequestURI());
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), req.getRequestURI()));
    }

    // 409 — Duplicate
    @ExceptionHandler(DuplicateEntityException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(
            DuplicateEntityException ex, HttpServletRequest req) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), req.getRequestURI()));
    }

    // 400 — Bean Validation errors (@Valid)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest req) {

        List<ErrorResponse.FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> new ErrorResponse.FieldError(e.getField(), e.getDefaultMessage()))
            .toList();

        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(
                "VALIDATION_ERROR",
                "Validation error in submitted data",
                req.getRequestURI(),
                LocalDateTime.now(),
                fieldErrors
            ));
    }

    // 400 — Invalid path/query parameters
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraint(
            ConstraintViolationException ex, HttpServletRequest req) {
        String message = ex.getConstraintViolations().stream()
            .map(v -> v.getPropertyPath() + ": " + v.getMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("CONSTRAINT_VIOLATION", message, req.getRequestURI()));
    }

    // 403 — Access denied (@PreAuthorize)
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
            AccessDeniedException ex, HttpServletRequest req) {
        log.warn("Access denied: {} | path: {}", ex.getMessage(), req.getRequestURI());
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ErrorResponse("ACCESS_DENIED",
                "You do not have permission to perform this operation", req.getRequestURI()));
    }

    // 409 — Version conflict (optimistic locking)
    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    public ResponseEntity<ErrorResponse> handleOptimisticLock(
            ObjectOptimisticLockingFailureException ex, HttpServletRequest req) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("OPTIMISTIC_LOCK",
                "The record was modified by another user. Please reload and try again.",
                req.getRequestURI()));
    }

    // 500 — Any unhandled exception
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(
            Exception ex, HttpServletRequest req) {
        // Log with full stacktrace — NEVER expose to the client
        log.error("Unhandled error | path: {} | error: {}", req.getRequestURI(), ex.getMessage(), ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR",
                "Internal server error. Please contact the administrator.", req.getRequestURI()));
    }
}
```

---

## Standard Error Response — Examples

```json
// 404 — Entity not found
{
  "errorCode": "NOT_FOUND",
  "message": "Paciente not found with id: 999",
  "path": "/api/pacientes/999",
  "timestamp": "2024-01-15T10:30:00"
}

// 422 — Business rule
{
  "errorCode": "BUSINESS_RULE_VIOLATION",
  "message": "Patient already has an active admission on that date",
  "path": "/api/admisiones",
  "timestamp": "2024-01-15T10:30:00"
}

// 400 — Field validation
{
  "errorCode": "VALIDATION_ERROR",
  "message": "Validation error in submitted data",
  "path": "/api/pacientes",
  "timestamp": "2024-01-15T10:30:00",
  "fieldErrors": [
    { "field": "documentoNumero", "message": "must not be blank" },
    { "field": "fechaNacimiento", "message": "must be a past date" }
  ]
}
```

---

## Usage in Entities and Handlers

```java
// In the entity — throw DomainException with descriptive code
public void cerrarAdmision(String motivoCierre) {
    if (this.estado == EstadoAdmision.CERRADA) {
        throw new InvalidStateException("Admission has already been closed");
    }
    if (motivoCierre == null || motivoCierre.isBlank()) {
        throw new DomainException("MOTIVO_REQUERIDO", "Closing reason is required");
    }
    this.estado = EstadoAdmision.CERRADA;
    this.motivoCierre = motivoCierre;
    this.fechaCierre = LocalDateTime.now();
}

// In the handler — throw EntityNotFoundException with entity name
Paciente paciente = pacienteRepo.findById(command.pacienteId())
    .orElseThrow(() -> new EntityNotFoundException("Paciente", command.pacienteId()));
```

---

## Exception Checklist — Before Generating

```
□ Is there a @RestControllerAdvice handling all domain exceptions?
□ Does the error response always follow the ErrorResponse structure?
□ Do exceptions not leak stacktraces to the client (only logged)?
□ Does EntityNotFoundException use the (entity, id) constructor?
□ Do 500 errors use log.error() with the full stacktrace?
□ Does optimistic locking (409) give a useful message to the user?
□ Do validation errors list ALL invalid fields?
