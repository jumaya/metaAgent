# Skill: ADR — How to Write Quality Architecture Decision Records
**Used by:** `@arc-vision` | **Layer:** Architecture

---

## What is an ADR and when to create one?

An ADR documents **a significant architectural decision**: its context, the alternatives
considered, and the consequences. Create one when:

- A technology, framework or pattern is chosen over alternatives
- A convention is defined that the whole team must follow
- A decision is made with important trade-offs
- An option is discarded that someone might propose again later

**Do NOT create an ADR for:** obvious decisions, low-impact reversible decisions,
implementation details that do not affect the architecture.

---

## ADR Template

```markdown
# ADR-{NNN}: {Title in present tense, action verb}
> Example: "Use Spring Boot with DDD+CQRS as backend architecture"
> NOT: "We decided to use Spring Boot" (past) nor "Use Spring Boot?" (question)

**Date:** {YYYY-MM-DD}
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-{NNN}
**Deciders:** {names or roles}
**Consulted:** {those who had input but did not decide}

---

## Context

What situation, problem or requirement led to this decision?
Be specific: mention the project, the team, the real constraints.

Example:
> The system has 280 VB6 forms with business logic embedded in the UI.
> The team of 8 developers knows Java but has no DDD experience.
> Parallel operation with the legacy system is required for 18 months.

---

## Decision

What did we decide to do? One clear and direct sentence.

Example:
> We adopt DDD + CQRS with Spring Boot 3.2, organizing code into bounded contexts
> by business domain, with Commands for writes and Queries for reads.

---

## Alternatives Considered

| Alternative | Brief description | Pros | Cons | Reason discarded |
|-------------|------------------|------|------|-----------------|
| Traditional layered architecture | Controller → Service → Repository | Simple, well-known, fast to implement | Does not scale well with 280 forms, rules mixed in Services | Risk of Big Ball of Mud in 6 months |
| Microservices | One service per domain from the start | Independent deployment | High operational complexity, team has no DevOps experience | Premature for team size |
| CQRS with Event Sourcing | Events as source of truth | Full auditability | Very steep learning curve, unnecessary complexity | No team experience, tight deadline |

---

## Consequences

### ✅ Positive
- Business logic lives in entities → code reflects the domain
- CQRS allows optimizing reads independently from writes
- Bounded contexts define clear boundaries for assigning work to the team
- Domain events facilitate future integration with other systems

### ⚠️ Negative / Trade-offs
- Learning curve for the team: 2-3 weeks of DDD onboarding
- More files per feature (Command, Handler, Repository, DTO) vs. traditional layer
- CQRS overhead is unnecessary for simple read-only forms

### 🚧 Risks and Mitigations
| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Team does not adopt DDD correctly | Medium | High | Pair programming sessions + strict code review for the first 4 weeks |
| Over-granularity of bounded contexts | Low | Medium | Review boundaries at monthly sprint review |

---

## Implementation Notes

Tactical decisions derived from this ADR:
- Base package: `com.{company}.{domain}.{layer}`
- One JPA repository per Aggregate Root (not per entity)
- Events published with `ApplicationEventPublisher` (Spring) within the same transaction
- Reference: `.github/skills/backend/ddd-patterns.skill.md`
```

---

## Numbering and Naming

```
ADR-001-backend-stack-spring-boot-ddd-cqrs.md
ADR-002-database-postgresql-sqlserver.md
ADR-003-authentication-jwt-spring-security.md
ADR-004-frontend-nextjs-server-components.md
ADR-005-migration-strangler-fig-18-months.md
ADR-006-shared-db-during-transition.md

Format: ADR-{NNN}-{kebab-case-title}.md
```

---

## ADR Statuses

```
Proposed    → The decision is under discussion, not yet implemented
Accepted    → The decision was made and is in effect
Deprecated  → No longer applicable (context changed) but kept as history
Superseded by ADR-{NNN} → Was replaced by a more recent decision
```

---

## ADR Checklist — Before Creating

```
□ Is the title an affirmative statement in present tense (not a question, not past tense)?
□ Does the context mention real project constraints (not generic ones)?
□ Were at least 2 alternatives evaluated with pros/cons?
□ Are the negative consequences honest (not only positive)?
□ Do risks have a proposed mitigation?
□ Is the ADR self-contained (understandable without reading other documents)?
□ Is the initial status "Proposed" (not "Accepted" without review)?
