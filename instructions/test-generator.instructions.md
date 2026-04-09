# @test-generator — FCV
**Proyecto:** Fundación Cardiovascular — Migración Backend
**Rol:** Generar tests automatizados para el código backend, verificando paridad funcional
con la lógica original del formulario VB6. Idioma de comentarios y DisplayName: ESPAÑOL.

> Adaptado por ARC-Meta-Agent para FCV. Paquete: `com.fcv` | Framework: JUnit 5 + Mockito + @DataJpaTest
> Roles FCV: MEDICO · ENFERMERA · ADMINISTRATIVO · AUDITOR · ADMIN

---

## Responsabilidades

Generar tests que verifiquen que el nuevo código hace exactamente lo mismo que el
formulario VB6 original. Usar el análisis VB6 como fuente de verdad para los casos de prueba.

---

## Entradas Requeridas

- `migration-registry/forms/{frmX}-analysis.json` (lógica original, validaciones, reglas de negocio)
- `migration-registry/forms/{frmX}-security-report.md` (hallazgos que requieren test específico)
- Código Java generado por `@backend-generator` en `src/main/java/com/fcv/{modulo}/`
- `migration-registry/forms/{frmX}-reuse-plan.json` (decisiones REUSE/EXTEND/CREATE)

---

## Tests Backend

### Tests de CommandHandler (JUnit 5 + Mockito)
```java
// Migrado por: @test-generator | Origen VB6: frmAdmAtencion.frm | Fecha: {fecha}
@ExtendWith(MockitoExtension.class)
class AdmitirPacienteCommandHandlerTest {

    @Mock private AtencionRepository atencionRepository;
    @Mock private PacienteRepository pacienteRepository;
    @Mock private ApplicationEventPublisher eventPublisher;
    @Mock private MeterRegistry meterRegistry;

    @InjectMocks
    private AdmitirPacienteCommandHandler handler;

    // ✅ Caso exitoso — basado en cmdGuardar_Click exitoso en VB6
    @Test
    @DisplayName("Debe registrar la atención cuando los datos son válidos")
    void debeRegistrarAtencionCuandoDatosValidos() {
        // Arrange
        var paciente = new Paciente(1L, "12345678", TipoDocumento.CC, "Juan Pérez");
        var command = new AdmitirPacienteCommand("12345678", TipoDocumento.CC, 1L);

        when(pacienteRepository.findByNumeroDocumento("12345678"))
            .thenReturn(Optional.of(paciente));
        when(atencionRepository.save(any())).thenAnswer(inv -> {
            var a = (Atencion) inv.getArgument(0);
            ReflectionTestUtils.setField(a, "id", 1L);
            return a;
        });
        // Mock del Counter de Micrometer para evitar NullPointerException
        when(meterRegistry.counter(anyString(), any(String[].class)))
            .thenReturn(mock(Counter.class));

        // Act
        Long id = handler.handle(command);

        // Assert
        assertThat(id).isEqualTo(1L);
        verify(atencionRepository).save(argThat(a ->
            a.getEstado() == AtencionEstado.ACTIVA &&
            a.getPacienteId().equals(1L)
        ));
        verify(eventPublisher).publishEvent(any(PacienteAdmitidoEvent.class));
    }

    // ❌ Paciente no encontrado
    @Test
    @DisplayName("Debe lanzar EntityNotFoundException cuando el paciente no existe")
    void debeLanzarExcepcionCuandoPacienteNoExiste() {
        var command = new AdmitirPacienteCommand("99999999", TipoDocumento.CC, 1L);
        when(pacienteRepository.findByNumeroDocumento("99999999")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> handler.handle(command))
            .isInstanceOf(EntityNotFoundException.class);
    }

    // ❌ Atención activa ya existe
    @Test
    @DisplayName("Debe lanzar BusinessException cuando el paciente ya tiene atención activa")
    void debeLanzarExcepcionCuandoAtencionActivaYaExiste() {
        var paciente = new Paciente(1L, "12345678", TipoDocumento.CC, "Juan");
        when(pacienteRepository.findByNumeroDocumento(any())).thenReturn(Optional.of(paciente));
        when(atencionRepository.existsByPacienteIdAndEstado(1L, AtencionEstado.ACTIVA)).thenReturn(true);

        assertThatThrownBy(() -> handler.handle(new AdmitirPacienteCommand("12345678", TipoDocumento.CC, 1L)))
            .isInstanceOf(BusinessException.class)
            .hasMessageContaining("atención activa");
    }
}
```

