# Exception Handling

The system adopts a layered exception handling architecture that allows each service to define its own error types while converting exceptions into a standardized response format through a unified global handler.

Key components:

- `AbstractServiceException`: Base exception class, defined in `common-shared`
- `BaseErrorType`: Error type interface, defined in `common-shared`
- `IErrorLevel`: Error level interface, defined in `common-shared`
- `ModelType`: Module type identifier, defined in `common-shared`
- Service-specific exception classes (e.g. `AuthServiceException`): defined within each `*-service`
- `BffExceptionHandler`: Global exception handler for the BFF layer, defined in `common-bff`
- `BizExceptionHandler`: Global exception handler for the service layer, defined in `common-biz`

---

## 1. AbstractServiceException

The base class for all custom exceptions, extending `RuntimeException`.

```java
public class AbstractServiceException extends RuntimeException {
    private final String errorCode;
    private final IErrorLevel errorLevel;
    private final ModelType modelType;
    private final String memo;

    public AbstractServiceException(BaseErrorType errorType, String exceptionMsg) {
        super(exceptionMsg);
        this.errorCode = errorType.getCustomErrorCode();
        this.errorLevel = errorType.getIErrorLevel();
        this.modelType = errorType.getModelType();
        this.memo = errorType.getMemo();
    }
    // Getters...
}
```

### Core Fields

| Field | Type | Description |
|---|---|---|
| `errorCode` | `String` | Error code identifying the specific error. Five-digit numeric format (e.g. `00001`) |
| `errorLevel` | `IErrorLevel` | Error severity level (e.g. `LOW`, `HIGH`) |
| `modelType` | `ModelType` | Module identifier (e.g. `AU` for the authentication module) |
| `memo` | `String` | Brief description of the error |

The `responseCode` returned to the client is composed of `modelType + errorCode`, for example `AU00001`. `ModelType` must be a fixed two-character uppercase string to ensure consistent formatting.

---

## 2. Service-Specific Exceptions

Each service defines its own exception class and a corresponding `ErrorType` enum, extending `AbstractServiceException`.

Example — authentication service:

```java
public class AuthServiceException extends AbstractServiceException {

    public enum AuthServiceErrorType implements BaseErrorType {
        USER_NOT_FOUND("00001", IErrorLevel.LOW, ModelType.AU, "User not found"),
        LOGIN_FAIL_OVER_THREE_TIME("00002", IErrorLevel.LOW, ModelType.AU, "Login failed more than 3 times"),
        ACCOUNT_PASSWD_NOT_MATCH("00003", IErrorLevel.LOW, ModelType.AU, "Account or password does not match"),
        ACCOUNT_BANNED("00004", IErrorLevel.LOW, ModelType.AU, "Account is banned"),
        ACCOUNT_ROLE_INACTIVE("00005", IErrorLevel.LOW, ModelType.AU, "Account has no active role assigned"),
        ACCOUNT_SESSION_SERIALIZE_FAILED("00006", IErrorLevel.HIGH, ModelType.AU, "Session serialization failed");

        // Constructor and interface implementation...
    }

    public AuthServiceException(BaseErrorType errorType) {
        super(errorType, errorType.getMemo());
    }
}
```

### ErrorType Design Benefits

- **Centralized management**: All error types are defined in a single enum, easy to review at a glance
- **Standardized**: Every error consistently carries a code, level, module, and description
- **Extensible**: Adding a new error only requires a new enum value, with no impact on existing logic

---

## 3. Usage Example

Throwing a custom exception inside a Service method:

```java
@Override
public LoginRs login(LoginRq rq) {

    CashaUser user = cashaUserRepository.findByUserAccount(rq.getUserAccount())
            .map(CashaUser::fromEntity)
            .orElseThrow(() -> new AuthServiceException(AuthServiceErrorType.USER_NOT_FOUND));

    if (isAccountLoginFailOverThreeTimes(user.getUsername())) {
        throw new AuthServiceException(AuthServiceErrorType.LOGIN_FAIL_OVER_THREE_TIME);
    }

    if (!user.isCredentialsValid(rq.getPasswd(), passwordEncoder)) {
        incrementLoginFailCount(rq.getUserAccount());
        throw new AuthServiceException(AuthServiceErrorType.ACCOUNT_PASSWD_NOT_MATCH);
    }

    if (!user.isActive()) {
        throw new AuthServiceException(AuthServiceErrorType.ACCOUNT_BANNED);
    }

    clearLoginFailCount(rq.getUserAccount());
    return loginSessionBuilderService.buildLoginSession(user);
}
```

---

## 4. Global Exception Handlers

### BffExceptionHandler (common-bff)

Converts exceptions into a standardized `ApiResponse` to return to the client.

```java
@Slf4j
@ControllerAdvice
public class BffExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse> handleMethodArgumentNotValidException(
            MethodArgumentNotValidException error) {
        // Handle parameter validation exceptions...
    }

    @ExceptionHandler(AbstractServiceException.class)
    public ResponseEntity<ApiResponse> handleCustomServiceException(AbstractServiceException error) {
        // HIGH-level errors can trigger alerts or be published to MQ
        // errorLogPublisher.publishErrorLogEvent(error);

        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ApiResponse(
                    error.getModelType() + error.getErrorCode(),
                    error.getMessage(),
                    null));
    }
}
```

### BizExceptionHandler (common-biz)

Handles exceptions at the service layer, primarily focused on log recording.

| | `BffExceptionHandler` | `BizExceptionHandler` |
|---|---|---|
| Location | `common-bff` | `common-biz` |
| Primary purpose | Convert to `ApiResponse` for the client | Log recording |
| HTTP Status | Semantically meaningful (400, 401, 403) | Typically 500 or custom |
| Additional handling | HIGH-level errors can trigger alerts / MQ | Error log recording |

---

## 5. FeignException Handling

When a downstream service throws a custom exception via Feign, it is wrapped as a `FeignException` on the upstream side. The upstream adapter is responsible for parsing and converting it into the appropriate domain exception.

```java
// Downstream order-service throws
// Error Code: OD00007
throw new OrderServiceException(OrderServiceErrorType.INSUFFICIENT_STOCK);
```

```java
// Upstream adapter layer
try {
    frs = orderFeign.checkout(frq);
} catch (FeignException e) {
    log.info(e.getMessage());

    if (e.contentUTF8().contains("OD00007")) {
        throw new OrderPortException(OrderPortErrorType.INSUFFICIENT_STOCK);
    }

    throw new OrderPortException(OrderPortErrorType.CLIENT_FAIL);
}
```

> The current FeignException parsing relies on string matching. This will be refactored to response body decoding in the adapter layer in a future iteration.

---

## 6. Module Location Summary

| Class | Location |
|---|---|
| `AbstractServiceException` | `common-shared` |
| `BaseErrorType` | `common-shared` |
| `IErrorLevel` | `common-shared` |
| `ModelType` | `common-shared` |
| `BffExceptionHandler` | `common-bff` |
| `BizExceptionHandler` | `common-biz` |
| `AuthServiceException` etc. | Each `*-service` internally |

Next Chapter: [PCI DSS Compliance](/guideline/6-pcidss.md)
