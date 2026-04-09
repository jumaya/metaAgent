# Skill: Test Patterns — JUnit 5 + Mockito + TestContainers
**Used by:** `@test-generator` | **Layer:** Quality

---

## Test Structure by Layer

```
src/test/java/{package_base}/{domain}/
  domain/
    {Entity}Test.java                   ← Domain logic tests (no Spring)
  application/
    commands/
      {Action}{Entity}CommandHandlerTest.java  ← Handler tests with Mockito
    queries/
      {Find}{Entity}QueryHandlerTest.java
  infrastructure/
    persistence/
      {Entity}RepositoryIT.java          ← Integration test with TestContainers
    rest/
      {Entity}CommandControllerIT.java   ← Integration test with MockMvc
```

---

## Domain Entity Test (no Spring)

```java
// The fastest and most valuable tests — test pure business logic
// No @SpringBootTest, no Mockito, no DB — pure Java only
@Tag("unit")
class AdmisionTest {

    @Test
    @DisplayName("create admission sets status to ACTIVA and sets admission date")
    void crear_shouldSetStatusActivaAndDate() {
        // GIVEN
        Paciente paciente = PacienteTestFactory.crearPaciente();
        Medico medico = MedicoTestFactory.crearMedico();
        LocalDate today = LocalDate.now();

        // WHEN
        Admision admision = Admision.crear(paciente, medico, today);

        // THEN
        assertThat(admision.getEstado()).isEqualTo(EstadoAdmision.ACTIVA);
        assertThat(admision.getFechaIngreso()).isEqualTo(today);
        assertThat(admision.getMedicoTratante()).isEqualTo(medico);
    }

    @Test
    @DisplayName("closing an active admission with a reason should change status to CERRADA")
    void cerrar_activeAdmission_shouldChangeStatus() {
        // GIVEN
        Admision admision = AdmisionTestFactory.crearAdmisionActiva();

        // WHEN
        admision.cerrar("Medical discharge");

        // THEN
        assertThat(admision.getEstado()).isEqualTo(EstadoAdmision.CERRADA);
        assertThat(admision.getMotivoCierre()).isEqualTo("Medical discharge");
        assertThat(admision.getFechaCierre()).isNotNull();
    }

    @Test
    @DisplayName("closing an already closed admission should throw InvalidStateException")
    void cerrar_closedAdmission_shouldThrowException() {
        // GIVEN
        Admision admision = AdmisionTestFactory.crearAdmisionCerrada();

        // WHEN / THEN
        assertThatThrownBy(() -> admision.cerrar("Another reason"))
            .isInstanceOf(InvalidStateException.class)
            .hasMessageContaining("already been closed");
    }

    @Test
    @DisplayName("adding a diagnosis to a closed admission should throw exception")
    void agregarDiagnostico_closedAdmission_shouldThrowException() {
        Admision admision = AdmisionTestFactory.crearAdmisionCerrada();

        assertThatThrownBy(() -> admision.agregarDiagnostico("J18.0", "Pneumonia"))
            .isInstanceOf(DomainException.class);
    }
}
```

---

## CommandHandler Test (Mockito)

```java
@Tag("unit")
@ExtendWith(MockitoExtension.class)
class CrearAdmisionCommandHandlerTest {

    @Mock private AdmisionRepository admisionRepository;
    @Mock private PacienteRepository pacienteRepository;
    @Mock private MedicoRepository medicoRepository;
    @Mock private ApplicationEventPublisher eventPublisher;

    @InjectMocks
    private CrearAdmisionCommandHandler handler;

    @Test
    @DisplayName("handle should create admission and return the generated ID")
    void handle_validCommand_shouldCreateAdmisionAndReturnId() {
        // GIVEN
        CrearAdmisionCommand command = new CrearAdmisionCommand(
            1L, "1234567890", 2L, 3L, LocalDate.now(), null
        );
        Paciente paciente = PacienteTestFactory.crearPaciente(1L);
        Medico medico = MedicoTestFactory.crearMedico(3L);
        Admision savedAdmision = AdmisionTestFactory.crearAdmisionActiva(99L);

        given(admisionRepository.existsByPacienteYFecha(1L, LocalDate.now())).willReturn(false);
        given(pacienteRepository.findById(1L)).willReturn(Optional.of(paciente));
        given(medicoRepository.findById(3L)).willReturn(Optional.of(medico));
        given(admisionRepository.save(any(Admision.class))).willReturn(savedAdmision);

        // WHEN
        Long resultId = handler.handle(command);

        // THEN
        assertThat(resultId).isEqualTo(99L);
        verify(admisionRepository).save(any(Admision.class));
        verify(eventPublisher).publishEvent(any(AdmisionCreadaEvent.class));
    }

    @Test
    @DisplayName("handle should throw BusinessException if patient already has an active admission")
    void handle_patientWithActiveAdmission_shouldThrowBusinessException() {
        // GIVEN
        CrearAdmisionCommand command = new CrearAdmisionCommand(
            1L, "1234567890", 2L, 3L, LocalDate.now(), null
        );
        given(admisionRepository.existsByPacienteYFecha(1L, LocalDate.now())).willReturn(true);

        // WHEN / THEN
        assertThatThrownBy(() -> handler.handle(command))
            .isInstanceOf(BusinessException.class)
            .hasMessageContaining("active admission");

        verify(admisionRepository, never()).save(any());
        verify(eventPublisher, never()).publishEvent(any());
    }

    @Test
    @DisplayName("handle should throw EntityNotFoundException if doctor does not exist")
    void handle_nonExistentDoctor_shouldThrowEntityNotFoundException() {
        // GIVEN
        CrearAdmisionCommand command = new CrearAdmisionCommand(
            1L, "1234567890", 2L, 999L, LocalDate.now(), null
        );
        given(admisionRepository.existsByPacienteYFecha(any(), any())).willReturn(false);
        given(pacienteRepository.findById(1L)).willReturn(Optional.of(PacienteTestFactory.crearPaciente()));
        given(medicoRepository.findById(999L)).willReturn(Optional.empty());

        // WHEN / THEN
        assertThatThrownBy(() -> handler.handle(command))
            .isInstanceOf(EntityNotFoundException.class);
    }
}
```

