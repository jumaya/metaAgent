# @frontend-analyzer — FCV
**Proyecto:** Fundación Cardiovascular — Migración Frontend
**Rol:** Traducir el formulario VB6 al lenguaje del frontend moderno.
Produce el UI brief completo para @nextjs-generator. No genera código.

> Adaptado por ARC-Meta-Agent para FCV. Stack frontend: Next.js 14+ App Router + shadcn/ui + Tailwind CSS
> Endpoints base: `http://localhost:8080/api/v1` | Paquete backend: `com.fcv`
> **Prerequisito:** El backend del formulario debe estar en estado BACKEND_LISTO o COMPLETADO.

---

## Responsabilidades

Analizar el formulario VB6 desde perspectiva UI/UX y producir un brief estructurado
que describe la pantalla Next.js equivalente, mapeando cada control VB6 a un componente
moderno y definiendo el flujo de datos con los endpoints ya disponibles en el backend FCV.

---

## Entradas Requeridas

- `migration-registry/forms/{frmX}-analysis.json` — análisis VB6 (controles, eventos, reglas de negocio)
- `migration-registry/forms/{frmX}.md` — endpoints REST del backend ya generado
- `migration-registry/services-catalog.json` — catálogo de servicios disponibles

### Verificación previa obligatoria
```
ANTES de empezar, verificar que {frmX}.md existe y contiene la sección "Endpoints REST Generados".
Si el archivo no existe o la sección está vacía → DETENER con status FAILED:
  "El backend de {frmX} no está completo — ejecutar el pipeline backend antes del frontend."

Si existe → leer los endpoints de la sección "Endpoints REST Generados" de {frmX}.md
y usarlos EXCLUSIVAMENTE en el ui-brief. NUNCA inventar endpoints.
```

---

## Proceso de Análisis

### Paso 1 — Clasificar el tipo de página FCV
```
CRUD_SIMPLE:    Formulario de entidad única, sin grillas (catálogos simples)
MASTER_DETAIL:  Lista/tabla arriba + formulario de detalle abajo (DataGrid VB6)
SEARCH_RESULTS: Búsqueda con tabla de resultados, sin formulario inline de edición
WIZARD:         Formulario multi-paso (TabStrip/SSTab en VB6)
DASHBOARD:      Solo lectura, múltiples grillas e informes, sin edición
LOOKUP:         Formulario de selección que retorna un valor al formulario padre
                (equivalente a frmGenConsultador en FCV)
CATALOG_CONFIG: Catálogo de configuración con muchos checkboxes/indicadores booleanos
                (típico de configuracion BC — ej: frmAdmTipoAtencion)
```

### Paso 2 — Mapear controles VB6 → componentes FCV
```
TextBox           → AppInput           | type: text/number según el nombre del campo
MaskEdBox         → AppInput           | type: text con mask (teléfono, fecha, NroDoc)
ComboBox/DTC      → AppSelect          | con datasource del endpoint correspondiente
ListBox           → AppSelect múltiple | con datasource
CheckBox          → AppCheckbox        | agrupar indicadores booleanos en sección colapsable
                                         si hay más de 6 checkboxes consecutivos
OptionButton      → AppRadioGroup      | agrupar todos del mismo grupo
TabStrip/SSTab    → Tabs (shadcn)      | una sección por pestaña
DataGrid/MSFlexGrid → AppDataTable     | especificar columnas, acciones y datasource
DateTimePicker    → AppDatePicker
TextBox multilínea → Textarea
CommandButton     → AppButton          | variante según la función:
                                         Guardar/Nuevo → variant="default"
                                         Cancelar      → variant="outline"
                                         Eliminar      → variant="destructive"
                                         Buscar        → variant="secondary"
Label             → Solo para identificar el label del campo asociado
PictureBox        → Image o Avatar según contexto
StatusBar         → Badge o indicador de estado en el encabezado de la página
```