### Tests de QueryHandler
```java
@ExtendWith(MockitoExtension.class)
class BuscarPacientePorDocumentoQueryHandlerTest {

    @Mock private PacienteRepository pacienteRepository;
    @InjectMocks private BuscarPacientePorDocumentoQueryHandler handler;

    // ✅ Búsqueda exitosa — basado en cmdBuscar_Click en VB6
    @Test
    @DisplayName("Debe retornar el DTO del paciente cuando el documento existe")
    void debeRetornarPacienteCuandoDocumentoExiste() { ... }

    // ❌ Paciente no encontrado
    @Test
    @DisplayName("Debe lanzar EntityNotFoundException cuando el documento no existe")
    void debeLanzarExcepcionCuandoDocumentoNoExiste() { ... }
}
```

### Tests de Repositorio (@DataJpaTest)
```java
// Migrado por: @test-generator | Origen VB6: frmAdmAtencion.frm
@DataJpaTest
@Sql("/test-data/admision.sql")
@ActiveProfiles("dev")
class AtencionRepositoryTest {

    @Autowired private AtencionRepository repository;
    @Autowired private TestEntityManager entityManager;

    @Test
    @DisplayName("findByPacienteIdAndEstado debe retornar atención activa del paciente")
    void findByPacienteIdAndEstado_debeRetornarAtencionActiva() {
        var result = repository.findByPacienteIdAndEstado(1L, AtencionEstado.ACTIVA);
        assertThat(result).isPresent();
        assertThat(result.get().getEstado()).isEqualTo(AtencionEstado.ACTIVA);
    }

    @Test
    @DisplayName("findByPacienteIdAndEstado debe retornar vacío cuando no hay atención activa")
    void findByPacienteIdAndEstado_debeRetornarVacioSinAtenciones() {
        var result = repository.findByPacienteIdAndEstado(999L, AtencionEstado.ACTIVA);
        assertThat(result).isEmpty();
    }
}
```

### Tests de Controlador (@WebMvcTest)
```java
@WebMvcTest(AtencionCommandController.class)
class AtencionCommandControllerTest {

    @Autowired private MockMvc mockMvc;
    @MockBean private AdmitirPacienteCommandHandler admitirHandler;

    @Test
    @WithMockUser(roles = "ADMINISTRATIVO")
    @DisplayName("POST /api/v1/admision/atenciones debe retornar 201 con datos válidos")
    void debeRetornar201ConDatosValidos() throws Exception {
        when(admitirHandler.handle(any())).thenReturn(1L);

        mockMvc.perform(post("/api/v1/admision/atenciones")
            .contentType(MediaType.APPLICATION_JSON)
            .content """
                {"numeroDocumento": "12345678", "tipoDocumento": "CC", "epsId": 1}
            """)
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$").value(1));
    }

    @Test
    @DisplayName("POST /api/v1/admision/atenciones debe retornar 401 sin autenticación")
    void debeRetornar401SinAutenticacion() throws Exception {
        mockMvc.perform(post("/api/v1/admision/atenciones")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{}"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "MEDICO")
    @DisplayName("POST /api/v1/admision/atenciones debe retornar 403 con rol sin permisos de escritura")
    void debeRetornar403ConRolSinPermisosEscritura() throws Exception {
        mockMvc.perform(post("/api/v1/admision/atenciones")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{}"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMINISTRATIVO")
    @DisplayName("POST /api/v1/admision/atenciones debe retornar 400 con datos inválidos")
    void debeRetornar400ConDatosInvalidos() throws Exception {
        mockMvc.perform(post("/api/v1/admision/atenciones")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{}"))
            .andExpect(status().isBadRequest());
    }
}
```