---

## Repository Integration Test (TestContainers)

```java
// Tests against a real DB in Docker — slower but more reliable
@Tag("integration")
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class AdmisionRepositoryIT {

    @Container
    static final MSSQLServerContainer<?> sqlServer =
        new MSSQLServerContainer<>("mcr.microsoft.com/mssql/server:2019-latest")
            .acceptLicense();

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", sqlServer::getJdbcUrl);
        registry.add("spring.datasource.username", sqlServer::getUsername);
        registry.add("spring.datasource.password", sqlServer::getPassword);
    }

    @Autowired private AdmisionRepository admisionRepository;
    @Autowired private TestEntityManager entityManager;

    @Test
    @DisplayName("findByNumeroAdmision should return the correct admission")
    void findByNumeroAdmision_existing_shouldReturnAdmision() {
        // GIVEN — persist test data
        Admision admision = AdmisionTestFactory.crearAdmisionActiva();
        entityManager.persistAndFlush(admision);

        // WHEN
        Optional<Admision> result = admisionRepository.findByNumeroAdmision(admision.getNumeroAdmision());

        // THEN
        assertThat(result).isPresent();
        assertThat(result.get().getNumeroAdmision()).isEqualTo(admision.getNumeroAdmision());
    }
}
```

---

## Controller Integration Test (MockMvc)

```java
@Tag("integration")
@SpringBootTest
@AutoConfigureMockMvc
class AdmisionCommandControllerIT {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;

    @MockBean private CrearAdmisionCommandHandler crearHandler;

    @Test
    @DisplayName("POST /api/admisiones should return 201 with the created ID")
    @WithMockUser(roles = "ADMISION_WRITE")
    void crear_validRequest_shouldReturn201WithId() throws Exception {
        // GIVEN
        AdmisionRequestDTO request = new AdmisionRequestDTO(1L, 2L, 3L, LocalDate.now(), null);
        given(crearHandler.handle(any())).willReturn(99L);

        // WHEN / THEN
        mockMvc.perform(post("/api/admisiones")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$").value(99L));
    }

    @Test
    @DisplayName("POST /api/admisiones without authentication should return 401")
    void crear_noAuthentication_shouldReturn401() throws Exception {
        mockMvc.perform(post("/api/admisiones")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("POST /api/admisiones with wrong role should return 403")
    @WithMockUser(roles = "ADMISION_READ")
    void crear_wrongRole_shouldReturn403() throws Exception {
        mockMvc.perform(post("/api/admisiones")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isForbidden());
    }
}
```

---

## Test Factories — Reusable Test Data

```java
// Centralize test object creation — avoid duplication across tests
public class AdmisionTestFactory {

    public static Admision crearAdmisionActiva() {
        return crearAdmisionActiva(1L);
    }

    public static Admision crearAdmisionActiva(Long id) {
        Admision a = Admision.crear(
            PacienteTestFactory.crearPaciente(),
            MedicoTestFactory.crearMedico(),
            LocalDate.now()
        );
        ReflectionTestUtils.setField(a, "id", id);
        return a;
    }

    public static Admision crearAdmisionCerrada() {
        Admision a = crearAdmisionActiva();
        a.cerrar("Medical discharge");
        return a;
    }
}
```

---

## Test Checklist — Before Generating

```
□ Are domain tests tagged with @Tag("unit") and have no Spring context?
□ Do CommandHandler tests verify save() is NOT called when it fails?
□ Do CommandHandler tests verify events are NOT published when it fails?
□ Do controller integration tests cover the 401 (no auth) case?
□ Do controller integration tests cover the 403 (wrong role) case?
□ Do tests follow the GIVEN / WHEN / THEN pattern with comments?
□ Do test names follow the pattern: method_condition_expectedBehavior?
□ Are Test Factories in src/test and centralize data creation?
□ Are @DisplayName descriptions in English and describe business behavior?
