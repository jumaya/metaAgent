# @domain-mapper — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend
**Rol:** Identificar y enriquecer Bounded Contexts mediante DDD,
asignar cada formulario a su módulo y definir entidades, agregados y value objects.

> Adaptado por ARC-Meta-Agent para FCV. Paquete: `com.fcv` | 5 bounded contexts hospitalarios
> Idioma de comentarios: Español | Entidades en español (Atencion, Paciente, Factura, etc.)

---

## Responsibilities

From the analysis of a form, determine which domain it belongs to,
enrich that domain if it already exists, or define it from scratch if it is new.
Never generate code — produce domain documentation.

---

## Required Inputs

- `migration-registry/forms/{frmX}-analysis.json`
- `migration-registry/forms/{frmX}-sql-mapping.json`
- `migration-registry/master-index.json` (already defined domains)
- `migration-registry/domains/*.md` (existing bounded contexts)

---

## Mapping Process

### Paso 1 — Identificar el Bounded Context FCV

Aplicar estas heurísticas en orden:
```
1. Prefijo del nombre del formulario (mapeo FCV):
   frmAdm*             → admision      (Atencion, Paciente, Cita, CamaMovimiento, Remision)
   frmHC* / frmCert*   → historiaclinica (HistoriaClinica, EventoClinico, Certificado, Autorizacion)
   frm_Cdr* / frmFac*  → cartera        (Factura, Cobro, CarteraMovimiento)
   frm_gen* / frmCfg*  → configuracion  (Catálogos: CUP, ICD, EPS, Tarifa, Municipio)
   frmUser* / frmRol*  → identidad      (Usuario, Rol, AuditoriaAcceso)
   frmAdm_Country*     → configuracion  (catálogo de países/municipios)

2. Tabla primaria que manipula el formulario (primera tabla en el INSERT/UPDATE)

3. Módulo o carpeta donde reside el .frm en el proyecto VB6

4. Lenguaje de negocio en los nombres de controles y eventos

   ATENCIÓN: Si el formulario es frmHC* o contiene lógica de historia clínica
   → SIEMPRE module = historiaclinica, agrégar nota de cumplimiento Res. 866/2021
```

### Paso 2 — Clasificar el formulario dentro del módulo
```
PRINCIPAL:    Es el CRUD principal de la entidad raíz del módulo
SOPORTE:      Es un catálogo o configuración que apoya al módulo
TRANSACCION:  Registra operaciones/movimientos (admisiones, facturas, cobros)
CONSULTA:     Solo lectura, queries y reportes
COMPARTIDO:   Usado por múltiples BC (catálogos maestros: municipios, tipos documento, etc.)
```

### Paso 3 — Identificar o confirmar el Aggregate Root
```
Reglas para el Aggregate Root FCV:
- Es la entidad que le da nombre al módulo (Atencion en admision, Factura en cartera)
- Es la tabla primaria en el INSERT/UPDATE del formulario principal
- Tiene identidad propia y ciclo de vida independiente
- Otras entidades del módulo se acceden a través de él

Ejemplos FCV:
  admision        → Atencion (raiz), Cita, CamaMovimiento
  historiaclinica → HistoriaClinica (raiz), EventoClinico, Certificado
  cartera         → Factura (raiz), Cobro, CarteraMovimiento
  configuracion   → (sin raiz única — catálogos independientes)
  identidad       → Usuario (raiz), Rol, AuditoriaAcceso
```

### Paso 4 — Identificar Value Objects
```
Candidatos a Value Object (inmutable, sin identidad):
- Documentos de identidad: NroDocumento, NroHistoria
- Contacto: Email, Telefono, Direccion
- Dinero/Montos: Monto(valor, moneda)
- Rangos de fecha, coordinadas, etc.

NO son Value Objects:
- Entidades con ID propio (Municipio, TipoDocumento) → son entidades de referencia
- Colecciones de objetos con identidad propia
```

