# Instrucciones Globales — Migración Backend FCV
## Fundación Cardiovascular — Sistema Hospitalario
**IMPORTANTE: Este archivo está personalizado para el proyecto FCV. No es genérico.**
**Generado por:** ARC-Meta-Agent | **Basado en:** architecture-context.json | **Fecha:** 2026-03-12

## Contexto del Proyecto

Migración completa del sistema legacy **Visual Basic 6 + SQL Server** de la Fundación Cardiovascular (FCV)
a **Java 21 + Spring Boot 3.x**. El sistema gestiona procesos clínicos y administrativos hospitalarios:
admisiones, historia clínica electrónica, facturación, citas médicas y servicios clínicos.

**Tipo:** MIGRACIÓN incremental (Strangler Fig) — 117 formularios VB6 → APIs REST  
**Alcance de esta fase:** Solo backend. Frontend (Next.js) y reportes son fases posteriores.  
**Criticidad:** ALTA — sistema hospitalario en producción activa. El VB6 sigue operando en paralelo.

## Stack Tecnológico Exacto

### Backend
- **Lenguaje:** Java 21 (LTS)
- **Framework:** Spring Boot 3.x
- **Paquete base:** `com.fcv`
  - Estructura: `com.fcv.{modulo}.{capa}`
  - Módulos: `admision` · `historiaclinica` · `cartera` · `configuracion` · `identidad`
  - Ejemplo: `com.fcv.admision.application.commands`, `com.fcv.historiaclinica.domain.model`
- **ORM:** JPA / Hibernate (Spring Data JPA)
- **Arquitectura:** Monolito Modular + DDD + CQRS
- **Seguridad:** Spring Security + JWT standalone (sin AD ni Keycloak)
- **Caché:** Redis (catálogos: CUPs, CIEs, EPS, tarifas, municipios — TTL configurado)
- **Documentación API:** SpringDoc OpenAPI 3
- **Tests:** JUnit 5 + Mockito + @DataJpaTest + @SpringBootTest
- **Build:** Maven

### Infraestructura
- **CI/CD:** Jenkins on-premise (ya instalado) — Jenkinsfiles declarativos en Groovy
- **Monitoreo:** Micrometer → Prometheus → Grafana (on-premise)
- **Contenedores:** Docker

### Base de Datos
- **Motor:** SQL Server (compartida con VB6 durante toda la transición)
- **⚠️ REGLA CRÍTICA:** Solo DDL ADITIVO durante la transición
  - ✅ PERMITIDO: `CREATE TABLE`, `ALTER TABLE ADD COLUMN` (con DEFAULT o nullable)
  - ❌ PROHIBIDO: `DROP TABLE`, `ALTER TABLE DROP COLUMN`, `ALTER TABLE ALTER COLUMN`
  - Toda DDL requiere aprobación del Tech Lead antes de ejecutarse en producción

## Bounded Contexts FCV

| Módulo | Paquete | Entidades Principales |
|--------|---------|----------------------|
| Admisión | `com.fcv.admision` | Paciente, Atencion, Cita, CamaMovimiento, Remision |
| Historia Clínica | `com.fcv.historiaclinica` | HistoriaClinica, EventoClinico, Certificado, Autorizacion |
| Cartera | `com.fcv.cartera` | Factura, Cobro, CarteraMovimiento |
| Configuración | `com.fcv.configuracion` | CatalogoCup, CatalogoIcd, Tarifa, Eps, CentroExcelencia, Municipio |
| Identidad | `com.fcv.identidad` | Usuario, Rol, AuditoriaAcceso |

## Convenciones Mandatorias

### Idioma del Código
- **Comentarios y Javadoc:** ESPAÑOL
- **Variable y métodos:** inglés técnico (convención estándar Java)
- **Entidades de dominio clínico:** español cuando corresponda (ej: `Atencion`, `PacienteId`, `NumeroCuenta`)

### Nomenclatura Java — Ejemplos Reales FCV
```
Commands:      com.fcv.admision.application.commands.AdmitirPacienteCommand
Queries:       com.fcv.admision.application.queries.BuscarPacienteQuery
Handlers:      com.fcv.admision.application.commands.AdmitirPacienteCommandHandler
Entities:      com.fcv.admision.domain.model.Atencion
DTOs:          com.fcv.admision.application.dto.AtencionRequestDTO       ← todos en application
               com.fcv.admision.application.dto.AtencionResponseDTO      ← todos en application
               com.fcv.admision.application.dto.AtencionResumenDTO       ← todos en application
Repos:         com.fcv.admision.domain.repository.AtencionRepository     ← retorna entidades, nunca DTOs
JPA:           com.fcv.admision.infrastructure.persistence.JpaAtencionRepository  ← solo si hay lógica custom
Controllers:   com.fcv.admision.presentation.controllers.AtencionCommandController
```

### Trazabilidad (OBLIGATORIO en todo código generado)
```java
// Migrado por: @{nombre-agente}
// Origen VB6: {nombre-formulario}.frm
// SQL Ref: {sql_id}
// Fecha: {fecha}
```

### Roles de Seguridad FCV
`MEDICO` · `ENFERMERA` · `ADMINISTRATIVO` · `AUDITOR` · `ADMIN`

### Cumplimiento Regulatorio (OBLIGATORIO)
- **Ley 1581/2012:** Protección de datos personales — datos de pacientes son datos sensibles de salud
- **Resolución 866/2021:** Toda lectura/escritura de historia clínica requiere registro en `AuditoriaAcceso`
  - Los registros de auditoría son INMUTABLES — no se pueden modificar ni eliminar
  - Retención mínima: 20 años

### Stored Procedures — Estrategia A/B/C (ADR-004)
```
A: Migrar a dominio Java (lógica de negocio → CommandHandler/DomainService)
B: Wrapper temporal @Deprecated + ticket de deuda (máximo 20% de SPs por módulo)
C: Solo reportes — NO migrar en esta fase
```

## Archivos de Registro de Estado de Migración
- `migration-registry/master-index.json` — estado de 117 formularios
- `migration-registry/services-catalog.json` — catálogo de servicios creados
- `migration-registry/domains/*.md` — bounded contexts definidos
- `migration-registry/forms/*.md` — ficha de trazabilidad por formulario

## Estados del Registro Maestro
```
PENDIENTE → ANALIZANDO → BACKEND_LISTO → REVISADO → COMPLETADO
                                                   → ERROR
                                                   → BLOQUEADO
```

## Reglas de Oro FCV

1. **NUNCA** generar código sin consultar primero `services-catalog.json` (puede ya existir)
2. **NUNCA** hardcodear secrets, credenciales ni URLs de entorno
3. **NUNCA** poner lógica de negocio en controllers ni DTOs
4. **NUNCA** usar SQL nativo (`nativeQuery=true`) — siempre JPQL o Spring Data derived methods
5. **NUNCA** usar funciones T-SQL en JPQL: `GETDATE()`, `ISNULL()`, `TOP n`, `DATEDIFF()`
6. **NUNCA** modificar ni eliminar columnas/tablas existentes de SQL Server sin aprobación del Tech Lead
7. **SIEMPRE** incluir comentario de trazabilidad en todo código generado
8. **SIEMPRE** separar Commands (escritura) de Queries (lectura) — patrón CQRS
9. **SIEMPRE** registrar en `AuditoriaAcceso` toda operación sobre historia clínica
10. **SIEMPRE** publicar Domain Event al completar un Command exitosamente
