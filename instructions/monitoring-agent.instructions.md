# @monitoring-agent — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend  
**Rol:** Configurar Prometheus + Grafana y métricas Micrometer en Spring Boot
para observabilidad completa del sistema migrado.

> Adaptado por ARC-Meta-Agent para FCV. On-premise: Jenkins + Prometheus + Grafana ya instalados
> Paquete: `com.fcv` | 5 bounded contexts hospitalarios

---

## Responsabilidades

Configurar el stack de monitoreo completo: instrumentar el backend con Micrometer,
configurar Prometheus para recolectar métricas, y generar dashboards Grafana
con métricas técnicas (JVM, HTTP) y **métricas de negocio hospitalario** (por bounded context).

**Métricas de negocio obligatorias para FCV:**
- `fcv.admision.atenciones.registradas` — admisiones por día
- `fcv.admision.citas.agendadas` — citas agendadas
- `fcv.historiaclinica.accesos` — accesos a HC por tipo de usuario (Res. 866/2021)
- `fcv.cartera.facturas.emitidas` — facturas generadas
- `fcv.business.commands.executed{modulo}` — counter por módulo

---

## Phase 1 — Backend Instrumentation (once per project)

### Dependencies to add in `pom.xml`
```xml
<!-- Micrometer + Actuator + Prometheus -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Configuration in `application.yml`
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized   # do not expose details without auth
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:local}
    distribution:
      percentiles-histogram:
        http.server.requests: true    # histogram for p95/p99 latency
      percentiles:
        http.server.requests: [0.5, 0.95, 0.99]
```

### Configuration class `MonitoringConfig.java`
```java
@Configuration
public class MonitoringConfig {

    // Global operation counter per bounded context
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "migration-system");
    }
}
```

---

## Phase 2 — Business Metrics per Bounded Context

For each migrated bounded context, add metrics in the CommandHandlers:

```java
// Patrón FCV — agregar a TODO CommandHandler de FCV
@Service
@RequiredArgsConstructor
public class AdmitirPacienteCommandHandler {

    private final AtencionRepository atencionRepository;
    private final MeterRegistry meterRegistry;      // ← inyectar siempre

    @Transactional
    public Long handle(AdmitirPacienteCommand command) {
        // ... lógica del handler ...

        // Métrica de negocio — contador de comandos ejecutados
        meterRegistry.counter("fcv.business.commands.executed",
            "modulo", "admision",
            "command", "AdmitirPaciente",
            "status", "success"
        ).increment();

        // Métrica específica del módulo admision
        meterRegistry.counter("fcv.admision.atenciones.registradas",
            "tipo", command.getTipoAtencion()
        ).increment();

        // Timer de operación
        return meterRegistry.timer("fcv.business.command.duration",
            "modulo", "admision",
            "command", "AdmitirPaciente"
        ).record(() -> {
            atencionRepository.save(atencion);
            return atencion.getId();
        });
    }
}

// En caso de error, registrar con status=error:
meterRegistry.counter("fcv.business.commands.executed",
    "modulo", "admision",
    "command", "AdmitirPaciente",
    "status", "error",
    "error_type", e.getClass().getSimpleName()
).increment();

// Patrón especial para Historia Clínica (Resolución 866/2021):
// SIEMPRE registrar acceso ANTES de retornar datos
meterRegistry.counter("fcv.historiaclinica.accesos",
    "tipo_usuario", SecurityContextHolder.getContext().getAuthentication().getName(),
    "modulo", "historiaclinica"
).increment();
```

---

## Phase 3 — Local Monitoring Stack

### Generate `monitoring/docker-compose.monitoring.yml`
```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123   # change in production
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards

volumes:
  prometheus_data:
  grafana_data:
```

### Generate `monitoring/prometheus/prometheus.yml`
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-backend'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']   # local backend
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'backend'

  - job_name: 'prometheus-self'
    static_configs:
      - targets: ['localhost:9090']
```

### Generate `monitoring/grafana/provisioning/datasources/prometheus.yml`
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    access: proxy
```

---

## Phase 4 — Grafana Dashboards

### Dashboard 1: `backend-overview.json` — System Overview
```
Panels:
├── Row: HTTP Traffic (RED method)
│   ├── Requests/second per endpoint (rate)
│   ├── 4xx and 5xx error rate (%)
│   └── p50, p95, p99 latency (ms)
├── Row: JVM Health
│   ├── Heap used vs maximum (gauge + sparkline)
│   ├── Non-heap memory
│   ├── GC pause time (ms)
│   └── Active threads
└── Row: Database (HikariCP)
    ├── Active connections vs pool maximum
    ├── Connection wait time
    └── Slow queries (>500ms)
```

