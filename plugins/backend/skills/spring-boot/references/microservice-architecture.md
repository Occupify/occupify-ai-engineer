# Microservice Layered Architecture & API Contracts

This reference manual provides architectural standards and implementation guidelines for building resilient, modular Spring Boot 3 microservices on Java 25.

---

## 1. Clean Layered Package Structure

Each microservice follows a strict package layout to maintain high cohesion and low coupling:

```
src/main/java/com/occupify/<service>/
├── config/                  # Spring configuration (@Configuration, Beans, Security)
├── constant/                # Route paths, message keys, queue names, error codes
├── controller/              # HTTP controllers (@RestController, inherits AbstractBaseController)
├── dto/
│   ├── request/             # Incoming request contracts (Java 25 records with @Valid)
│   └── response/            # Outgoing response payloads (Java 25 records)
├── event/                   # Event schemas, publishers, and RabbitMQ listener records
├── exception/               # BaseException, custom domain exceptions, GlobalExceptionHandler
├── mapper/                  # MapStruct interfaces (@Mapper(componentModel = "spring"))
├── repository/              # Spring Data JPA interfaces
└── service/
    ├── impl/                # Business logic implementation with @Transactional
    └── <Domain>Service.java # Interface defining domain capabilities
```

---

## 2. API Contract & ResponseFactory Taxonomy

All HTTP endpoints must return unified response envelopes. Never expose raw domain entities or unstandardized maps.

### 2.1 Unified Response Envelopes

```java
// Single object response
public record SingleResponse<T>(
    int status,
    String message,
    T data,
    Instant timestamp
) {}

// Created object response (HTTP 201)
public record CreatedResponse<T>(
    int status,
    String message,
    T data,
    Instant timestamp
) {}

// Action completed with no body (HTTP 200)
public record SuccessResponse(
    int status,
    String message,
    Instant timestamp
) {}

// Paginated response
public record PageResponse<T>(
    int status,
    String message,
    List<T> data,
    int pageNumber,
    int pageSize,
    long totalElements,
    int totalPages,
    boolean isFirst,
    boolean isLast,
    Instant timestamp
) {}

// Error response
public record ErrorResponse(
    int status,
    String code,
    String message,
    List<String> details,
    Instant timestamp
) {}
```

### 2.2 ResponseFactory Implementation

```java
package com.occupify.common.response;

import org.springframework.data.domain.Page;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import java.time.Instant;
import java.util.Collections;
import java.util.List;

public final class ResponseFactory {

    private ResponseFactory() {}

    public static <T> ResponseEntity<SingleResponse<T>> ok(T data, String message) {
        return ResponseEntity.ok(new SingleResponse<>(HttpStatus.OK.value(), message, data, Instant.now()));
    }

    public static <T> ResponseEntity<CreatedResponse<T>> created(T data, String message) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(new CreatedResponse<>(HttpStatus.CREATED.value(), message, data, Instant.now()));
    }

    public static ResponseEntity<SuccessResponse> success(String message) {
        return ResponseEntity.ok(new SuccessResponse(HttpStatus.OK.value(), message, Instant.now()));
    }

    public static <T> ResponseEntity<PageResponse<T>> page(Page<T> page, String message) {
        return ResponseEntity.ok(new PageResponse<>(
                HttpStatus.OK.value(),
                message,
                page.getContent(),
                page.getNumber(),
                page.getSize(),
                page.getTotalElements(),
                page.getTotalPages(),
                page.isFirst(),
                page.isLast(),
                Instant.now()
        ));
    }

    public static ResponseEntity<ErrorResponse> error(HttpStatus status, String code, String message, List<String> details) {
        return ResponseEntity.status(status).body(new ErrorResponse(
                status.value(),
                code,
                message,
                details != null ? details : Collections.emptyList(),
                Instant.now()
        ));
    }
}
```

### 2.3 AbstractBaseController Usage

All REST controllers inherit from `AbstractBaseController` for common logging, utility hooks, and standardized response generation:

```java
public abstract class AbstractBaseController {

    protected <T> ResponseEntity<SingleResponse<T>> responseOk(T data, String message) {
        return ResponseFactory.ok(data, message);
    }

    protected <T> ResponseEntity<CreatedResponse<T>> responseCreated(T data, String message) {
        return ResponseFactory.created(data, message);
    }

    protected ResponseEntity<SuccessResponse> responseSuccess(String message) {
        return ResponseFactory.success(message);
    }

    protected <T> ResponseEntity<PageResponse<T>> responsePage(Page<T> page, String message) {
        return ResponseFactory.page(page, message);
    }
}
```

---

## 3. Exception Hierarchy & Global Handler

### 3.1 BaseException

```java
public abstract class BaseException extends RuntimeException {
    private final HttpStatus status;
    private final String errorCode;

    protected BaseException(String message, HttpStatus status, String errorCode) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
    }

    public HttpStatus getStatus() { return status; }
    public String getErrorCode() { return errorCode; }
}
```

### 3.2 Standard Domain Exceptions
- `ResourceNotFoundException`: HTTP 404, code `NOT_FOUND`
- `BadRequestException`: HTTP 400, code `BAD_REQUEST`
- `UnauthorizedException`: HTTP 401, code `UNAUTHORIZED`
- `ForbiddenException`: HTTP 403, code `FORBIDDEN`
- `ConflictException`: HTTP 409, code `CONFLICT`

### 3.3 GlobalExceptionHandler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BaseException.class)
    public ResponseEntity<ErrorResponse> handleBaseException(BaseException ex) {
        log.warn("Business exception caught: [{}] {}", ex.getErrorCode(), ex.getMessage());
        return ResponseFactory.error(ex.getStatus(), ex.getErrorCode(), ex.getMessage(), List.of());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(err -> err.getField() + ": " + err.getDefaultMessage())
                .toList();
        return ResponseFactory.error(HttpStatus.BAD_REQUEST, "VALIDATION_ERROR", "Validation failed", errors);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        log.error("Unhandled server error: ", ex);
        return ResponseFactory.error(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERROR", "An unexpected error occurred", List.of());
    }
}
```

---

## 4. MapStruct DTO Mapping Standards

MapStruct ensures zero runtime reflection overhead and compile-time type safety:

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface PropertyMapper {

    Property toEntity(CreatePropertyRequest request);

    PropertyResponse toResponse(Property entity);

    List<PropertyResponse> toResponseList(List<Property> entities);
}
```

---

## 5. Centralized Constants Architecture

Do not scatter hardcoded strings across annotations or service classes:

```java
public final class RouteConstants {
    private RouteConstants() {}

    public static final String API_PREFIX = "/api/v1";
    public static final String NOTIFICATIONS_BASE = API_PREFIX + "/notifications";
    public static final String PROPERTIES_BASE = API_PREFIX + "/properties";
    public static final String AUTH_BASE = API_PREFIX + "/auth";
}

public final class MessageConstants {
    private MessageConstants() {}

    public static final String CREATED_SUCCESS = "Resource created successfully";
    public static final String RETRIEVED_SUCCESS = "Resource retrieved successfully";
    public static final String UPDATED_SUCCESS = "Resource updated successfully";
    public static final String DELETED_SUCCESS = "Resource deleted successfully";
}
```