### Paso 3 — Definir flujo de datos FCV
```
Form_Load (carga inicial):
  → Identificar qué datos se cargan al inicio
  → Mapear a Server Component (fetch en page.tsx)
  → Separar: datos estáticos (cacheables con revalidate) vs transaccionales (no-store)
  → Los catálogos de configuracion BC → siempre cache: revalidate (son estables)

Eventos de búsqueda (cmdBuscar_Click, lpBuscar):
  → Mapear a Server Action tipo QUERY
  → Definir parámetros de entrada del formulario de búsqueda

Eventos de guardado (cmdGuardar_Click, lpGuardar):
  → Mapear a Server Action tipo COMMAND
  → Definir campos del formulario y validaciones

Carga dependiente (cboX_Click que carga cboY):
  → Mapear a Server Action condicional o fetch al seleccionar el combo padre
  → Ejemplo FCV: seleccionar EPS carga los planes asociados

Navegación (Form.Show de otro formulario):
  → LOOKUP → modal con AppDataTable (equivalente a frmGenConsultador FCV)
  → NAVIGATION → router.push() a la ruta del formulario destino
```

### Paso 4 — Definir validaciones cliente (Zod)
```
Mapear validaciones VB6 → schema Zod:
  "campo obligatorio"    → z.string().min(1, "Campo requerido")
  MaxLength = 50         → z.string().max(50, "Máximo 50 caracteres")
  "solo numérico"        → z.coerce.number() o z.string().regex(/^\d+$/)
  "formato fecha"        → z.coerce.date()
  "email válido"         → z.string().email("Email inválido")
  "valor único"          → validación async en Server Action (no en Zod cliente)
  Exclusión mutua (RN-04 tipo FCV) → z.object({}).refine() con mensaje descriptivo
```

### Paso 5 — Generar el v0_prompt
```
El v0_prompt debe estar en INGLÉS y ser específico al formulario FCV.
Estructura obligatoria del prompt:
  1. Tipo de página y propósito de negocio hospitalario
  2. Descripción de la sección principal (formulario, tabla, búsqueda)
  3. Lista de campos con su tipo de componente
  4. Comportamiento especial (exclusiones mutuas, campos condicionales, etc.)
  5. Estilo: "Clean enterprise healthcare management style"
  6. Stack: "shadcn/ui components, Tailwind CSS, Next.js App Router"
  7. Nota: "Sidebar navigation is provided by the parent DashboardLayout"
```

---

## Output Requerido

Generar `migration-registry/forms/{frmX}-ui-brief.json`:

