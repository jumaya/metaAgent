# Skill: Spring Security + JWT
**Used by:** `@backend-generator`, `@security-reviewer` | **Layer:** Backend / Security

---

## Main Spring Security Configuration

```java
// SecurityConfig — central point of all security configuration
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)     // ← enables @PreAuthorize
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)               // REST API — no CSRF needed
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS)) // no session
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()     // login/refresh without auth
                .requestMatchers("/actuator/health").permitAll() // public health check
                .requestMatchers("/v3/api-docs/**", "/swagger-ui/**").permitAll()
                .anyRequest().authenticated()                    // everything else: authenticated
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(UNAUTHORIZED))
                .accessDeniedHandler((req, res, denied) ->
                    res.setStatus(HttpStatus.FORBIDDEN.value()))
            )
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);     // cost 12 — balance security/performance
    }

    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## JWT Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");

        // No token → continue unauthenticated (config decides if it is required)
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        try {
            String token = authHeader.substring(7);
            String username = jwtService.extractUsername(token);

            // Only process if username exists and not yet authenticated
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (JwtException e) {
            // Invalid or expired token — 401 response
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("{\"error\": \"Invalid or expired token\"}");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

---

## JWT Service

```java
@Service
public class JwtService {

    // Secret from environment variables — NEVER hardcoded
    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration-ms:3600000}")   // default: 1 hour
    private long expirationMs;

    @Value("${jwt.refresh-expiration-ms:604800000}")  // default: 7 days
    private long refreshExpirationMs;

    public String generateToken(UserDetails userDetails) {
        return generateToken(Map.of(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(getSigningKey())
            .compact();
    }

    public String generateRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + refreshExpirationMs))
            .signWith(getSigningKey())
            .compact();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    private Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    private <T> T extractClaim(String token, Function<Claims, T> resolver) {
        return resolver.apply(
            Jwts.parser().verifyWith(getSigningKey()).build().parseSignedClaims(token).getPayload()
        );
    }

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));
    }
}
```

---

## Auth Controller — Login and Refresh

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
@Tag(name = "Authentication")
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<AuthResponseDTO> login(@Valid @RequestBody LoginRequestDTO request) {
        return ResponseEntity.ok(authService.login(request));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponseDTO> refresh(@Valid @RequestBody RefreshRequestDTO request) {
        return ResponseEntity.ok(authService.refreshToken(request.refreshToken()));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(HttpServletRequest request) {
        authService.logout(request.getHeader("Authorization"));
        return ResponseEntity.noContent().build();
    }
}

// Auth DTOs
public record LoginRequestDTO(@NotBlank String username, @NotBlank String password) {}
public record RefreshRequestDTO(@NotBlank String refreshToken) {}
public record AuthResponseDTO(String accessToken, String refreshToken, Long expiresIn) {}
```

---

## RBAC — Roles and Permissions in Controllers

```java
// Roles by domain granularity — pattern: ROLE_{DOMAIN}_{ACTION}
// Apply at class level (all methods) or at individual method level

// Option 1 — class-level security (recommended for simplicity)
@RestController
@PreAuthorize("hasRole('ROLE_ADMISION_WRITE')")
public class AdmisionCommandController { ... }

@RestController
@PreAuthorize("hasRole('ROLE_ADMISION_READ')")
public class AdmisionQueryController { ... }

// Option 2 — method-level security (more granular)
@RestController
public class AdmisionController {

    @PostMapping
    @PreAuthorize("hasRole('ROLE_ADMISION_WRITE')")
    public ResponseEntity<Long> crear(...) { ... }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ROLE_ADMISION_DELETE')")   // more restrictive role for delete
    public ResponseEntity<Void> eliminar(@PathVariable Long id) { ... }

    @GetMapping
    @PreAuthorize("hasAnyRole('ROLE_ADMISION_READ', 'ROLE_ADMIN')")
    public ResponseEntity<Page<?>> listar(...) { ... }
}

// Data-level security — filter by authenticated user
@PreAuthorize("hasRole('ROLE_MEDICO') and #medicoId == authentication.principal.id")
public List<AdmisionSummaryDTO> getMisAdmisiones(Long medicoId) { ... }
```

---

## application.yml — Secure Configuration

```yaml
# Never hardcode secrets — use environment variables
jwt:
  secret: ${JWT_SECRET}                    # export as environment variable
  expiration-ms: ${JWT_EXPIRATION:3600000} # default 1 hour
  refresh-expiration-ms: 604800000         # 7 days

spring:
  datasource:
    password: ${DB_PASSWORD}               # never in the repo yml
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OAUTH_ISSUER:}     # if using external OAuth2

# CORS — restrict to known origins
cors:
  allowed-origins: ${CORS_ORIGINS:http://localhost:3000}
  allowed-methods: GET,POST,PUT,DELETE,OPTIONS
  allowed-headers: "*"
  allow-credentials: true
  max-age: 3600
```

---

## Spring Security Checklist — Before Generating

```
□ Does the JWT secret come from an environment variable (never hardcoded)?
□ Do all endpoints have @PreAuthorize with a specific role?
□ Is the /api/auth/** endpoint in permitAll()?
□ Is the session configured as STATELESS?
□ Is CSRF disabled (REST API)?
□ Does BCryptPasswordEncoder have cost >= 10?
□ Does JwtFilter handle JwtException and return 401 (not 500)?
□ Does the refresh token have a longer expiration than the access token?
□ Do roles follow the pattern ROLE_{DOMAIN}_{ACTION}?
□ Is CORS configured with specific origins (not "*" in production)?
