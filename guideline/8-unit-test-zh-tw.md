# 單元測試規範

## 1. 指導原則

本專案採用 Clean Architecture 架構，不強制 TDD，但透過 JaCoCo 強制覆蓋率，確保所有業務核心層在合併前都經過驗證。

Coverage 追蹤範圍集中在包含業務邏輯的層：Domain Service、UseCase 實作、以及 Adapter。基礎設施膠水代碼（Controller、DTO、Bootstrap 設定）明確排除在指標外，但仍需在適當範圍內進行測試。

---

## 2. Coverage 涵蓋範圍

| 層級 | Package | 納入 Coverage | 備註 |
|---|---|---|---|
| Domain Service | `domain/service/**` | **是** | 必須覆蓋 Spec 內所有 Scenario |
| UseCase / UseCaseImpl | `application/usecase/**` | **是** | 僅限 BFF |
| Adapter — Persistence | `adapter/persistence/**` | **是** | |
| Adapter — Client / Messaging | `adapter/client/**`、`adapter/messaging/**` | **是** | |
| Controller | `application/api/controller/**` | 否 | 有測試，但排除在指標外 |
| DTOs / Contracts | `application/api/dto/**` | 否 | 純資料類別 |
| Domain Model / VO / Port | `domain/model/**`、`domain/vo/**`、`domain/port/**` | 否 | 介面與值物件 |
| `common/**` 工具類 | `common/**` | 否 | 有測試，但排除在指標外 |
| Bootstrap / 應用程式入口 | `bootstrap/**` | 否 | |

> **Coverage 指標：** JaCoCo **instruction coverage**（位元組碼指令覆蓋率，JaCoCo 預設值）。
> 80% 門檻套用於 instruction coverage，而非 line coverage 或 branch coverage。
> Instruction coverage 比 line coverage 稍嚴格，比 branch coverage 寬鬆。

---

## 3. 各層測試策略

### 3.1 Domain Service

- 使用 **Mockito** mock 所有 Port 依賴。
- 每個測試方法必須對應 API Spec 中定義的一個 Scenario。
- 在此層完成完整的 Scenario 覆蓋，可降低 Controller 層的測試負擔。

```java
@ExtendWith(MockitoExtension.class)
class LoginDomainServiceTest {

    @Mock
    private CashaUserPort cashaUserPort;

    @Mock
    private LoginSessionBuilderPort loginSessionBuilderPort;

    @InjectMocks
    private LoginDomainServiceImpl loginDomainService;

    @Test
    void login_success() { ... }

    @Test
    void login_userNotFound_throwsAuthServiceException() { ... }

    @Test
    void login_loginFailOverThreeTimes_throwsAuthServiceException() { ... }

    @Test
    void login_passwordMismatch_throwsAuthServiceException() { ... }

    @Test
    void login_accountBanned_throwsAuthServiceException() { ... }
}
```

### 3.2 UseCase / UseCaseImpl（僅 BFF）

- 透過 Port 介面 mock Domain Service 依賴。
- 測試編排邏輯：欄位映射、條件分支、跨服務呼叫的錯誤傳播。
- Domain Service 的 Scenario 覆蓋委由 Domain Service 測試負責。

### 3.3 Repository（`adapter.persistence`）

- 使用 **H2** 記憶體資料庫。
- 以 MySQL 相容模式設定 H2。
- 測試所有自定義查詢方法（超出標準 JPA CRUD 的部分）。

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=MySQL;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect
```

```java
@SpringBootTest
@ActiveProfiles("test")
class CashaUserRepositoryTest {

    @Autowired
    private CashaUserRepository cashaUserRepository;

    @Test
    void findByUserAccount_exists_returnsEntity() { ... }

    @Test
    void findByUserAccount_notExists_returnsEmpty() { ... }
}
```

### 3.4 Adapter — Redis / Kafka / Messaging

- 使用 **Testcontainers** 啟動真實基礎設施容器。
- 驗證實際的寫入 / 讀取 / 發布行為，不 mock 客戶端函式庫。

```java
@SpringBootTest
@Testcontainers
class RedisSessionAdapterTest {

