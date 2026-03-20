# 異常處置

本系統採用分層的異常處理架構，讓各服務能夠定義自己的錯誤類型，同時透過統一的處理器將異常轉換為標準回應格式。

主要組件：

- `AbstractServiceException`：基礎自定義異常類，定義於 `common-shared`
- `BaseErrorType`：錯誤類型介面，定義於 `common-shared`
- `IErrorLevel`：錯誤等級介面，定義於 `common-shared`
- `ModelType`：模組類型標識，定義於 `common-shared`
- 各服務特定的異常類（如 `AuthServiceException`）：定義於各 `*-service` 內部
- `BffExceptionHandler`：BFF 層全局異常處理器，定義於 `common-bff`
- `BizExceptionHandler`：Service 層全局異常處理器，定義於 `common-biz`

---

## 1. AbstractServiceException

所有自定義異常的基底類別，繼承自 `RuntimeException`。

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

### 核心屬性

| 屬性 | 類型 | 說明 |
|---|---|---|
| `errorCode` | `String` | 錯誤代碼，用於識別特定錯誤，格式為五位數字（如 `00001`） |
| `errorLevel` | `IErrorLevel` | 錯誤嚴重程度（如 `LOW`、`HIGH`） |
| `modelType` | `ModelType` | 模組類型標識（如 `AU` 表示認證模組） |
| `memo` | `String` | 錯誤的簡要描述 |

回應給前端的 `responseCode` 由 `modelType + errorCode` 組合而成，例如 `AU00001`。`ModelType` 須為固定兩位英文大寫，確保格式一致。

---

## 2. 服務特定異常

每個服務定義自己的異常類與對應的 `ErrorType` enum，繼承 `AbstractServiceException`。

以認證服務為例：

```java
public class AuthServiceException extends AbstractServiceException {

    public enum AuthServiceErrorType implements BaseErrorType {
        USER_NOT_FOUND("00001", IErrorLevel.LOW, ModelType.AU, "查無用戶"),
        LOGIN_FAIL_OVER_THREE_TIME("00002", IErrorLevel.LOW, ModelType.AU, "登入失敗超過3次"),
        ACCOUNT_PASSWD_NOT_MATCH("00003", IErrorLevel.LOW, ModelType.AU, "帳號密碼不匹配"),
        ACCOUNT_BANNED("00004", IErrorLevel.LOW, ModelType.AU, "用戶帳號遭禁止"),
        ACCOUNT_ROLE_INACTIVE("00005", IErrorLevel.LOW, ModelType.AU, "用戶尚未綁定權限"),
        ACCOUNT_SESSION_SERIALIZE_FAILED("00006", IErrorLevel.HIGH, ModelType.AU, "用戶 Session 序列化失敗");

        // 構造方法和介面實作...
    }

    public AuthServiceException(BaseErrorType errorType) {
        super(errorType, errorType.getMemo());
    }
}
```

### ErrorType 設計優點

- **集中管理**：所有錯誤類型在同一個 enum 中定義，一目瞭然
- **標準化**：每個錯誤統一包含錯誤碼、等級、模組與描述
- **可擴展**：新增錯誤只需添加新的 enum 值，不影響現有邏輯

---

## 3. 異常使用範例

在 Service 方法中拋出自定義異常：

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

## 4. 全局異常處理器

### BffExceptionHandler（common-bff）

負責將異常轉換為標準 `ApiResponse` 回傳給前端。

```java
@Slf4j
@ControllerAdvice
public class BffExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse> handleMethodArgumentNotValidException(
            MethodArgumentNotValidException error) {
        // 處理參數驗證異常...
    }

    @ExceptionHandler(AbstractServiceException.class)
    public ResponseEntity<ApiResponse> handleCustomServiceException(AbstractServiceException error) {
        // HIGH 級錯誤可額外發送告警或寫入 MQ
        // errorLogPublisher.publishErrorLogEvent(error);

        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ApiResponse(
                    error.getModelType() + error.getErrorCode(),
                    error.getMessage(),
                    null));
    }
}
```

### BizExceptionHandler（common-biz）

負責 Service 層的異常處理，以記錄 log 為主，行為與 `BffExceptionHandler` 略有不同。

| | `BffExceptionHandler` | `BizExceptionHandler` |
|---|---|---|
| 位置 | `common-bff` | `common-biz` |
| 主要用途 | 轉換為 `ApiResponse` 回傳前端 | Log 記錄為主 |
| HTTP Status | 對前端有語意（400、401、403） | 通常為 500 或自訂 |
| 額外處理 | HIGH 級錯誤可觸發告警 / MQ | 記錄錯誤 log |

---

## 5. FeignException 的處置

微服務間透過 Feign 呼叫時，下游拋出的自定義異常會被包裝成 `FeignException` 傳遞給上游。上游需自行解析並轉換為對應的 domain exception。

```java
// 下游 order-service 拋出
// Error Code：OD00007
throw new OrderServiceException(OrderServiceErrorType.INSUFFICIENT_STOCK);
```

```java
// 上游 Adapter 層處理
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

> FeignException 的解析方式目前採用字串比對，後續將優化為 response body decode，統一在 Adapter 層處理。

---

## 6. 模組位置總覽

| 類別 | 位置 |
|---|---|
| `AbstractServiceException` | `common-shared` |
| `BaseErrorType` | `common-shared` |
| `IErrorLevel` | `common-shared` |
| `ModelType` | `common-shared` |
| `BffExceptionHandler` | `common-bff` |
| `BizExceptionHandler` | `common-biz` |
| `AuthServiceException` 等 | 各 `*-service` 內部 |

下一章: [PCI DSS 合規規範](/guideline/6-pcidss-zh-tw.md)