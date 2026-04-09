# @vb6-analyzer — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend  
**Rol:** Leer y deconstruir formularios VB6 legados de FCV, extrayendo toda la información
estructurada que los demás agentes necesitan. Solo analiza — nunca genera código.

> Adaptado por ARC-Meta-Agent para FCV. 117 formularios en `legacy/forms/` | Paquete: `com.fcv`

---

## Responsabilidades

Dado un archivo `.frm` (y opcionalmente su `.bas` asociado), producir un análisis JSON
completo que sirve como fundación para todos los demás agentes del pipeline.

### Mapeo de Módulos FCV
Usar el prefijo del formulario para asignar el `domain_hint` correcto:
```
frmAdm*         → domain_hint: "admision"       | paquete: com.fcv.admision
frmHC* / frmCert* → domain_hint: "historiaclinica" | paquete: com.fcv.historiaclinica
frmFac* / frm_Cdr* → domain_hint: "cartera"        | paquete: com.fcv.cartera
frm_gen* / frmConf* → domain_hint: "configuracion"  | paquete: com.fcv.configuracion
frmUser* / frmRol* → domain_hint: "identidad"      | paquete: com.fcv.identidad
Sin prefijo claro → domain_hint: "revisar_manualmente"
```

---

### Paso 1 — Controles UI
Identificar y clasificar cada control del formulario:

```
TextBox       → type: text_input    | capturar: Name, MaxLength, Enabled, Visible
ComboBox      → type: select        | capturar: Name, datasource (si aplica)
DataGrid      → type: data_table    | capturar: Name, columns, datasource
CheckBox      → type: checkbox      | capturar: Name, Caption
OptionButton  → type: radio         | capturar: Name, Caption, group
TabStrip/SSTab→ type: tabs          | capturar: Name, nombres de tabs
DateTimePicker→ type: date_picker   | capturar: Name, format
MaskEdBox     → type: masked_input  | capturar: Name, Mask
Label         → type: label         | capturar: Caption (es el label del campo asociado)
CommandButton → type: button        | capturar: Name, Caption
MSFlexGrid    → type: data_table    | igual que DataGrid
PictureBox    → type: image         | capturar: Name
```

### Paso 2 — Eventos y Lógica de Negocio
Para cada evento relevante extraer:
- Nombre del evento
- Tipo: `COMMAND` (modifica datos) o `QUERY` (lee datos) o `NAVIGATION` (abre otro formulario)
- Queries SQL referenciadas (inline o vía llamada a función)
- Lógica de negocio embebida (validaciones, cálculos, transformaciones)
- Referencias a variables globales (`gUsuario`, `gEmpresa`, `etc.`)

Eventos VB6 más comunes para buscar:
```
Form_Load          → inicialización, carga de combos, datos iniciales
cmdGuardar_Click   → COMMAND — insertar o actualizar
cmdBuscar_Click    → QUERY — búsqueda
cmdEliminar_Click  → COMMAND — eliminar
cmdNuevo_Click     → limpiar formulario para registro nuevo
cmdCancelar_Click  → NAVIGATION — cerrar/cancelar
txtCampo_Change    → validación en tiempo real
cboX_Click         → carga dependiente (ej: seleccionar EPS carga planes)
```

### Paso 3 — Queries SQL
Extraer TODOS los queries SQL, incluyendo:
- SQL inline (concatenado con variables VB6)
- Llamadas a stored procedures (`EXEC sp_nombre`)
- Uso de ADO (`rs.Open "SELECT...", conn`)
- Construcción dinámica de SQL con IF/variables

Para cada query:
- Asignar ID secuencial: sql_001, sql_002...
- Clasificar como COMMAND (INSERT/UPDATE/DELETE) o QUERY (SELECT)
- Identificar tablas involucradas
- Identificar parámetros (variables VB6 usadas en el query)

**Tablas comunes FCV a reconocer:**
`Atencion`, `Paciente`, `CamaMovimiento`, `Cita`, `HistoriaClinica`, `Factura`, `Autorizacion`

### Paso 4 — Referencias a Formularios
Buscar:
```vb
Form.Show          → referencia de navegación
Load frmX          → referencia de carga
frmX.txtCampo.Text → referencia de propiedad a otro formulario (acoplamiento)
Unload frmX        → cierre programático
```

### Paso 5 — Variables Globales
Buscar uso del módulo `.bas` y variables globales:
```vb
Public gUsuario As String    → variable global de sesión
Public gEmpresa As Integer   → variable global de contexto
Public gCentroExcelencia As Integer → centro de excelencia activo (FCV)
Module1.fnCalcular()         → función de módulo compartido
```

### Paso 6 — Clasificación de Complejidad
```
BAJA:   Solo CRUD simple, 1-3 queries, sin lógica de negocio compleja
MEDIA:  CRUD con validaciones, 4-8 queries, algo de lógica de negocio
ALTA:   Lógica de negocio compleja embebida, 8+ queries, cálculos, múltiples dependencias
CRÍTICA: SPs, triggers, lógica de facturación/autorización médica — requiere revisión manual
```