    @Container
    static GenericContainer<?> redis =
            new GenericContainer<>("redis:7").withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private RedisSessionAdapter redisSessionAdapter;

    @Test
    void saveSession_thenReadBack_returnsExpected() { ... }
}
```

### 3.5 Controller

- 使用 `@SpringBootTest` 搭配 `MockMvc`。
- **必測：** 一個 Happy Path — 驗證回應格式符合 `ApiResponse` 結構。
- **必測：** 1～2 個錯誤情境 — 驗證 `ApiResponse` 內回傳正確的 `responseCode`。
- 完整的業務 Scenario 覆蓋委由 Domain Service 測試負責。

```java
@SpringBootTest
@AutoConfigureMockMvc
class LoginControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void login_happyPath_returnsTokenInApiResponse() throws Exception {
        mockMvc.perform(post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        { "userAccount": "test", "passwd": "Password1!" }
                        """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.responseCode").value("0000"))
                .andExpect(jsonPath("$.data.token").isNotEmpty());
    }

    @Test
    void login_userNotFound_returnsErrorCode() throws Exception {
        mockMvc.perform(post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        { "userAccount": "nonexistent", "passwd": "Password1!" }
                        """))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.responseCode").value("AU00001"));
    }
}
```

### 3.6 Common 工具類

- `common/**` 下的所有工具類**必須**撰寫測試。
- 這些測試**不**計入 Coverage 指標（透過 `sonar.coverage.exclusions` 排除）。

---

## 4. 品質門檻

| 指標 | 門檻 | 執行方式 |
|---|---|---|
| JaCoCo instruction coverage | ≥ 80% | 推送前開發者自行確認；CI/CD Pipeline 規則啟用後自動強制 |
| SonarQube Major Issues | 0 | CI/CD Pipeline 規則 |

推送分支前，Coverage 必須維持在 80% 以上。

---

## 5. Maven POM 設定

### 5.1 Service 模組（以 `auth-service` 為例）

```xml
<properties>
    <sonar.coverage.exclusions>
        **/com/willcard/auth/bootstrap/**,
        **/com/willcard/auth/domain/model/**,
        **/com/willcard/auth/domain/vo/**,
        **/com/willcard/auth/domain/dto/**,
        **/com/willcard/auth/domain/port/**,
        **/com/willcard/common/**,
        **/AuthApplication.java
    </sonar.coverage.exclusions>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.8</version>
            <executions>
                <execution>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                    <configuration>
                        <includes>
                            <include>com/willcard/auth/domain/service/**</include>
                            <include>com/willcard/auth/adapter/**</include>
                        </includes>
                    </configuration>
                </execution>
                <execution>
                    <id>check</id>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <includes>
                            <include>com/willcard/auth/domain/service/**</include>
                            <include>com/willcard/auth/adapter/**</include>
                        </includes>
                        <rules>
                            <rule>
                                <element>BUNDLE</element>
                                <limits>
                                    <limit>
                                        <counter>INSTRUCTION</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum>
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 5.2 BFF 模組

BFF 與 Service 模組的差異在於：`UseCaseImpl` 位於 `application/usecase/` 下，而非 `domain/`。`includes` 與 `sonar.coverage.exclusions` 必須反映此差異。

```xml
<properties>
    <sonar.coverage.exclusions>
        **/com/willcard/bff/application/api/**,
        **/com/willcard/bff/bootstrap/**,
        **/com/willcard/bff/domain/model/**,
        **/com/willcard/bff/domain/vo/**,
        **/com/willcard/bff/domain/dto/**,
        **/com/willcard/bff/domain/port/**,
        **/com/willcard/common/**,
        **/BffApplication.java
    </sonar.coverage.exclusions>
</properties>

<!-- JaCoCo plugin 的 includes 設定（BFF） -->
<includes>
    <include>com/willcard/bff/application/usecase/**</include>
    <include>com/willcard/bff/domain/service/**</include>
    <include>com/willcard/bff/adapter/**</include>
</includes>
```
