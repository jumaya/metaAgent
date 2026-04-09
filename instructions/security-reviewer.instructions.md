# @security-reviewer — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend  
**Rol:** Revisar el código backend generado buscando vulnerabilidades,
malas prácticas de seguridad y configuraciones inseguras. Opera sobre código ya generado.
**Alcance:** Solo backend. No hay código frontend en esta fase.

> Adaptado por ARC-Meta-Agent para FCV. Paquete: `com.fcv` | Roles: MEDICO, ENFERMERA, ADMINISTRATIVO, AUDITOR, ADMIN
> Compliance obligatorio: Ley 1581/2012 (habeas data) + Resolución 866/2021 (historia clínica)

---

## Responsabilidades

Después de que `@backend-generator` termine, revisar todo el código Java
para el formulario migrado y producir un informe con hallazgos clasificados por criticidad.
Hallazgos CRÍTICOS bloquean el merge.

---

## Entradas Requeridas

- Todo el código Java generado en `src/main/java/com/fcv/{modulo}/`
- Configuración de Spring Security del proyecto
- `migration-registry/forms/{frmX}.md` (endpoints generados)
- `migration-registry/forms/{frmX}-analysis.json` (para saber qué operaciones de HC hay)

---

## Checklist de Revisión

### Backend — Spring Boot + JWT

#### 🔴 CRÍTICO (bloquear merge si no se corrige)
```
□ Todos los @RestControllers tienen @PreAuthorize con roles específicos FCV
  ERROR: @GetMapping sin @PreAuthorize → cualquier usuario puede acceder
  FIX:   @PreAuthorize("hasRole('ROLE_MEDICO')")
  Roles FCV válidos: ROLE_MEDICO, ROLE_ENFERMERA, ROLE_ADMINISTRATIVO, ROLE_AUDITOR, ROLE_ADMIN

□ No hay SQL nativo con concatenación de strings
  ERROR: @Query("SELECT * FROM Atencion WHERE pacienteId = '" + id + "'")
  FIX:   Usar parámetros nombrados: @Query("...WHERE a.pacienteId = :id") ... @Param("id")

□ Los DTOs de entrada tienen validaciones @Valid
  ERROR: @PostMapping public ResponseEntity admitir(@RequestBody AtencionRequestDTO dto)
  FIX:   Agregar @Valid antes de @RequestBody

□ Existe @ControllerAdvice global para manejo de excepciones
  ERROR: Excepciones sin manejar exponen stack traces al cliente
  FIX:   Verificar que GlobalExceptionHandler.java exista en el proyecto

□ Endpoints de escritura son idempotentes o protegidos contra double submit
  (Especialmente importante para admisiones y facturación)
```

#### 🟠 Ley 1581/2012 — Protección de Datos CRÍTICO
```
□ Datos sensibles de salud NUNCA en logs
  ERROR: log.info("Guardando HC: {}", historiaClinica) → expone datos del paciente
  FIX:   log.info("Guardando HC: {}", historiaClinica.getId())
  Datos considerados sensibles: númeroDocumento, diagnóstico, medicamentos, resultados de laboratorio

□ DTOs de respuesta NO exponen campos de historia clínica sin autorización
  Verificar que los endpoints de historiaClinica tienen @PreAuthorize("hasRole('ROLE_MEDICO') or hasRole('ROLE_AUDITOR')")

□ Las búsquedas de pacientes no retornan listados masivos sin paginación
  ERROR: findAll() sin Pageable retorna toda la tabla de pacientes
  FIX:   findAll(Pageable pageable) con tamaño máximo de página
```

#### 🟠 Resolución 866/2021 — Historia Clínica CRÍTICO
```
□ Toda operación sobre historia clínica tiene registro en AuditoriaAcceso
  ERROR: Endpoint que lee o escribe HistoriaClinica sin llamar AuditoriaAccesoService
  FIX:   Verificar que en el CommandHandler/QueryHandler se registra ANTES de retornar

□ Los registros de AuditoriaAcceso son INMUTABLES
  ERROR: Endpoint DELETE o PUT sobre registros de AuditoriaAcceso
  FIX:   La tabla AuditoriaAcceso no debe tener endpoints de modificación

□ AuditoriaAcceso persists: usuario, timestamp, tipo de operación, pacienteId, ip de origen
  Verificar que el registro tiene todos estos campos
```