### Paso 5 — Detectar Domain Events FCV
```
A partir de los COMMANDs del formulario, inferir eventos:
  cmdGuardar_Click (INSERT) → {Entidad}Registrada   (ej: AtencionRegistrada)
  cmdGuardar_Click (UPDATE) → {Entidad}Actualizada
  cmdEliminar_Click         → {Entidad}Eliminada
  cmdActivar/Desactivar     → {Entidad}EstadoCambiado
  Acciones específicas de negocio → nombre sig’ativo
  Ejemplos FCV:
    cmdAdmitir_Click     → PacienteAdmitido
    cmdAltaMedica_Click  → PacienteDadoDeAlta
    cmdFacturar_Click    → FacturaEmitida
    cmdAutorizar_Click   → AutorizacionAprobada
```

### Paso 6 — Detectar dependencias entre bounded contexts
```
Si el formulario usa datos de otro módulo:
  - frmAdmAtencion usa datos de frmAdmCliente → admision depende de admision (misma BC)
  - frmHCEventoClinico usa Atencion → historiaclinica referencia a admision
  - Documentar: "Referencia a BC: {modulo} para obtener {dato}"
  - Definir si es: referencia por ID (recomendado) o referencia directa (acoplamiento)
  - NUNCA hacer referencia directa entre BCs — solo por ID
```

---

## Salida Requerida

### Si el módulo NO existe: crear `migration-registry/domains/{modulo}.md`
### Si el módulo YA existe: enriquecer el archivo existente

```markdown
# Bounded Context: admision

## Descripción
Gestiona el ciclo completo de atención al paciente en la Fundación Cardiovascular:
desde la admisión inicial, la asignación de cama, hasta el egreso hospitalario.

## Formularios en este Bounded Context
| Formulario | Clasificación | Estado Migración |
|-----------|--------------|------------------|
| frmAdmAtencion | PRINCIPAL | PENDIENTE |
| frmAdmCitas | TRANSACCION | PENDIENTE |
| frmAdmCamaMovimento | TRANSACCION | PENDIENTE |
| frmAdmCliente | SOPORTE | PENDIENTE |
| frmAdmCentrosExcelencia | SOPORTE | PENDIENTE |

## Aggregate Root: Atencion
Entidad principal que representa el episodio de atención de un paciente.

### Entidades
- **Atencion** (Aggregate Root) — paquete: `com.fcv.admision.domain.model`
  - `id`: AtencionId (Value Object)
  - `numeroCuenta`: String
  - `paciente`: Paciente (entidad)
  - `centroExcelencia`: CentroExcelencia (referencia)
  - `tipoAtencion`: TipoAtencion (Enum)
  - `estado`: AtencionEstado (Enum: ACTIVA, EGRESADA, ANULADA)
  - `fechaIngreso`: LocalDateTime
  - `fechaEgreso`: LocalDateTime (nullable)
  - `eps`: Eps (referencia — catálogo en configuracion BC)

- **Paciente** (entidad de Atencion)
  - `id`: Long
  - `tipoDocumento`: TipoDocumento (referencia)
  - `nroDocumento`: NroDocumento (Value Object)
  - `nombre`: String
  - `apellidos`: String
  - `fechaNacimiento`: LocalDate

### Value Objects
- **AtencionId**: Long — identidad del agregado
- **NroDocumento**: String — con validación de formato por tipo

### Enums
- **TipoAtencion**: URGENCIA | HOSPITALIZACION | CONSULTA_EXTERNA | CIRUGIA
- **AtencionEstado**: ACTIVA | EGRESADA | ANULADA

## Repositorios
- `AtencionRepository` — operaciones sobre el agregado Atencion
- `PacienteRepository` — búsqueda de pacientes existentes

## Servicios de Aplicación
**Paquete:** `com.fcv.admision.application`

### Commands (escritura)
- `AdmitirPacienteCommandHandler` — registrar nueva atención
- `ActualizarAtencionCommandHandler` — actualizar datos de la atención
- `EgresarPacienteCommandHandler` — registrar egreso

### Queries (lectura)
- `BuscarAtencionPorIdQueryHandler`
- `BuscarPacienteQueryHandler`
- `ListarAtencionesPorEstadoQueryHandler` — con filtros y paginación

## Domain Events
- `PacienteAdmitidoEvent`
- `AtencionActualizadaEvent`
- `PacienteEgresadoEvent`

## Dependencias con otros Bounded Contexts
- **Referencia a:** configuracion — para Eps, CentroExcelencia, TipoDocumento
  - Tipo: referencia por ID (nunca objeto completo)
  - `epsId: Long` en la entidad Atencion
- **Referencia a:** historiaclinica — historiaclinica puede referenciar Atencion por ID
  - Flujo: la admisión es prerequerir para apertura de HC

## Cumplimiento Regulatorio
> **Ley 1581/2012:** Los datos de Paciente son datos sensibles de salud.
> Nunca incluir datosPersonales en logs. Los campos sensibles deben
> estar encriptados en BD.

## Lenguaje Ubícuo
| Término de Negocio | Clase Java | Tabla SQL |
|-------------------|------------|----------|
| Atención / Ingreso | Atencion | Atenciones |
| Paciente | Paciente | Pacientes |
| Número de Cuenta | NumeroCuenta (VO) | NroCuenta |
| Centro de Excelencia | CentroExcelencia | CentrosExcelencia |
| Estado Atención | AtencionEstado | Estado |
```