---

## Output Requerido

Generar el archivo `migration-registry/forms/{frmX}-analysis.json`:

```json
{
  "_execution": {
    "agent": "vb6-analyzer",
    "form": "{frmX}",
    "status": "SUCCESS | PARTIAL | FAILED",
    "timestamp": "{ISO-8601}",
    "warnings_count": 0,
    "degradation_notes": "",
    "context_forward": {
      "for_agent": "sp-migration-planner",
      "_protocol_note": "El destinatario SIEMPRE es sp-migration-planner. Si sp_count=0, el orchestrator puede omitir sp-migration-planner pero el protocolo de cadena no cambia.",
      "alerts": []
    }
  },
  "form_name": "frmAdmAtencion",
  "domain_hint": "admision",
  "target_package": "com.fcv.admision",
  "complexity": "MEDIA",
  "sp_count": 0,
  "stored_procedures": [],
  "ui_controls": [
    {
      "name": "txtNroDocumento",
      "type": "text_input",
      "label": "Número de Documento",
      "validations": ["required", "maxLength:20"],
      "enabled": true,
      "visible": true
    },
    {
      "name": "cboTipoDocumento",
      "type": "select",
      "label": "Tipo de Documento",
      "datasource": "TiposDocumento"
    }
  ],
  "events": [
    {
      "name": "cmdGuardar_Click",
      "cqrs_type": "COMMAND",
      "sql_refs": ["sql_001"],
      "business_logic": "Valida campos requeridos, verifica paciente activo, registra atención",
      "validations_found": ["NroDoc no vacío", "TipoDoc seleccionado", "EPS seleccionada"]
    },
    {
      "name": "Form_Load",
      "cqrs_type": "QUERY",
      "sql_refs": ["sql_002", "sql_003"],
      "business_logic": "Carga combo TiposDocumento y combo EPS"
    }
  ],
  "sql_queries": [
    {
      "id": "sql_001",
      "raw_sql": "INSERT INTO Atencion (PacienteId, EpsId, FechaIngreso, Estado) VALUES (?)",
      "cqrs_type": "COMMAND",
      "tables": ["Atencion"],
      "parameters": ["txtNroDocumento.Text", "cboEps.ItemData", "Now()", "'ACTIVA'"],
      "event_source": "cmdGuardar_Click"
    },
    {
      "id": "sql_002",
      "raw_sql": "SELECT TipoDocId, Descripcion FROM TiposDocumento WHERE Activo = 1",
      "cqrs_type": "QUERY",
      "tables": ["TiposDocumento"],
      "parameters": [],
      "event_source": "Form_Load"
    }
  ],
  "form_references": [
    { "form": "frmAdmCitas", "type": "NAVIGATION" },
    { "form": "frmAdmCliente", "type": "DATA_DEPENDENCY" }
  ],
  "global_variables_used": ["gUsuario", "gEmpresa", "gCentroExcelencia"],
  "bas_modules_used": ["modGlobales", "modValidaciones"],
  "analyst_notes": "El campo NroDocumento y TipoDocumento identifican unívocamente al paciente. Verificar lógica de unicidad."
}
```

### Valores del campo `_execution.status`
```
SUCCESS  → el .frm fue analizado completamente, ningún elemento omitido
PARTIAL  → el .frm fue analizado pero con degradación:
           ejemplos: SQL dinámico no parseable, controles no reconocidos,
           módulo .bas referenciado no encontrado
FAILED   → no fue posible producir un output utilizable
           el orchestrator activará el mecanismo de retry/bloqueo
```

### Reglas para `context_forward`
El destinatario **SIEMPRE es `sp-migration-planner`** — nunca `sql-analyzer` directamente.
El orchestrator decide si omite `sp-migration-planner` cuando `sp_count = 0`,
pero la cadena de protocolo no cambia:
```
Poblar context_forward.alerts cuando:
  - Stored procedures detectados → "Formulario usa {N} SPs: {lista}. @sp-migration-planner debe clasificarlos."
  - SQL dinámico muy complejo → "sql_{id} tiene SQL dinámico con {N} condiciones. Considerar Specification."
  - Módulo .bas con lógica compartida → "modX.bas contiene lógica de {descripción}. Revisar en SP analysis."
  - Complejidad ALTA o CRÍTICA → "Formulario complejidad {nivel}. Esperar múltiples queries y lógica embebida."

Si no hay alertas → context_forward.alerts = []
```

### Regla de decisión para el orchestrator — sp_count
```
sp_count = 0  → orchestrator PUEDE omitir la invocación de @sp-migration-planner
                 y continuar directamente con @sql-analyzer.
                 El agente de trazabilidad creará de todas formas el archivo
                 {frmX}-sp-classification.json con applies: false para mantener
                 el registro completo del pipeline.

sp_count > 0  → orchestrator DEBE invocar @sp-migration-planner ANTES que @sql-analyzer.
                 @sql-analyzer necesita el archivo {frmX}-sp-classification.json
                 para saber qué SPs son categoría A/B/C antes de mapear.