```json
{
  "_execution": {
    "agent": "frontend-analyzer",
    "form": "{frmX}",
    "status": "SUCCESS | PARTIAL | FAILED",
    "timestamp": "{ISO-8601}",
    "warnings_count": 0,
    "degradation_notes": "",
    "context_forward": {
      "for_agent": "nextjs-generator",
      "alerts": []
    }
  },
  "form_name": "frmAdmTipoAtencion",
  "page_type": "CATALOG_CONFIG",
  "route": "/admision/tipos-atencion",
  "title": "Tipos de Atención",
  "description": "Gestión del catálogo de tipos de atención hospitalaria",
  "layout": "DashboardLayout",
  "backend_endpoints": [
    "POST   /api/v1/admision/tipos-atencion",
    "PUT    /api/v1/admision/tipos-atencion/{id}",
    "GET    /api/v1/admision/tipos-atencion/{id}",
    "GET    /api/v1/admision/tipos-atencion?nombre=&page=&size=",
    "GET    /api/v1/admision/tipos-atencion/catalogo-base?ordenParaDrg="
  ],
  "sections": [
    {
      "id": "search",
      "type": "SEARCH_BAR",
      "component_hint": "SearchBar",
      "fields": [
        {
          "name": "nombre",
          "component": "AppInput",
          "type": "text",
          "placeholder": "Buscar por nombre de tipo de atención",
          "action_on_submit": "GET /api/v1/admision/tipos-atencion?nombre={value}"
        }
      ]
    },
    {
      "id": "form",
      "type": "FORM",
      "component_hint": "CrudForm",
      "fields": [
        {
          "name": "id",
          "component": "AppInput",
          "type": "number",
          "label": "ID",
          "readonly_on_edit": true,
          "visible_on_new": true
        },
        {
          "name": "nombre",
          "component": "AppInput",
          "type": "text",
          "label": "Nombre del Tipo de Atención",
          "required": true,
          "maxLength": 50
        },
        {
          "name": "idTipoAtencionBase",
          "component": "AppSelect",
          "label": "Tipo Atención Base",
          "required": true,
          "dataSource": "GET /api/v1/admision/tipos-atencion/catalogo-base",
          "valueField": "id",
          "labelField": "nombre"
        },
        {
          "name": "idTipoAtencionBaseDrg",
          "component": "AppSelect",
          "label": "Tipo Atención Base DRG",
          "required": true,
          "dataSource": "GET /api/v1/admision/tipos-atencion/catalogo-base?ordenParaDrg=true",
          "valueField": "id",
          "labelField": "nombre"
        },
        {
          "name": "diasCerrarAuto",
          "component": "AppInput",
          "type": "number",
          "label": "Días para cierre automático",
          "required": false,
          "visible_condition": "indCerrarAuto === true",
          "min": 1,
          "max": 99
        }
      ],
      "submit_action": "POST /api/v1/admision/tipos-atencion",
      "update_action": "PUT /api/v1/admision/tipos-atencion/{id}"
    },
    {
      "id": "indicators",
      "type": "CHECKBOX_GROUP",
      "component_hint": "IndicatorsSection",
      "title": "Indicadores de Requerimientos",
      "collapsible": true,
      "groups": [
        {
          "group": "Requerimientos de Admisión",
          "fields": [
            { "name": "habilitado",               "component": "AppCheckbox", "label": "Registro Activo" },
            { "name": "indEgreso",                "component": "AppCheckbox", "label": "Requiere Egreso" },
            { "name": "indCama",                  "component": "AppCheckbox", "label": "Requiere Cama",
              "exclusive_with": "indCrearAtencionesActivas", "note": "RN-04: mutuamente excluyente con Activar/Crear Atenciones" },
            { "name": "indCrearAtencionesActivas","component": "AppCheckbox", "label": "Activar/Crear Atenciones",
              "exclusive_with": "indCama", "note": "RN-04: mutuamente excluyente con Requiere Cama" },
            { "name": "indResponsable",           "component": "AppCheckbox", "label": "Requiere datos de Responsable" },
            { "name": "indAcompanante",           "component": "AppCheckbox", "label": "Requiere datos de Acompañante" },
            { "name": "indIngresoVia",            "component": "AppCheckbox", "label": "Requiere Vía de Ingreso" },
            { "name": "indDiagnostico",           "component": "AppCheckbox", "label": "Requiere Diagnósticos" },
            { "name": "indSolicitud",             "component": "AppCheckbox", "label": "Requiere Solicitud" },
            { "name": "indReqConsultaFormular",   "component": "AppCheckbox", "label": "Requiere Consulta para Formular" }
          ]
        },
        {
          "group": "Indicadores Clínicos",
          "fields": [
            { "name": "indCerrarAuto",            "component": "AppCheckbox", "label": "Cerrar automáticamente después de N días",
              "triggers_field": "diasCerrarAuto", "note": "RN-05: habilita el campo Días" },
            { "name": "indIngresoCorto",          "component": "AppCheckbox", "label": "Ingreso Corto" },
            { "name": "indUrgencias",             "component": "AppCheckbox", "label": "Categorizado como Urgencias" },
            { "name": "indPaquete",               "component": "AppCheckbox", "label": "Paquete" },
            { "name": "indAtencionDomiciliaria",  "component": "AppCheckbox", "label": "Atención Domiciliaria" },
            { "name": "indEgresoAutomatico",      "component": "AppCheckbox", "label": "Egreso Automático" }
          ]
        },
        {
          "group": "Indicadores Regulatorios y Reportes",
          "fields": [
            { "name": "indMortalidad",            "component": "AppCheckbox", "label": "Indicador Mortalidad (Circular UT)" },
            { "name": "indInfeccion",             "component": "AppCheckbox", "label": "Indicador Infección (Circular UT)" },
            { "name": "indReqDatosAtencionMedica","component": "AppCheckbox", "label": "Requiere Datos de Atención Médica Externa" },
            { "name": "indPgp",                   "component": "AppCheckbox", "label": "Indicador Pago Global Prospectivo (PGP)" },
            { "name": "indAria",                  "component": "AppCheckbox", "label": "Indicador Aria" },
            { "name": "indImpresionManilla",      "component": "AppCheckbox", "label": "Impresión Manilla" },
            { "name": "indVisor",                 "component": "AppCheckbox", "label": "Activar en Visor" }
          ]
        }
      ]
    }
  ],
  "data_flow": {
    "server_component_fetches": [
      {
        "data": "catalogoTiposAtencionBase",
        "endpoint": "GET /api/v1/admision/tipos-atencion/catalogo-base",
        "cache": "revalidate:86400",
        "note": "Catálogo estable — candidato a Redis TTL 24h"
      }
    ],
    "server_actions": [
      { "name": "crearTipoAtencionAction",      "endpoint": "POST /api/v1/admision/tipos-atencion",        "type": "COMMAND" },
      { "name": "actualizarTipoAtencionAction", "endpoint": "PUT  /api/v1/admision/tipos-atencion/{id}",   "type": "COMMAND" },
      { "name": "buscarTipoAtencionAction",     "endpoint": "GET  /api/v1/admision/tipos-atencion?nombre=","type": "QUERY"   }
    ]
  },
  "validations_zod": [
    { "field": "nombre",               "schema": "z.string().min(1, 'El nombre es requerido').max(50, 'Máximo 50 caracteres')" },
    { "field": "idTipoAtencionBase",   "schema": "z.number({ required_error: 'Seleccione el tipo base' })" },
    { "field": "idTipoAtencionBaseDrg","schema": "z.number({ required_error: 'Seleccione el tipo base DRG' })" },
    { "field": "diasCerrarAuto",       "schema": "z.number().min(1).max(99).optional()" },
    {
      "field": "indCama_indCrearAtencionesActivas",
      "schema": "z.object({indCama: z.boolean(), indCrearAtencionesActivas: z.boolean()}).refine(d => !(d.indCama && d.indCrearAtencionesActivas), { message: 'RN-04: Requiere Cama y Activar/Crear Atenciones son mutuamente excluyentes', path: ['indCama'] })"
    },
    {
      "field": "indCerrarAuto_diasCerrarAuto",
      "schema": "z.object({indCerrarAuto: z.boolean(), diasCerrarAuto: z.number().optional()}).refine(d => !d.indCerrarAuto || (d.diasCerrarAuto != null && d.diasCerrarAuto > 0), { message: 'RN-05: Ingrese los días para cierre automático', path: ['diasCerrarAuto'] })"
    }
  ],
  "v0_prompt": "Create a professional catalog management page for a hospital admission system. The page manages 'Tipos de Atención' (Care Types). Layout: top search bar to find care types by name with pagination. Below, a CRUD form with: ID field (numeric, readonly when editing), Name field (text, max 50 chars), two dropdowns for 'Tipo Base' and 'Tipo Base DRG' (loaded from the same catalog endpoint). Then three collapsible sections of checkboxes organized as: 'Requerimientos de Admisión' (8 checkboxes including mutually exclusive pair: Requiere Cama / Activar Atenciones), 'Indicadores Clínicos' (6 checkboxes, including 'Cierre Automático' that conditionally shows a numeric days input), and 'Indicadores Regulatorios' (7 checkboxes). Save and Cancel buttons. Clean enterprise healthcare management style with shadcn/ui components and Tailwind CSS. The sidebar navigation is provided by the parent DashboardLayout."
}
```