### Test de AuditoriaAcceso (Resolución 866/2021)
```java
// Migrado por: @test-generator | Origen VB6: frmHCEvento.frm
// Patrón obligatorio para cualquier formulario que opera sobre historiaclinica
@ExtendWith(MockitoExtension.class)
class ConsultarHistoriaClinicaQueryHandlerTest {

    @Mock private HistoriaClinicaRepository hcRepository;
    @Mock private AuditoriaAccesoService auditoriaService;
    @InjectMocks private ConsultarHistoriaClinicaQueryHandler handler;

    @Test
    @DisplayName("Debe registrar acceso en AuditoriaAcceso antes de retornar datos de HC")
    void debeRegistrarAuditoriaAntesDeRetornarHC() {
        // Arrange
        var query = new ConsultarHistoriaClinicaQuery(1L, "user-medico");
        when(hcRepository.findByPacienteId(1L)).thenReturn(Optional.of(new HistoriaClinica()));

        // Act
        handler.handle(query);

        // Assert: AuditoriaAcceso DEBE ser llamado — Resolución 866/2021
        verify(auditoriaService).registrar(argThat(a ->
            a.getPacienteId().equals(1L) &&
            a.getOperacion().equals(TipoOperacion.LECTURA_HC)
        ));
    }

    @Test
    @DisplayName("Debe registrar intento de acceso fallido en AuditoriaAcceso")
    void debeRegistrarAccesoFallidoEnAuditoria() {
        var query = new ConsultarHistoriaClinicaQuery(999L, "user-medico");
        when(hcRepository.findByPacienteId(999L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> handler.handle(query))
            .isInstanceOf(EntityNotFoundException.class);

        // Incluso en caso de fallo, debe registrarse el intento de acceso
        verify(auditoriaService).registrar(any());
    }
}
```

---

## Datos de Prueba

Generar `src/test/resources/test-data/{frmX}.sql` con datos representativos:
```sql
-- Datos de prueba para {frmX}
-- Generado por @test-generator | Basado en casos de prueba VB6
-- ⚠️ NUNCA usar datos reales de pacientes (Ley 1581/2012)
-- Los tests de repositorio (@DataJpaTest) usan perfil 'dev' con H2 en memoria
INSERT INTO TiposDocumento (TipoDocId, Codigo, Descripcion, Activo) VALUES (1, 'CC', 'Cédula de Ciudadanía', 1);
INSERT INTO Paciente (PacienteId, NroDocumento, TipoDocId, Nombre, Estado) VALUES (1, '12345678', 1, 'Juan Prueba', 'ACTIVO');
INSERT INTO Atencion (AtencionId, PacienteId, Estado, FechaIngreso) VALUES (1, 1, 'ACTIVA', CURRENT_TIMESTAMP);
```

> **Nota de compatibilidad H2:** Usar `CURRENT_TIMESTAMP` en lugar de `GETDATE()` en los scripts
> de test-data para que sean compatibles con H2 en modo MSSQL (perfil dev).

---

## Reglas Mandatorias FCV

1. **SIEMPRE** usar `@DisplayName` en ESPAÑOL describiendo el comportamiento esperado
2. **SIEMPRE** incluir el comentario de trazabilidad: `// Migrado por: @test-generator | Origen VB6: {frmX}.frm | Fecha: {fecha}`
3. Cobertura mínima objetivo: **80% en capa de aplicación** (handlers)
4. **Para REUSE:** no generar tests del servicio reutilizado (ya tiene los propios).
   Solo testear que el nuevo código lo llama correctamente (verify())
5. **Para EXTEND:** agregar tests del nuevo método al archivo de test del repositorio existente
6. Separar tests unitarios de integración con `@Tag("unit")` y `@Tag("integration")`
7. Roles válidos en `@WithMockUser`: `MEDICO`, `ENFERMERA`, `ADMINISTRATIVO`, `AUDITOR`, `ADMIN`
8. **Para formularios de historiaclinica:** SIEMPRE generar test del patrón `AuditoriaAcceso`
   (ver sección "Test de AuditoriaAcceso") — Resolución 866/2021
9. **NUNCA** usar datos reales de pacientes en `test-data/*.sql` (Ley 1581/2012)
10. **SIEMPRE** mockear `MeterRegistry` en tests de CommandHandlers — evitar `NullPointerException`
11. **SIEMPRE** anotar tests de repositorio con `@ActiveProfiles("dev")` para usar H2 en memoria
12. En scripts `test-data/*.sql` usar `CURRENT_TIMESTAMP` en lugar de `GETDATE()` — compatibilidad H2
13. **Las rutas en `@WebMvcTest`** deben coincidir exactamente con el `@RequestMapping` del controller
    — verificar que el `context-path` del `application.yml` es `/` antes de escribir las rutas

---

## Skills Activos

```bash
@workspace #file:.github/instructions/test-generator.instructions.md \
           #file:.github/skills/quality/test-patterns.skill.md \
           Generar tests para {frmX} — adjuntar {frmX}-analysis.json y archivos Java generados
```

| Skill | Ruta | Qué aporta |
|-------|------|------------|
| Test Patterns | `.github/skills/quality/test-patterns.skill.md` | Patrones JUnit 5, Mockito, @DataJpaTest |
