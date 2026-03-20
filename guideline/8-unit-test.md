# Unit Testing

## 1. Guiding Philosophy

This project is built on Clean Architecture. TDD is not mandated, but JaCoCo enforcement ensures that all business-critical layers are verified before a branch is merged.

Coverage tracking is scoped to layers where business logic lives: domain services, use case implementations, and adapters. Infrastructure glue code — controllers, DTOs, bootstrap configuration — is explicitly excluded from metrics but is still tested where appropriate.

---

## 2. Coverage Scope

| Layer | Package | Coverage Tracked | Notes |
|---|---|---|---|
| Domain Service | `domain/service/**` | **Yes** | Must cover all Spec scenarios |
| UseCase / UseCaseImpl | `application/usecase/**` | **Yes** | BFF only |
| Adapter — Persistence | `adapter/persistence/**` | **Yes** | |
| Adapter — Client / Messaging | `adapter/client/**`, `adapter/messaging/**` | **Yes** | |
| Controller | `application/api/controller/**` | No | Tested, excluded from metrics |
| DTOs / Contracts | `application/api/dto/**` | No | Pure data classes |
| Domain Model / VO / Port | `domain/model/**`, `domain/vo/**`, `domain/port/**` | No | Interfaces and value objects |
| `common/**` utilities | `common/**` | No | Tested, excluded from metrics |
| Bootstrap / Entry point | `bootstrap/**` | No | |

> **Coverage metric:** JaCoCo **instruction coverage** (bytecode instruction level, JaCoCo default).
> The 80% threshold is applied to instruction coverage — not line coverage or branch coverage.
> Instruction coverage is slightly stricter than line coverage and more lenient than branch coverage.

---

## 3. Testing Strategy by Layer

### 3.1 Domain Service

- Use **Mockito** to mock all Port dependencies.
- Every test method must map directly to a scenario defined in the corresponding API Spec.
- Full scenario coverage at this layer reduces the controller test burden.

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

### 3.2 UseCase / UseCaseImpl (BFF only)

- Mock domain service dependencies via their Port interfaces.
- Test orchestration logic: field mapping, conditional branching, error propagation across service calls.
- Domain service scenario coverage is delegated to the domain service tests.

### 3.3 Repository (`adapter.persistence`)

- Use **H2** in-memory database.
- Configure H2 in `MySQL` compatibility mode.
- Test all custom query methods (beyond standard JPA CRUD).

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

- Use **Testcontainers** to spin up real infrastructure containers.
- Verify actual write/read/publish behavior — do not mock the client library.

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

- Use `@SpringBootTest` with `MockMvc`.
- **Required:** one Happy Path test — verify the response body matches the `ApiResponse` contract.
- **Required:** 1–2 error scenario tests — verify that the correct `responseCode` is returned in `ApiResponse`.
- Full business scenario coverage is delegated to the domain service tests.

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

### 3.6 Common Utilities

- All utility classes in `common/**` **must** have tests.
- These tests are **not** counted toward coverage metrics (excluded via `sonar.coverage.exclusions`).

---

## 4. Quality Gate

| Metric | Threshold | Enforcement |
|---|---|---|
| JaCoCo instruction coverage | ≥ 80% | Developer self-check on push; CI/CD pipeline rule once active |
| SonarQube Major Issues | 0 | CI/CD pipeline rule |

Coverage must be ≥ 80% before pushing a branch.

---

## 5. Maven POM Configuration

### 5.1 Service Module (e.g. `auth-service`)

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

### 5.2 BFF Module

BFF differs from service modules: `UseCaseImpl` lives under `application/usecase/`, not `domain/`. The `includes` and `sonar.coverage.exclusions` must reflect this difference.

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

<!-- JaCoCo plugin includes for BFF -->
<includes>
    <include>com/willcard/bff/application/usecase/**</include>
    <include>com/willcard/bff/domain/service/**</include>
    <include>com/willcard/bff/adapter/**</include>
</includes>
```