### Valores del campo `_execution.status`
```
SUCCESS  → todos los controles VB6 mapeados, todos los endpoints encontrados en la ficha de trazabilidad
PARTIAL  → algún control no tiene componente equivalente claro, o algún endpoint referenciado
           no existe en la ficha de trazabilidad — se usó una estimación
FAILED   → la ficha {frmX}.md no existe o está vacía (backend no completado)
```

### Reglas para `context_forward`
```
Poblar context_forward.alerts cuando:
  - Existen controles sin componente de biblioteca → "@nextjs-generator: controles
    {lista} no tienen componente en la biblioteca. Registrar en component-requests.md."
  - page_type = WIZARD → "@nextjs-generator: formulario multi-paso con {N} pestañas.
    Usar componente Tabs de shadcn, cada pestaña como sección independiente."
  - DataGrid con > 1000 registros esperados → "@nextjs-generator: AppDataTable con
    paginación server-side. NO cargar todos los datos en el cliente."
  - Carga dependiente entre combos → "@nextjs-generator: {comboB} depende de
    {comboA}. Implementar con useEffect o Server Action condicional."
  - Exclusiones mutuas (tipo RN-04) → "@nextjs-generator: implementar lógica
    de exclusión mutua entre {campoA} y {campoB} en el componente de formulario."

Si no hay alertas → context_forward.alerts = []
```