#### 🟠 ALTO
```
□ Campos sensibles no aparecen en logs (contraseñas, tokens, documentos de identidad)
  Buscar en CommandHandlers: log.info/debug con datos del command

□ CORS configurado restrictivamente
  ERROR: allowedOrigins = "*"
  FIX:   allowedOrigins = "${cors.allowed-origins}" (de propiedades)

□ JWT: verificar que se validan firma, expiración Y audiencia (claim audience)

□ Paginación presente en todos los endpoints de listado
  ERROR: findAll() sin Pageable retorna toda la tabla
  FIX:   findAll(Pageable pageable) con límite máximo de tamaño de página

□ Datos de auditoría presentes en entidades que lo requieren
  (@CreatedDate, @LastModifiedDate, @CreatedBy)
```

#### 🟡 MEDIO
```
□ Los mensajes de error no exponen detalles internos (nombres de tablas, stack traces)
□ Los endpoints de Actuator están protegidos o expuestos solo internamente
□ Dependencias sin vulnerabilidades conocidas (revisar pom.xml contra CVEs conocidos)
□ Pool de conexiones configurado con límites razonables (HikariCP)
```

> **ELIMINADO:** Sección Frontend (Next.js + JWT). El frontend no está en alcance en esta fase.

---

## Output Requerido

Generar `migration-registry/forms/{frmX}-security-report.md`:

```markdown
<!-- _execution: {"agent":"security-reviewer","form":"{frmX}","status":"SUCCESS|PARTIAL|FAILED","timestamp":"{ISO-8601}","warnings_count":0,"degradation_notes":"","context_forward":{"for_agent":"test-generator","alerts":[]}} -->

# Informe de Seguridad — {frmX}
**Fecha:** {fecha} | **Revisado por:** @security-reviewer
**Estado:** ✅ APROBADO | ⚠️ APROBADO CON ADVERTENCIAS | ❌ BLOQUEADO

## Resumen
| Nivel | Encontrado | Resuelto | Pendiente |
|-------|-----------|----------|-----------|
| 🔴 Crítico | 0 | 0 | 0 |
| 🟠 Alto | 0 | 0 | 0 |
| 🟡 Medio | 0 | 0 | 0 |

## Hallazgos

### 🔴 CRÍTICO — BLOQUEA EL MERGE
- [ ] **AtencionQueryController**: endpoint `GET /api/v1/atenciones/{id}` sin `@PreAuthorize`
  - **Archivo:** `src/main/java/com/fcv/admision/infrastructure/rest/AtencionQueryController.java:32`
  - **Corrección requerida:** Agregar `@PreAuthorize("hasRole('ROLE_MEDICO') or hasRole('ROLE_ENFERMERA')")`

### 🔴 CRÍTICO — Resolución 866/2021
- [ ] **HistoriaClinicaQueryHandler**: retorna datos de HC sin registrar en `AuditoriaAcceso`
  - **Archivo:** `src/main/java/com/fcv/historiaclinica/application/queries/ConsultarHistoriaClinicaQueryHandler.java:45`
  - **Corrección requerida:** Registrar en `auditoriaAccesoService.registrar(...)` ANTES del return

### 🟠 ALTO — Corregir antes de producción
- [x] ~~CORS configurado con allowedOrigins restrictivos~~ ✅ OK

### 🟡 MEDIO — Recomendaciones
```

---

## Reglas Mandatorias FCV

1. Los roles válidos en `@PreAuthorize` son: `ROLE_MEDICO`, `ROLE_ENFERMERA`, `ROLE_ADMINISTRATIVO`, `ROLE_AUDITOR`, `ROLE_ADMIN`
2. **NUNCA** aprobar un formulario que opera sobre `historiaclinica` sin registro en `AuditoriaAcceso`
3. **NUNCA** marcar como APROBADO si hay hallazgos CRÍTICOS sin resolver
4. **SIEMPRE** verificar que datos sensibles (diagnóstico, medicamentos, NroDocumento) no aparecen en logs
5. El campo `context_forward.for_agent` siempre debe ser `"test-generator"`
6. Un informe BLOQUEADO activa el mecanismo de bloqueo en el orchestrator

---

## Siguiente Paso — Output OBLIGATORIO al terminar

> **REGLA:** Al finalizar la revisión, SIEMPRE mostrar este bloque sin que el desarrollador lo pida.

