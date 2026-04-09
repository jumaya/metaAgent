# Skills — Biblioteca de Capacidades Reutilizables

Carpeta de skills para el ecosistema ARC-Vision + ARC-Meta-Agent.
Cada skill es un fragmento de conocimiento especializado que los agentes
invocan con `#file:` para ampliar su capacidad sin duplicar instrucciones.

---

## Integración Automática con el Ecosistema

> ✅ **Los skills están integrados en el flujo de generación de agentes.**
> ARC-Meta-Agent los mapea automáticamente a cada agente generado usando la
> tabla de Skills Library definida en su lógica interna.
>
> El campo `agents.skills_path` en el `architecture-context.json` activa este
> comportamiento — ARC-Vision lo captura en el Bloque [K] de la conversación.

**Flujo completo:**
```
ARC-Vision (Bloque K)
  → detecta .github/skills/ → registra agents.skills_path en architecture-context.json
      ↓
ARC-Meta-Agent (PASO 5 - Skills Library)
  → lee agents.skills_path → mapea skills a cada agente según tipo
      ↓
Agentes generados
  → incluyen sección "## Skills Activos" con #file: exactos para invocar
      ↓
Desarrollador
  → copia el comando de invocación completo con skills ya incluidos
```

---

## Estructura

```
.github/skills/
  backend/
    jpa-patterns.skill.md           ← Patrones JPA/Hibernate avanzados
    ddd-patterns.skill.md           ← Patrones DDD: Agregados, Value Objects, Eventos
    cqrs-patterns.skill.md          ← Patrones CQRS: Commands, Queries, Handlers
    spring-security.skill.md        ← Configuración Spring Security + JWT
    exception-handling.skill.md     ← Manejo global de excepciones
  frontend/
    nextjs-patterns.skill.md        ← Server Components, Server Actions, Streaming
    form-patterns.skill.md          ← React Hook Form + Zod + validación
    table-patterns.skill.md         ← TanStack Table, paginación, filtros
  data/
    sql-to-jpql.skill.md            ← Transformación SQL Server → JPQL/JPA
    migration-scripts.skill.md      ← Patrones Flyway/Liquibase
  quality/
    test-patterns.skill.md          ← JUnit 5, Mockito, TestContainers
    security-checklist.skill.md     ← OWASP checklist por capa
  infra/
    docker-patterns.skill.md        ← Dockerfiles y Compose optimizados
    cicd-patterns.skill.md          ← Pipelines Jenkins / GitHub Actions
  architecture/
    adr-patterns.skill.md           ← Cómo escribir ADRs de calidad
    domain-modeling.skill.md        ← Técnicas de modelado de dominio
```

---

## Mapeo Skills → Agentes

ARC-Meta-Agent aplica este mapeo automáticamente al generar cada agente:

| Agente Generado | Skills Asignados |
|----------------|-----------------|
| `backend-generator` (DDD_CQRS) | `ddd-patterns` · `cqrs-patterns` · `jpa-patterns` · `exception-handling` |
| `backend-generator` (LAYERED) | `jpa-patterns` · `exception-handling` |
| `entity-modeler` | `ddd-patterns` · `jpa-patterns` |
| `sql-legacy-analyzer` | `sql-to-jpql` |
| `sp-migration-planner` | `sql-to-jpql` · `jpa-patterns` |
| `nextjs-generator` | `nextjs-patterns` |
| `component-library-agent` | `nextjs-patterns` |
| `security-reviewer` | `security-checklist` · `spring-security` |
| `test-generator` | `test-patterns` |
| `docker-agent` | `docker-patterns` |
| `cicd-agent` | `docker-patterns` |
| `domain-discovery` | `ddd-patterns` · `cqrs-patterns` |
| `arc-vision` (ADRs) | `adr-patterns` |

---

## Cómo Usar un Skill Manualmente

Si necesitas un skill fuera del flujo de agentes generados:

```bash
# Invocar un skill junto a un agente
@workspace #file:.github/instructions/backend-generator.instructions.md
           #file:.github/skills/backend/ddd-patterns.skill.md
Genera la entidad Paciente para frmAdmision.

# Invocar múltiples skills
@workspace #file:.github/instructions/backend-generator.instructions.md
           #file:.github/skills/backend/jpa-patterns.skill.md
           #file:.github/skills/data/sql-to-jpql.skill.md
Genera el repositorio para Admisiones con estos SPs complejos: [...]
```

---

## Agregar un Skill Nuevo

1. Crear el archivo en la subcarpeta correcta: `{area}/{nombre}.skill.md`
2. Seguir la convención de nomenclatura: kebab-case, descriptivo
3. Actualizar la tabla "Mapeo Skills → Agentes" en este README
4. Si el skill aplica a un tipo de agente que ARC-Meta-Agent genera,
   notificarlo en el `agent-plan.md` del proyecto para que se incluya

---

## Convención de Nomenclatura

`{area}/{nombre-del-skill}.skill.md`

- **área**: backend | frontend | data | quality | infra | architecture
- **nombre**: kebab-case, descriptivo de lo que enseña
- **extensión**: `.skill.md` (diferencia de `.instructions.md` y `.agent.md`)