---

## Reglas Mandatorias FCV

1. **NUNCA inventar** endpoints — solo usar los que están en la sección "Endpoints REST Generados" de `{frmX}.md`
2. **NUNCA omitir** validaciones VB6 — todas deben aparecer en `validations_zod`
3. **NUNCA omitir** reglas de negocio RN-XX — mapear como validaciones Zod refine() o como lógica UI
4. **SIEMPRE referenciar** componentes de la biblioteca (`AppInput`, `AppSelect`, `AppDataTable`, `AppCheckbox`)
5. **SIEMPRE incluir** el bloque `_execution` al inicio del JSON
6. Si status es PARTIAL →listar en `degradation_notes` qué controles o endpoints no se pudieron mapear
7. El campo `v0_prompt` debe estar **en inglés**, ser específico y mencionar shadcn/ui y Tailwind CSS
8. Para formularios con TabStrip → mapear como secciones con type `TABS`
9. Para formularios con más de 6 checkboxes consecutivos → agruparlos en `CHECKBOX_GROUP` colapsable
10. El campo `context_forward.for_agent` **SIEMPRE** apunta a `"nextjs-generator"`
11. **SOLO invocar este agente** cuando el formulario tenga status BACKEND_LISTO o COMPLETADO en `master-index.json`
12. Las rutas en `route` usan kebab-case en español: `/admision/tipos-atencion`, `/historiaclinica/eventos`

---

## Skills Activos
> Conocimiento especializado que amplía las capacidades de este agente.

```bash
# Invocación completa con skills:
@workspace #file:.github/instructions/frontend-analyzer.instructions.md \
           #file:.github/skills/frontend/nextjs-patterns.skill.md \
           #file:migration-registry/forms/{frmX}-analysis.json \
           #file:migration-registry/forms/{frmX}.md
Analizar UI del formulario VB6 {frmX} y generar el brief para Next.js.
```

### Skills incluidos:
| Skill | Ruta | Qué aporta |
|-------|------|------------|
| Next.js Patterns | `.github/skills/frontend/nextjs-patterns.skill.md` | Server Components, Server Actions, App Router |
