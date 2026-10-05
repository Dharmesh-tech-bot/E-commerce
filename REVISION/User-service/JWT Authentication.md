# JWT Authentication in Spring Boot (User Service)

Revision notes and reusable template for JWT token generation, validation and authentication with Spring Security.

Project: E-commerce, User Service
Stack: Spring Boot 3, Spring Security 6, jjwt 0.12.x

---

## Table of Contents

1. [What is JWT](#what-is-jwt)
2. [Why We Use It](#why-we-use-it)
3. [Full Flow](#full-flow)
4. [Dependencies and Config](#dependencies-and-config)
5. [Issues in the First Version](#issues-in-the-first-version)
6. [Improved Implementation](#improved-implementation)
7. [Login and Register Service](#login-and-register-service)
8. [Using the User Id in a Controller](#using-the-user-id-in-a-controller)
9. [Sample Responses](#sample-responses)
10. [Changes for Different Projects](#changes-for-different-projects)
11. [Rules to Remember](#rules-to-remember)
12. [Revision Checklist](#revision-checklist)
13. [Interview Questions: JWT and Spring Security](#interview-questions-jwt-and-spring-security)
14. [Interview Questions: AI Assistant Related](#interview-questions-ai-assistant-related)

---

## What is JWT

- JWT = JSON Web Token
- A signed string that proves who the user is
- Three parts separated by dots:

```
header.payload.signature
```

| Part | Contains |
|---|---|
| Header | Algorithm (example: HS256), token type |
| Payload (claims) | `sub` (user id), `email`, `role`, `iat`, `exp` |
| Signature | Hash of header + payload using the secret key |

- Payload is only **Base64URL encoded**, not encrypted. Anyone can read it
- Signature only proves it was **not changed**

---

## Why We Use It

| Session based login | JWT based login |
|---|---|
| Server stores session in memory or DB | Server stores nothing (stateless) |
| Hard to scale across many servers | Any server can verify the token |
| Cookie based | Usually `Authorization: Bearer <token>` |
| Easy to revoke | Hard to revoke before expiry |

- Good fit for REST APIs and microservices

---

## Full Flow

```
1. POST /api/users/login (email + password)
        |
2. Service checks password with BCrypt
        |
3. JwtUtil.generateToken() -> returns token
        |
4. Client sends: Authorization: Bearer <token>
        |
5. JwtAuthenticationFilter reads and validates token
        |
6. Sets Authentication in SecurityContext
        |
7. Spring checks authorization rules
        |
8. Controller runs (user id available from principal)
```

---

## Dependencies and Config

### pom.xml

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

### application.yml

```yaml
app:
  jwt:
    secret: ${JWT_SECRET}      # at least 32 characters, from environment variable
    expiration-ms: 900000      # 15 minutes
```

- Never commit the real secret to GitHub
- HS256 needs a key of at least 256 bits (32 bytes), else the app fails at startup

---

## Issues in the First Version

| # | Issue | Why it matters | Fix |
|---|---|---|---|
| 1 | `getBody()` used on `Jws<Claims>` | Deprecated in jjwt 0.12 | Use `getPayload()` |
| 2 | Token parsed two times in the filter | Wasted work | Parse once, reuse claims |
| 3 | `role` stored in token but authorities are `emptyList()` | Role based security will never work | Build authorities from the `role` claim |
| 4 | Filter swallows exception without log | Hard to debug | Log at `warn` level |
| 5 | No `AuthenticationEntryPoint` | Missing or bad token gives default 403, not 401 | Add custom 401 and 403 handlers |
| 6 | `UserRepository` injected in `JwtUtil` but not used | Dead code, can cause circular dependency | Remove it |
| 7 | `Secret.getBytes()` without charset | Depends on platform | Use `StandardCharsets.UTF_8` |
| 8 | `/actuator/**` is fully public | Can expose internal info | Allow only `/actuator/health` |
| 9 | Bean method named `PasswordEncoder`, field named `JwtFilter` | Breaks Java naming style | Use `passwordEncoder`, `jwtFilter` |
| 10 | No `jti` in token | Cannot blocklist a single token | Add `.id(UUID)` |

---

## Improved Implementation

### JwtUtil.java

```java
package com.ecommerce.user.util;

import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.UUID;

import javax.crypto.SecretKey;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;

@Component
public class JwtUtil {

    private final SecretKey key;
    private final long jwtExpirationMs;

    public JwtUtil(@Value("${app.jwt.secret}") String secret,
                   @Value("${app.jwt.expiration-ms}") long jwtExpirationMs) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.jwtExpirationMs = jwtExpirationMs;
    }

    // GENERATE
    public String generateToken(Long userId, String email, String role) {
        long now = System.currentTimeMillis();
        return Jwts.builder()
                .id(UUID.randomUUID().toString())
                .subject(String.valueOf(userId))
                .claim("email", email)
                .claim("role", role)
                .issuedAt(new Date(now))
                .expiration(new Date(now + jwtExpirationMs))
                .signWith(key)
                .compact();
    }

    // VALIDATE + READ (throws JwtException if token is bad or expired)
    public Claims parseClaims(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    public boolean isValid(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

### JwtAuthenticationFilter.java

```java
package com.ecommerce.user.util;

import java.io.IOException;
import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(JwtAuthenticationFilter.class);
    private static final String BEARER = "Bearer ";

    private final JwtUtil jwtUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String header = request.getHeader(HttpHeaders.AUTHORIZATION);

        if (header != null && header.startsWith(BEARER)) {
            String token = header.substring(BEARER.length());
            try {
                Claims claims = jwtUtil.parseClaims(token);

                Long userId = Long.valueOf(claims.getSubject());
                String role = claims.get("role", String.class);

                List<SimpleGrantedAuthority> authorities = (role == null)
                        ? List.of()
                        : List.of(new SimpleGrantedAuthority("ROLE_" + role));

                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(userId, null, authorities);

                SecurityContextHolder.getContext().setAuthentication(authentication);

            } catch (JwtException | IllegalArgumentException e) {
                log.warn("Invalid JWT on {}: {}", request.getRequestURI(), e.getMessage());
                SecurityContextHolder.clearContext();
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

### RestAuthenticationEntryPoint.java (401)

```java
package com.ecommerce.user.config;

import java.io.IOException;

import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import com.ecommerce.user.exception.ErrorResponse;
import com.fasterxml.jackson.databind.ObjectMapper;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;

@Component
@RequiredArgsConstructor
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException ex) throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        objectMapper.writeValue(response.getOutputStream(),
                ErrorResponse.of(401, "Unauthorized",
                        "Authentication required or token is invalid", request.getRequestURI()));
    }
}
```

### RestAccessDeniedHandler.java (403)

```java
package com.ecommerce.user.config;

import java.io.IOException;

import org.springframework.http.MediaType;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import com.ecommerce.user.exception.ErrorResponse;
import com.fasterxml.jackson.databind.ObjectMapper;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;

@Component
@RequiredArgsConstructor
public class RestAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException ex) throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        objectMapper.writeValue(response.getOutputStream(),
                ErrorResponse.of(403, "Forbidden",
                        "You do not have permission", request.getRequestURI()));
    }
}
```

### SecurityConfig.java

```java
package com.ecommerce.user.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

import com.ecommerce.user.util.JwtAuthenticationFilter;

import lombok.RequiredArgsConstructor;

@Configuration
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    private final RestAuthenticationEntryPoint authenticationEntryPoint;
    private final RestAccessDeniedHandler accessDeniedHandler;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sess -> sess.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .exceptionHandling(ex -> ex
                    .authenticationEntryPoint(authenticationEntryPoint)
                    .accessDeniedHandler(accessDeniedHandler))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/users/register", "/api/users/login", "/actuator/health").permitAll()
                    .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

### Optional: stop the filter from registering twice

`@Component` filters are also auto-registered in the servlet container. `OncePerRequestFilter` protects you from running twice, but this makes it clean:

```java
@Bean
public FilterRegistrationBean<JwtAuthenticationFilter> jwtFilterRegistration(JwtAuthenticationFilter filter) {
    FilterRegistrationBean<JwtAuthenticationFilter> reg = new FilterRegistrationBean<>(filter);
    reg.setEnabled(false);
    return reg;
}
```

---

## Login and Register Service

Adjust entity and DTO names to your project.

```java
public void register(RegisterRequest req) {
    if (userRepository.existsByEmail(req.email())) {
        throw new ResourceAlreadyExistsException("User", "email", req.email());
    }
    User user = new User();
    user.setEmail(req.email());
    user.setPassword(passwordEncoder.encode(req.password()));   // never store plain password
    userRepository.save(user);
}

public LoginResponse login(LoginRequest req) {
    User user = userRepository.findByEmail(req.email())
            .orElseThrow(() -> new BadCredentialsException("Invalid email or password"));

    if (!passwordEncoder.matches(req.password(), user.getPassword())) {
        throw new BadCredentialsException("Invalid email or password");   // same message
    }

    String token = jwtUtil.generateToken(user.getId(), user.getEmail(), user.getRole().name());
    return new LoginResponse(token, "Bearer");
}
```

- Same message for wrong email and wrong password (stops user enumeration)
- `BadCredentialsException` is handled in `GlobalExceptionHandler` as 401

---

## Using the User Id in a Controller

Principal is the `Long` userId, so this works:

```java
@GetMapping("/api/users/me")
public UserResponse me(@AuthenticationPrincipal Long userId) {
    return userService.getById(userId);
}
```

Role based access:

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/api/users/{id}")
public void delete(@PathVariable Long id) { ... }
```

- Always take the user id from the token, never from request body or query param

---

## Sample Responses

Login success:

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "type": "Bearer"
}
```

Missing or bad token (401):

```json
{
  "timestamp": "2026-10-06T10:20:00Z",
  "status": 401,
  "error": "Unauthorized",
  "message": "Authentication required or token is invalid",
  "path": "/api/users/me"
}
```

Wrong role (403):

```json
{
  "timestamp": "2026-10-06T10:21:00Z",
  "status": 403,
  "error": "Forbidden",
  "message": "You do not have permission",
  "path": "/api/users/5"
}
```

---

## Changes for Different Projects

| Project type | What to change |
|---|---|
| Simple monolith | HS256 with one secret, 15 to 60 minutes expiry |
| Microservices | Use RS256 or ES256. Auth service signs with private key, other services verify with public key (JWKS) |
| Public API | Short access token plus refresh token, rate limit login |
| Web app with cookies | Store token in `HttpOnly`, `Secure`, `SameSite` cookie and enable CSRF protection |
| Mobile app | Access token in secure storage (Keychain or Keystore), refresh token rotation |
| Admin panel | Shorter expiry, check role from DB on sensitive actions |

---

## Rules to Remember

- JWT payload is readable. Never put password, card number or private data inside
- Keep access token short (5 to 15 minutes)
- Secret from environment variable or vault, never in Git
- Pin the algorithm: use `verifyWith(key)` and `parseSignedClaims` (rejects unsigned tokens)
- Authentication = who are you. Authorization = what can you do
- 401 = not logged in. 403 = logged in but not allowed
- Filter exceptions do not reach `@RestControllerAdvice`. Use `AuthenticationEntryPoint` and `AccessDeniedHandler`
- Use `ROLE_` prefix for authorities when using `hasRole()`
- Hash passwords with BCrypt, compare with `matches()`
- Do not log the full `Authorization` header

---

## Revision Checklist

- [ ] Secret length at least 32 characters, loaded from environment
- [ ] Token has `sub`, `exp`, `iat`, `jti`
- [ ] Token parsed once in the filter
- [ ] Authorities built from role claim
- [ ] 401 and 403 handlers added
- [ ] Only needed endpoints are `permitAll`
- [ ] Passwords stored with BCrypt
- [ ] Same error message for wrong email and wrong password
- [ ] User id read from token, not from request
- [ ] No unused injection in `JwtUtil`
- [ ] Tests for valid, expired, tampered and missing token

---

## Interview Questions: JWT and Spring Security

### Q1. Is JWT encrypted?

- No. A normal JWT (JWS) is only **signed**, payload is Base64URL encoded
- Anyone can decode and read it
- Encrypted version is JWE, rarely used
- So never store secrets or sensitive data in claims

---

### Q2. HS256 vs RS256. When to use which?

| | HS256 | RS256 |
|---|---|---|
| Type | Symmetric (one shared secret) | Asymmetric (private and public key) |
| Who can create tokens | Anyone with the secret | Only owner of private key |
| Good for | Single service | Microservices, third party verification |

- With HS256, every service that verifies can also forge tokens
- With RS256, services only need the public key

---

### Q3. What is the `alg: none` attack and the algorithm confusion attack?

- `none`: attacker removes signature and sets `alg` to `none`, a weak library accepts it
- Confusion: attacker changes RS256 to HS256 and signs with the public key as the secret
- Defense: **pin the expected algorithm and key**
- In jjwt: `verifyWith(key)` with `parseSignedClaims` rejects unsigned and wrong algorithm tokens

---

### Q4. JWT is stateless. How do you log a user out or revoke a token?

- Pure JWT cannot be revoked before it expires
- Options:
  - Short expiry plus refresh token
  - Blocklist of `jti` in Redis with TTL equal to remaining token life
  - Token version field in DB, compare on each request
- Logout on client alone only deletes the token, it does not make it invalid

---

### Q5. How does a refresh token flow work? What is rotation?

- Access token: short life, sent on every request
- Refresh token: long life, used only to get a new access token
- Rotation: each refresh gives a **new** refresh token and the old one is invalid
- If an old refresh token is used again, treat it as theft and revoke the whole token family
- Store refresh tokens **hashed** in the DB

---

### Q6. Where should the client store the JWT?

| Place | Risk |
|---|---|
| `localStorage` | Stolen by XSS |
| `HttpOnly` cookie | Safe from XSS, but needs CSRF protection |
| Memory only | Lost on refresh, safest from theft |

- No option is perfect. Choose based on threat model
- Mobile: Keychain or Keystore

---

### Q7. Why is `csrf.disable()` fine in this project?

- CSRF attacks use cookies that the