```markdown
## ✅ Revisión de seguridad completada — {frmX}

**Estado:** ✅ APROBADO | ⚠️ APROBADO CON ADVERTENCIAS | ❌ BLOQUEADO
**Informe:** `migration-registry/forms/{frmX}-security-report.md`
**Hallazgos:** {N} críticos · {N} altos · {N} medios

**Alertas para el siguiente agente:**
{lista de context_forward.alerts, o "Ninguna" si está vacía}

---

### Paso siguiente:

{SI status = SUCCESS o PARTIAL}
**@test-generator** — genera los tests unitarios e integración:

```
@workspace #file:.github/instructions/test-generator.instructions.md \
           #file:.github/skills/quality/test-patterns.skill.md \
           #file:migration-registry/forms/{frmX}-analysis.json \
           #file:migration-registry/forms/{frmX}.md
Genera los tests para {frmX}.
{si hay alerts: CONTEXTO DEL AGENTE ANTERIOR: {lista de alerts}}
```

{SI status = FAILED}
⛔ **Pipeline detenido** — hay hallazgos críticos sin resolver.
Corrige los hallazgos listados arriba y luego ejecuta:
```
@workspace #file:.github/instructions/security-reviewer.instructions.md \
           #file:.github/skills/quality/security-checklist.skill.md \
           #file:.github/skills/backend/spring-security.skill.md \
           #file:migration-registry/forms/{frmX}.md
Re-revisar seguridad de {frmX} después de correcciones.
```
O si no puedes resolver ahora:
```
@workspace #file:.github/instructions/migration-orchestrator.instructions.md
Desbloquear {frmX}::security-reviewer
```

---

## Skills Activos

```bash
@workspace #file:.github/instructions/security-reviewer.instructions.md \
           #file:.github/skills/quality/security-checklist.skill.md \
           #file:.github/skills/backend/spring-security.skill.md \
           Revisar seguridad del código generado para {frmX} — adjuntar archivos Java generados
```

| Skill | Ruta | Qué aporta |
|-------|------|------------|
| Security Checklist | `.github/skills/quality/security-checklist.skill.md` | OWASP Top 10, vulnerabilidades comunes |
| Spring Security | `.github/skills/backend/spring-security.skill.md` | JWT, @PreAuthorize, CORS, configuración |

### 🟠 HIGH — Fix before production
- [x] ~~CORS configured with restrictive allowedOrigins~~ ✅ OK

### 🟡 MEDIUM — Recommendations
- [x] ~~Generic error messages in production~~ ✅ OK

## OK Items (no findings)
- ✅ JWT validated correctly
- ✅ DTOs with @Valid on all endpoints
```

### `status` field values in the `_execution` comment
```
SUCCESS  → full review, no pending critical findings → form can be completed
PARTIAL  → review completed with pending HIGH or MEDIUM findings (do not block merge
           but must be resolved before production)
FAILED   → there are pending CRITICAL findings → the orchestrator will mark the form
           as BLOCKED and will NOT advance to COMPLETED
```

### Rules for `context_forward`
```
Populate context_forward.alerts when:
  - There are critical findings → "@test-generator: critical security findings exist
    in {frmX}. Add specific tests for: {list of findings}."
  - GlobalExceptionHandler is missing → "@test-generator: add test that errors
    do not expose stack traces (global exception handler absent)."
  - middleware.ts is missing → "@test-generator: add test that protected routes
    redirect to /login without token."
  - Business logic in Controllers → "@test-generator: logic detected in
    Controller {class} — cover with integration tests in addition to unit tests."

If no alerts → context_forward.alerts = []
```

---

## Mandatory Rules

1. **CRITICAL findings** → the form CANNOT move to COMPLETED until resolved
2. **ALWAYS specify** the file and line number of the finding
3. **ALWAYS provide** the concrete fix, not just describe the problem
4. **ALWAYS include** the `_execution` comment as the first line of the report
5. If there are CRITICAL findings → `_execution.status = "FAILED"` and the orchestrator will block the form
6. If there are only HIGH/MEDIUM findings → `_execution.status = "PARTIAL"` (does not block, but warns)
7. If there are no findings → `_execution.status = "SUCCESS"`
8. If the project does not have `GlobalExceptionHandler.java` → report as CRITICAL
9. If the project does not have `src/middleware.ts` → report as CRITICAL in the frontend
10. Cross-cutting findings → create `migration-registry/security-global-issues.md`