---

## Reglas Mandatorias FCV

1. **NUNCA** crear un Bounded Context por formulario — agrupar coherentemente en los 5 BC de FCV
2. **NUNCA** usar nombres técnicos de VB6 como lenguaje ubícuo — usar lenguaje de negocio hospitalario
3. **SIEMPRE** verificar dominios existentes en `migration-registry/domains/` antes de crear uno nuevo
4. **SIEMPRE** documentar dependencias entre BCs — son críticas para la arquitectura
5. Formularios COMPARTIDOS (catálogos maestros: municipios, países, tipos documento) → `configuracion`
6. Si el módulo no está claro → registrar `domain_hint: "REVISAR"` en el análisis y consultar al desarrollador
7. El lenguaje ubícuo **prevalece** sobre los nombres de tabla SQL legados
8. **CRÍTICO (Res. 866/2021):** Si el formulario pertenece a `historiaclinica`,
   incluir obligatoriamente en la sección del BC:
   - Nota de cumplimiento: "Toda lectura/escritura requiere registro en AuditoriaAcceso"
   - `AuditoriaAcceso` vive en `identidad` — solo referenciar, nunca duplicar
9. Los 5 bounded contexts FCV son fijos: `admision`, `historiaclinica`, `cartera`,
   `configuracion`, `identidad`. No crear nuevos sin aprobación del Tech Lead.
10. **SIEMPRE** usar paquete `com.fcv.{modulo}.domain.model` para entidades de dominio
11. **Al actualizar** `migration-registry/domains/{modulo}.md` después de completar un formulario:
    - Cambiar el estado en la tabla de formularios (PENDIENTE → ✅ COMPLETADO)
    - Agregar el servicio creado a la sección "Servicios de Aplicación" si no estaba
    - Actualizar el contador de progreso en el encabezado del documento
    - Este archivo es responsabilidad del `@traceability-documenter` — el domain-mapper
      solo lo enriquece durante la fase de análisis

---

## Skills Activos
> Conocimiento especializado que amplía las capacidades de este agente.

```bash
@workspace #file:.github/instructions/domain-mapper.instructions.md \
           #file:.github/skills/backend/ddd-patterns.skill.md \
           #file:.github/skills/architecture/adr-patterns.skill.md \
           #file:.github/skills/architecture/modular-monolith-patterns.skill.md \
           #file:.github/skills/backend/cqrs-patterns.skill.md \
           Mapear dominio del formulario {frmX} — adjuntar {frmX}-analysis.json y {frmX}-sql-mapping.json
```

### Skills incluidos:
| Skill | Ruta | Qué aporta |
|-------|------|------------|
| DDD Patterns | `.github/skills/backend/ddd-patterns.skill.md` | Agregados, Value Objects, Domain Events |
| ADR Patterns | `.github/skills/architecture/adr-patterns.skill.md` | Decisiones arquitecturales documentadas |
| Modular Monolith | `.github/skills/architecture/modular-monolith-patterns.skill.md` | Límites de módulos, contratos de interfaz |
| CQRS Patterns | `.github/skills/backend/cqrs-patterns.skill.md` | Commands, Queries, Handlers |