### Dashboard 2: `business-metrics.json` — Métricas de Negocio FCV
```
Panels (uno por bounded context migrado):
├── Row: Admisión
│   ├── Atenciones registradas por hora (fcv.admision.atenciones.registradas)
│   ├── Citas agendadas vs canceladas
│   └── Tasa de error en admisiones (%)
├── Row: Historia Clínica (Resolución 866/2021)
│   ├── Accesos a HC por tipo de usuario (MEDICO/ENFERMERA/AUDITOR)
│   ├── Accesos anómalos > 100/min (gauge con alerta)
│   └── AuditoriaAcceso registros por hora
├── Row: Cartera
│   ├── Facturas emitidas por hora
│   └── Tasa de error en facturación
└── Top 10 endpoints más lentos (tabla)

Consultas Prometheus para FCV:
  # Comandos por módulo en los últimos 5 minutos
  rate(fcv_business_commands_executed_total[5m])

  # Tasa de error por módulo
  rate(fcv_business_commands_executed_total{status="error"}[5m])
  / rate(fcv_business_commands_executed_total[5m])

  # Latencia p95 de requests HTTP
  histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))

  # Accesos a Historia Clínica (auditoría Res. 866/2021)
  rate(fcv_historiaclinica_accesos_total[5m])
```

### Dashboard 3: `migration-progress.json` — Migration Progress
```
Project status panels (updated manually or via API):
├── Migrated forms vs total (gauge: {completed}/{total})
├── Forms by status (pie chart: PENDING/IN PROGRESS/COMPLETED/ERROR)
├── Backend services created per domain (bar chart)
└── Test coverage per bounded context (from JaCoCo)
```

---

## Phase 5 — Basic Alerts

### Generate `monitoring/prometheus/alerts.yml`
```yaml
groups:
  - name: backend-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High 5xx error rate on backend"
          description: "More than 10% of requests are failing"

      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Alta latencia en backend FCV (p95 > 800ms)"
          description: "El tiempo de respuesta p95 supera el SLA de 800ms"

      - alert: HistoriaClinicaAccesosAnomalos
        expr: rate(fcv_historiaclinica_accesos_total[1m]) > 100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Accesos anómalos a Historia Clínica (Res. 866/2021)"
          description: "Más de 100 accesos/min a HC — revisar AuditoriaAcceso"

      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool nearly exhausted (>90%)"
```

---

## Reglas Mandatorias FCV

1. El prefijo de **todas** las métricas de negocio FCV es `fcv.` (ej: `fcv.admision.atenciones.registradas`)
2. **SIEMPRE** incluir el tag `modulo` con uno de: `admision`, `historiaclinica`, `cartera`, `configuracion`, `identidad`
3. **NUNCA** exponer `/actuator/prometheus` sin autenticación en producción
4. Las contraseñas de Grafana en `docker-compose.monitoring.yml` son solo para desarrollo local — usar secretos en producción
5. **SIEMPRE** incluir `monitoring/README.md` con instrucciones para levantar el stack
6. Los dashboards Grafana deben exportarse como JSON para versionar en Git
7. Por cada bounded context migrado → agregar su panel en `business-metrics.json`
8. **CRÍTICO (Res. 866/2021):** El panel de accesos a Historia Clínica es OBLIGATORIO y tiene alerta configurada. Nunca eliminarlo.
9. Los umbrales de alerta FCV son: latencia p95 > 800ms (warning), tasa de error > 5% (critical), HC accesos > 100/min (critical)
10. La métrica `fcv.historiaclinica.accesos` se registra en Micrometer Y en `AuditoriaAcceso` por separado — son complementarias, no excluyentes

---

## Skills Activos
> Conocimiento especializado que amplía las capacidades de este agente.

```bash
# Invocación completa con skills:
@workspace #file:.github/instructions/monitoring-agent.instructions.md \
           #file:.github/skills/infra/docker-patterns.skill.md \
           [descripción de la tarea de monitoreo]
```

### Skills incluidos:
| Skill | Ruta | Qué aporta |
|-------|------|------------|
| Docker Patterns | `.github/skills/infra/docker-patterns.skill.md` | docker-compose para stack Prometheus+Grafana on-premise |
