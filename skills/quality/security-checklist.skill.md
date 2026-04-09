# Skill: OWASP Security Checklist — By Layer
**Used by:** `@security-reviewer` | **Layer:** Quality / Security

---

## Layer 1 — REST API (Controllers)

```
□ A01 - Broken Access Control
  □ Does every endpoint have @PreAuthorize with a specific role?
  □ Are there no endpoints accessible without authentication (except /auth/** and /actuator/health)?
  □ Can a user's resources NOT be accessed by another user (IDOR)?
     → Verify: is the resource ID validated against the authenticated user?
  □ Does the endpoint not expose more data than the role is allowed to see?

□ A03 - Injection
  □ Are there NO @Query with nativeQuery=true built with string concatenation?
  □ Do all query parameters use @Param (named parameters)?
  □ Are there NO StringBuilder/String.format() building dynamic SQL/JPQL?

□ A04 - Insecure Design
  □ Do write endpoints (POST/PUT/DELETE) have @Valid validation?
  □ Do input DTOs have validation annotations on every field?
  □ Does the endpoint never return JPA entities directly (only DTOs)?

□ A05 - Security Misconfiguration
  □ Does CORS have specific allowedOrigins (not "*")?
  □ Are Actuator endpoints protected or limited to /health?
  □ Do error messages not expose stacktraces or internal details?
  □ Is Swagger/OpenAPI disabled in production or protected?

□ A07 - Auth Failures
  □ Does the JWT expire in at most 1 hour?
  □ Does the refresh token expire in at most 7 days?
  □ Is there a logout endpoint that invalidates the token?
  □ Do authentication errors return a generic 401 (without revealing if user exists)?
```

---

## Layer 2 — Services / Handlers

```
□ A01 - Broken Access Control
  □ Does permission validation not rely solely on the frontend?
  □ Do CommandHandlers verify the resource belongs to the user when applicable?

□ A02 - Cryptographic Failures
  □ Are passwords stored with BCrypt (cost >= 10)?
  □ Are there no passwords, tokens or API keys in logs?
  □ Does the JWT secret have at least 256 bits?
  □ Are sensitive fields in DB (doc, phone) encrypted if applicable?

□ A04 - Insecure Design
  □ Do destructive operations have a confirmation step?
  □ Do CommandHandlers validate current state before modifying?
  □ Is there rate limiting for expensive or sensitive operations?

□ A09 - Logging Failures
  □ Are security events logged? (failed login, access denied)
  □ Do logs NOT contain sensitive data? (passwords, tokens, document numbers)
  □ Do logs have enough context for forensics? (userId, IP, action)
  □ Is MDC used to correlate logs for the same request?
```

---

## Layer 3 — Persistence / Repository

```
□ A03 - Injection
  □ Do all queries use named parameters (:param) not concatenation?
  □ Do native queries (if any) have a justification comment?
  □ Are user inputs sanitized before using them in LIKE?
     → Correct: CONCAT('%', :param, '%') with escaped param

□ A04 - Insecure Design
  □ Do repositories not expose bulk delete methods without confirmation?
  □ Do listing queries have a result limit (Pageable)?

□ A06 - Vulnerable Components
  □ Does the Hibernate version have no known CVEs?
  □ Is Spring Data JPA up to date?
```

---

## Layer 4 — Configuration and Infrastructure

```
□ A02 - Cryptographic Failures
  □ Does communication use HTTPS in all environments (including staging)?
  □ Are secrets in environment variables (not in the repo's application.yml)?
  □ Does .gitignore exclude .env and application-local.yml?

□ A05 - Security Misconfiguration
  □ Are security headers configured?
     → X-Content-Type-Options: nosniff
     → X-Frame-Options: DENY
     → Strict-Transport-Security (HSTS)
     → Content-Security-Policy
  □ Does the Docker image NOT run as root?
  □ Are sensitive environment variables NOT in the Dockerfile or docker-compose?
  □ Are only the necessary ports exposed in Docker?

□ A08 - Software and Data Integrity
  □ Do Maven/NPM dependencies have fixed versions (not ranges)?
  □ Is a dependency check run in the CI pipeline?

□ A09 - Logging Failures
  □ Are production logs NOT at DEBUG level?
  □ Are logs stored persistently (not only in the container)?
```

---

## Layer 5 — Frontend (Next.js)

```
□ A01 - Broken Access Control
  □ Do protected routes have authentication middleware?
  □ Do Server Actions validate permissions on the server (not just in the UI)?
  □ Is sensitive data NOT in localStorage (use httpOnly cookie)?

□ A03 - Injection (XSS)
  □ Is dangerouslySetInnerHTML used? → If so, sanitize with DOMPurify
  □ Are user inputs displayed with React's automatic escaping?
  □ Do external links use rel="noopener noreferrer"?

□ A05 - Security Misconfiguration
  □ Does next.config.js have security headers configured?
  □ Is the JWT token stored in an httpOnly cookie (not localStorage)?
  □ Do client environment variables (NEXT_PUBLIC_*) not contain secrets?

□ A07 - Auth Failures
  □ Does logout correctly clear the JWT cookie?
  □ Do admin routes verify the role on the server (middleware)?
```

---

## Security Report — Standard Format

```markdown
## Security Report — {frmX}.frm
**Reviewed by:** @security-reviewer | **Date:** {date}

### Summary
| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | 0 |
| 🟠 HIGH | 0 |
| 🟡 MEDIUM | 0 |
| 🟢 LOW | 0 |

### Findings

#### 🔴 CRITICAL — {description}
- **File:** `src/main/java/.../Controller.java`
- **Line:** 45
- **OWASP:** A01:2021 - Broken Access Control
- **Description:** Endpoint allows accessing any patient's data without verifying ownership
- **Remediation:** Add `@PreAuthorize("... and #id == authentication.principal.patientId")`

### ✅ Passed Checks
- JWT with 1-hour expiration
- BCrypt cost 12
- All endpoints with @PreAuthorize
- No queries with string concatenation found
