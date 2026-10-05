# Exception Handling in Spring Boot (User Service)

Quick revision notes and a reusable template for global exception handling in a Spring Boot REST API.

Project: E-commerce, User Service
Package: `com.ecommerce.user.exception`

---

## Table of Contents

1. [What is an Exception](#what-is-an-exception)
2. [Why We Need It](#why-we-need-it)
3. [How It Works](#how-it-works)
4. [Project Structure](#project-structure)
5. [Implementation](#implementation)
6. [Usage in Service Layer](#usage-in-service-layer)
7. [Sample Response](#sample-response)
8. [HTTP Status Cheat Sheet](#http-status-cheat-sheet)
9. [Changes for Different Projects](#changes-for-different-projects)
10. [Advanced: Base Exception with Error Code](#advanced-base-exception-with-error-code)
11. [Rules to Remember](#rules-to-remember)
12. [Common Mistakes](#common-mistakes)
13. [Revision Checklist](#revision-checklist)
14. [Interview Questions](#interview-questions)

---

## What is an Exception

- An unexpected problem that happens while the program is running
- Examples: user not found, duplicate email, invalid input, database down
- In Java it is an object that is thrown with `throw` and caught by a handler

---

## Why We Need It

| Without a handler | With a global handler |
|---|---|
| Ugly stack trace shown to client | Clean JSON error |
| Everything becomes 500 | Correct status codes (404, 409, 400) |
| try-catch in every controller | One central place |
| Internal details leak (security risk) | Safe generic message |
| Frontend cannot predict format | Same format for every error |

---

## How It Works

```
Service throws exception
        |
Exception leaves the controller
        |
@RestControllerAdvice catches it
        |
Matching @ExceptionHandler runs
        |
ResponseEntity (status + JSON body)
```

---

## Project Structure

```
com.ecommerce.user.exception
    ErrorResponse.java
    GlobalExceptionHandler.java
    ResourceNotFoundException.java
    ResourceAlreadyExistsException.java
```

---

## Implementation

### ErrorResponse.java

```java
package com.ecommerce.user.exception;

import java.time.Instant;
import java.util.Map;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record ErrorResponse(
        Instant timestamp,
        int status,
        String error,
        String message,
        String path,
        Map<String, String> details
) {
    public static ErrorResponse of(int status, String error, String message, String path) {
        return new ErrorResponse(Instant.now(), status, error, message, path, null);
    }

    public static ErrorResponse of(int status, String error, String message, String path,
                                   Map<String, String> details) {
        return new ErrorResponse(Instant.now(), status, error, message, path, details);
    }
}
```

### ResourceNotFoundException.java

```java
package com.ecommerce.user.exception;

public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String message) {
        super(message);
    }

    public ResourceNotFoundException(String resource, String field, Object value) {
        super(resource + " not found with " + field + ": " + value);
    }
}
```

### ResourceAlreadyExistsException.java

```java
package com.ecommerce.user.exception;

public class ResourceAlreadyExistsException extends RuntimeException {

    public ResourceAlreadyExistsException(String message) {
        super(message);
    }

    public ResourceAlreadyExistsException(String resource, String field, Object value) {
        super(resource + " already exists with " + field + ": " + value);
    }
}
```

### GlobalExceptionHandler.java

```java
package com.ecommerce.user.exception;

import java.util.LinkedHashMap;
import java.util.Map;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import jakarta.servlet.http.HttpServletRequest;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // 404
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex,
                                                        HttpServletRequest req) {
        return build(HttpStatus.NOT_FOUND, ex.getMessage(), req);
    }

    // 409
    @ExceptionHandler(ResourceAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> handleExists(ResourceAlreadyExistsException ex,
                                                      HttpServletRequest req) {
        return build(HttpStatus.CONFLICT, ex.getMessage(), req);
    }

    // 400 (@Valid failed)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex,
                                                          HttpServletRequest req) {
        Map<String, String> details = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(err -> details.putIfAbsent(err.getField(), err.getDefaultMessage()));

        return ResponseEntity.badRequest().body(
                ErrorResponse.of(400, "Validation Failed", "Invalid input", req.getRequestURI(), details));
    }

    // 400 (bad JSON body)
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ErrorResponse> handleBadJson(HttpMessageNotReadableException ex,
                                                       HttpServletRequest req) {
        return build(HttpStatus.BAD_REQUEST, "Malformed JSON request", req);
    }

    // 401 (wrong login)
    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<ErrorResponse> handleBadCredentials(BadCredentialsException ex,
                                                              HttpServletRequest req) {
        return build(HttpStatus.UNAUTHORIZED, "Invalid email or password", req);
    }

    // 403
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex,
                                                            HttpServletRequest req) {
        return build(HttpStatus.FORBIDDEN, "You do not have permission", req);
    }

    // 409 (DB unique constraint)
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrity(DataIntegrityViolationException ex,
                                                             HttpServletRequest req) {
        log.warn("Data integrity violation: {}", ex.getMostSpecificCause().getMessage());
        return build(HttpStatus.CONFLICT, "Data conflict. Duplicate or invalid reference", req);
    }

    // 500 (catch-all)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception ex, HttpServletRequest req) {
        log.error("Unexpected error at {}", req.getRequestURI(), ex);
        return build(HttpStatus.INTERNAL_SERVER_ERROR, "Something went wrong. Try again later", req);
    }

    private ResponseEntity<ErrorResponse> build(HttpStatus status, String message,
                                                HttpServletRequest req) {
        return ResponseEntity.status(status).body(
                ErrorResponse.of(status.value(), status.getReasonPhrase(), message, req.getRequestURI()));
    }
}
```

---

## Usage in Service Layer

```java
public User getById(Long id) {
    return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
}

public User register(RegisterRequest req) {
    if (userRepository.existsByEmail(req.email())) {
        throw new ResourceAlreadyExistsException("User", "email", req.email());
    }
    // save user
}
```

---

## Sample Response

```json
{
  "timestamp": "2026-10-06T10:15:30Z",
  "status": 404,
  "error": "Not Found",
  "message": "User not found with id: 42",
  "path": "/api/users/42"
}
```

Validation error:

```json
{
  "timestamp": "2026-10-06T10:16:10Z",
  "status": 400,
  "error": "Validation Failed",
  "message": "Invalid input",
  "path": "/api/users/register",
  "details": {
    "email": "must be a valid email",
    "password": "size must be between 8 and 64"
  }
}
```

---

## HTTP Status Cheat Sheet

| Situation | Status |
|---|---|
| Invalid input | 400 |
| Not logged in or wrong password | 401 |
| Logged in but no permission | 403 |
| Resource not found | 404 |
| Duplicate or conflict | 409 |
| Server bug | 500 |

---

## Changes for Different Projects

| Project type | What to change |
|---|---|
| Simple CRUD app | NotFound, AlreadyExists, Validation, catch-all. This is enough |
| E-commerce | Add `OutOfStockException` (409), `PaymentFailedException` (402), `InvalidOrderStateException` (409) |
| Banking or Fintech | Add `InsufficientBalanceException`, `TransactionFailedException`, plus an error code field (example: `ACC_1001`) |
| Microservices | Same `ErrorResponse` in every service, add `traceId`, handle `FeignException` or `RestClientException` |
| Public API | Error codes, docs link, i18n messages |
| Internal tool | Simple message is enough, keep detailed logs |

Reusable steps for any new project:

- Copy `ErrorResponse` and `GlobalExceptionHandler`
- Change the package name
- Add domain specific exceptions
- Remove handlers for libraries you do not use (example: Security)

---

## Advanced: Base Exception with Error Code

Idea: one parent class for all custom exceptions. Handler stays small.

```java
public abstract class BaseException extends RuntimeException {

    private final HttpStatus status;
    private final String errorCode;

    protected BaseException(HttpStatus status, String errorCode, String message) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
    }

    public HttpStatus getStatus() { return status; }
    public String getErrorCode() { return errorCode; }
}
```

```java
public class ResourceNotFoundException extends BaseException {
    public ResourceNotFoundException(String message) {
        super(HttpStatus.NOT_FOUND, "RESOURCE_NOT_FOUND", message);
    }
}
```

```java
@ExceptionHandler(BaseException.class)
public ResponseEntity<ErrorResponse> handleBase(BaseException ex, HttpServletRequest req) {
    return build(ex.getStatus(), ex.getMessage(), req);
}
```

- New exception means only a small new class
- Handler does not change

---

## Rules to Remember

- Use `RuntimeException` for custom exceptions (unchecked, clean with Spring)
- Specific handlers first, `Exception.class` last (Spring picks the most specific match)
- Never send stack trace or raw `ex.getMessage()` for 500 errors
- Log 4xx as `warn`, 5xx as `error`
- Keep one JSON format for all errors
- Throw in the service layer, convert in the handler
- No try-catch in controllers
- Filter exceptions (JWT) do not reach `@RestControllerAdvice`. Use `AuthenticationEntryPoint` and `AccessDeniedHandler` in Security config

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `Map.of(..., ex.getMessage())` with null message | Use a record or null safe value |
| Mixed keys like `"error"` and `"Error"` | Use one `ErrorResponse` class |
| Returning internal message in 500 | Generic message plus server log |
| Unused injected dependency in handler | Remove it |
| No handler for `AccessDeniedException` | Add 403 handler, else it becomes 500 |
| Same field error overwritten in validation map | Use `putIfAbsent` |

---

## Revision Checklist

- [ ] Custom exception classes created
- [ ] Common `ErrorResponse` format
- [ ] `@RestControllerAdvice` with handlers
- [ ] Correct HTTP status codes
- [ ] Catch-all returns generic message
- [ ] Logging added
- [ ] Validation errors shown field wise
- [ ] Security filter errors handled separately
- [ ] No unused code in handler

---

## Interview Questions

### Q1. What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?

- `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`
- It returns JSON directly, no need to add `@ResponseBody` on each handler
- Use `@ControllerAdvice` only when you return views (Thymeleaf, JSP)

---

### Q2. Two handlers match the same exception (parent and child). Which one runs?

- The most specific one wins
- Spring picks the handler whose exception type is closest in the class hierarchy
- Example: `ResourceNotFoundException` handler runs before the `Exception` handler
- Order of methods in the class does not matter

---

### Q3. Why does an exception thrown inside `JwtAuthenticationFilter` not reach `GlobalExceptionHandler`?

- Filters run before `DispatcherServlet`
- `@RestControllerAdvice` works only inside Spring MVC (after `DispatcherServlet`)
- Fix options:
  - Set `AuthenticationEntryPoint` (for 401) and `AccessDeniedHandler` (for 403) in Security config
  - Or catch the error in the filter and write the JSON response yourself
  - Or delegate to `HandlerExceptionResolver`

---

### Q4. Why do we extend `RuntimeException` and not `Exception` for custom exceptions?

- Unchecked exceptions do not force `throws` or try-catch everywhere
- Keeps service and controller code clean
- Important: `@Transactional` rolls back only on unchecked exceptions by default
- For checked exceptions you must write `@Transactional(rollbackFor = Exception.class)`

---

### Q5. You catch an exception inside a `@Transactional` method and do not rethrow. What happens?

- Transaction does **not** roll back, because Spring never sees the exception
- Data may be saved partly, which is dangerous
- Fix: rethrow the exception, or call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`

---

### Q6. Why not return 200 OK with an error message in the body?

- Breaks HTTP meaning, clients think the request succeeded
- Monitoring tools, API gateways, retries and caches depend on status codes
- Always use the correct status code (404, 409, 400 and so on)

---

### Q7. `existsByEmail()` check is done before saving. Is that enough to prevent duplicate users?

- No, there is a **race condition**
- Two requests at the same time can both pass the check
- Fix: add a **unique constraint** in the DB
- Then handle `DataIntegrityViolationException` and return 409
- Rule: app check gives a nice message, DB constraint gives the real safety

---

### Q8. Difference between `MethodArgumentNotValidException` and `ConstraintViolationException`?

| Exception | When it happens |
|---|---|
| `MethodArgumentNotValidException` | `@Valid` on `@RequestBody` object fails |
| `ConstraintViolationException` | `@Validated` on class, with `@RequestParam` or `@PathVariable` constraints (and JPA entity validation) |

- Newer Spring versions (Spring 6.1 and above) may throw `HandlerMethodValidationException` for method parameter validation
- Handle all of them if your API uses these validations

---

### Q9. Your catch-all `Exception` handler turns a wrong HTTP method (405) into 500. Why? How to fix?

- `Exception.class` handler also catches Spring's own exceptions like `HttpRequestMethodNotSupportedException`
- Fix: extend `ResponseEntityExceptionHandler`, it already handles standard Spring MVC exceptions with correct status
- Override `handleExceptionInternal` to keep your `ErrorResponse` format

---

### Q10. Why should we not send `ex.getMessage()` to the client for 500 errors?

- Can leak SQL queries, table names, file paths, library versions
- Helps attackers
- Correct way: log full error on server, send generic message to client
- Add a `traceId` so support can find the log

---

### Q11. 401 vs 403. What is the difference?

| Code | Meaning |
|---|---|
| 401 Unauthorized | Not authenticated (no token, bad token, wrong password) |
| 403 Forbidden | Authenticated, but not allowed |

---

### Q12. User A asks for the order of User B. Return 403 or 404?

- Many teams return **404**
- Reason: 403 confirms that the resource exists (information leak)
- Be consistent across the whole API

---

### Q13. Exceptions are slow. How to reduce the cost for custom exceptions?

- Most cost comes from building the stack trace (`fillInStackTrace`)
- For business exceptions that do not need a trace, disable it:

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message, null, false, false); // no suppression, no stack trace
    }
}
```

- Also: do not use exceptions for normal control flow (like loop exit)

---

### Q14. Local `@ExceptionHandler` in a controller vs global advice. Which one wins?

- Local handler inside the controller has **higher priority**
- Global advice is used only if the controller has no matching handler

---

### Q15. How to limit advice to some controllers only?

```java
@RestControllerAdvice(basePackages = "com.ecommerce.user.controller")
public class UserExceptionHandler { }
```

- Other options: `assignableTypes`, `annotations`
- With many advices, control order using `@Order`

---

### Q16. How to handle errors from another microservice (Feign or RestClient)?

- Catch the client exception and convert it to your own exception
- Do not pass the downstream message directly to the client
- Map status carefully (downstream 404 may need to become 502 or a domain exception)
- Add `traceId` in logs and response for tracking across services

---

### Q17. Difference between `@ResponseStatus` on an exception class and `ResponseEntity` in a handler?

| Approach | Use |
|---|---|
| `@ResponseStatus(HttpStatus.NOT_FOUND)` on exception | Quick, but no custom body |
| `ResponseEntity` in `@ExceptionHandler` | Full control: status, headers, JSON body |

- For real projects use the handler approach (common format)

---

### Q18. How do you test your exception handler?

- Use `@WebMvcTest` with `MockMvc`
- Mock the service to throw the exception
- Assert status and JSON fields

```java
@Test
void shouldReturn404WhenUserNotFound() throws Exception {
    when(userService.getById(1L))
            .thenThrow(new ResourceNotFoundException("User", "id", 1L));

    mockMvc.perform(get("/api/users/1"))
           .andExpect(status().isNotFound())
           .andExpect(jsonPath("$.message").value("User not found with id: 1"));
}
```

---

### Quick Answer Summary

| Topic | One line answer |
|---|---|
| Advice type | `@RestControllerAdvice` returns JSON |
| Handler match | Most specific exception wins |
| Filter errors | Use `AuthenticationEntryPoint` |
| Rollback | Only unchecked by default |
| Duplicate safety | DB unique constraint |
| 500 message | Generic to client, full log on server |
| Spring built-in errors | Extend `ResponseEntityExceptionHandler` |- [ ] No unused code in handler
