# 專案架構指南
 
本文件定義 WillCard Backend 專案的架構規範與開發慣例。
 
系統基於 **Clean Architecture** 與 **Domain-Driven Design（DDD）** 原則建構，每個服務皆採用嚴格的關注點分離，並維持從基礎設施層向領域核心層的單向依賴流向。
 
本指南採用由上而下的方式組織——從整體專案佈局與模組結構出發，逐步深入至套件組織與服務間依賴關係，最後涵蓋各個物件型別、命名慣例與設計約束。
 
---
 
## 1. 專案結構
 
### 1.1 Maven 多模組總覽
 
```
willcard-parent
├── bff
├── card-service
├── card-contract
├── wallet-service
├── wallet-contract
├── points-service
├── points-contract
├── fx-service
├── fx-contract
├── transaction-service
├── transaction-contract
├── ...
└── pom.xml
```
 
### 1.2 Contract 模組原則
 
- 純 POJO — 不得引入任何 framework 依賴（不含 Spring、JPA 等）
- 僅允許的依賴：`lombok`、`jackson-databind`、`spring-boot-starter-validation`
- 僅包含 Rq / Rs 及 Event schema 定義 — 不含業務邏輯
- 版本管理必須嚴格；破壞性變更會影響所有消費方
- 所有呼叫某服務的模組，必須引入對應的 contract
- 不得引入任何內部 common 模組
 
### 1.3 Common 模組
 
三個共用函式庫模組，皆為純 Library，不含 Spring Boot 入口點，不得作為獨立應用程式運行。
 
#### 依賴關係
 
```
      common-shared
    ↑               ↑
common-biz       common-bff
    ↑               ↑
*-service           bff
```
 
#### common-shared
 
所有服務模組的基礎設施共用層，提供全專案通用的工具與基礎 Spring 配置。
 
| 內容 | 說明 |
|---|---|
| Exception 基底類別 | `AbstractServiceException`、`BaseErrorType`、`IErrorLevel`、`ModelType` |
| 通用工具類 | 日期、字串、數字等純工具類 |
| Spring 基礎配置 | Web、Validation、Redis、AMQP、AOP 等 starter |
| 可觀測性 | Actuator、Micrometer Tracing、OpenTelemetry、Prometheus |
| 日誌 | Logstash Logback Encoder |
 
#### common-biz
 
Service 層的共用業務基礎設施，僅供 `*-service` 模組引用，`bff` 不得引用。
 
| 內容 | 說明 |
|---|---|
| JPA 基礎設定 | `BaseEntity`、Audit 欄位（`createdAt`、`updatedAt`） |
| 業務性工具類 | 金融計算、幣別換算等具有業務語意的工具 |
| 共用異常處理器 | `BizExceptionHandler`（`@ControllerAdvice`） |
 
#### common-bff
 
BFF 層的共用基礎設施，僅供 `bff` 模組引用，`*-service` 模組不得引用。
 
| 內容 | 說明 |
|---|---|
| Response 封裝 | 統一的 API 回應結構（`ApiResponse`） |
| 分頁結構 | 共用分頁 Rq / Rs 定義 |
| 共用異常處理器 | `BffExceptionHandler`（`@ControllerAdvice`） |
 
#### 使用邊界規則
 
- `*-contract` 不得引入任何內部 common 模組
- `*-service` 只能引用 `common-biz`，不得引用 `common-bff`
- `bff` 只能引用 `common-bff`，不得引用 `common-biz`
 
---
 
## 2. 物件命名慣例
 
### 2.1 命名對照表
 
| 類型 | 後綴 | 範例 | 所在層 |
|---|---|---|---|
| API 請求 | `Rq` | `LoginRq` | `application.api.dto` / contract |
| API 回應 | `Rs` | `LoginRs` | `application.api.dto` / contract |
| JPA 實體 | `Entity` | `UserEntity` | `adapter.persistence` |
| 領域實體 | `Model` | `UserCardModel` | `domain.model` |
| 值物件 | `Vo` | `MoneyVo` | `domain.vo` |
| 資料傳輸物件 | `Dto` | `AuthByPasswdDto` | `domain.dto` |
 
### 2.2 類型定義
 
**Rq / Rs**
- API 請求與回應物件
- BFF 的 Rq/Rs 由 BFF 模組獨立擁有，定義於 `application.api.dto`
- 其他所有服務的 Rq/Rs 定義於對應的 contract 模組
 
**JPA 實體（後綴 `Entity`）**
- 直接對應資料庫表的 JPA 物件
- 僅在 `adapter.persistence` 內使用
- 不得作為 Rs 回傳給前端或其他服務
 
**領域實體（後綴 `Model`）**
- 具有唯一 ID 與生命週期的領域物件
- 定義於 `domain.model`
- 範例：`UserCardModel`、`WalletModel`
 
**值物件（後綴 `Vo`）**
- 無 ID、不可變，以值定義相等性
- 定義於 `domain.vo`
- 範例：`MoneyVo`（金額 + 幣別）、`CardNumberVo`
 
**資料傳輸物件（後綴 `Dto`）**
- Service 方法的內部產出物，通常以方法命名
- 定義於 `domain.dto`
- 由 UseCase 將多個 Dto 組裝成 Rs
- 範例：`AuthByPasswdDto`
 
### 2.3 物件流程範例
 
```
LoginRq
  → Controller
  → LoginUseCase
  → AuthService.authByPasswd()  → AuthByPasswdDto
  → TokenService.generate()     → TokenDto
  → 組裝成 LoginRs
  ← 由 Controller 回傳
```
 
---
 
## 3. 套件結構
 
### 3.1 BFF 套件結構
 
```
bff
├── application
│     ├── api
│     │     └── dto                  # Rq / Rs（BFF 專屬）
│     ├── controller                 # Controller
│     ├── usecase                    # Use Case 介面
│     ├── usecase.impl               # Use Case 實作
│     └── usecase.assembler          # 大型 Use Case 的 Assembler
├── domain
│     ├── model                      # 領域實體（UserCardModel...）
│     ├── vo                         # 值物件（MoneyVo...）
│     ├── dto                        # 服務內部產出物（AuthByPasswdDto...）
│     ├── port                       # 外部依賴的抽象介面
│     ├── service                    # Domain Service 介面
│     └── service.impl               # Domain Service 實作（選擇性）
├── adapter
│     ├── persistence                # JPA RepositoryImpl、Entity
│     ├── client                     # FeignClient 實作 Port
│     └── messaging                  # Kafka / RabbitMQ Publisher 實作 Port
└── bootstrap                        # Spring Boot 入口點、ControllerAdvice
```
 
### 3.2 Domain Service 套件結構（以 fx-service 為例）
 
```
fx-service
├── domain
│     ├── model
│     ├── vo
│     ├── dto
│     ├── port
│     ├── service
│     └── service.impl
├── adapter
│     ├── persistence
│     ├── client
│     └── messaging
└── bootstrap
 
fx-contract                          # 獨立 Maven 模組 — 純 POJO
└── dto
      ├── FxRateRq
      ├── FxRateRs
      ├── FxLockRateRq
      └── FxLockRateRs
```
 
---
 
## 4. 設計原則與約束
 
### 4.0 核心設計前提
 
本章所有規則，須由以下各元件共同遵守。
 
#### 4.0.1 Controller 與 Use Case 命名
 
本專案採用 Use Case 導向的 API 設計，每支 API 對應一個明確的 Use Case。
 
命名規則：
- API 端點、Controller 方法、Use Case 方法的語意必須一致
- 命名以動詞為主，代表清晰的使用者或系統行為
- 一支 API 對應一個 Use Case，不允許合併多個行為
 
範例：
 
```
POST /auth/guest/start
  → AuthController.start()
  → GenerateGuestTokenUseCase.start()
```
 
此規則的目的：
- 建立 API 與業務行為的一對一關係
- 降低系統導覽與維護時的認知負擔
- 確保 Controller 保持為 Thin Adapter
 
#### 4.0.2 Rq / Rs 套件策略
 
Rq / Rs 是對外或跨模組通訊的資料契約，其位置依模組角色而定。
 
| 模組類型 | Rq / Rs 位置 | 說明 |
|---|---|---|
| BFF / Facade | `application.api.dto` | 由 Facade 模組獨立擁有 |
| Domain Service / Core | `{service}-contract` | 跨模組 / 跨服務的共用資料契約 |
 
禁止：Facade 專屬的 Rq/Rs 不得被其他服務引入。
 
#### 4.0.3 選擇性實作策略
 
並非每個 Service、Repository 或 Adapter 都需要對應的 Impl 類別。
 
判斷標準：
 
| 情境 | 需要介面？ |
|---|---|
| 同一介面有多個實作（例如 Mock vs Real） | 是 |
| 跨模組依賴邊界 | 是 |
| 單一實作且不預期替換 | 否 — 直接使用具體類別 |
 
**例外：Use Case 無論實作數量，一律必須定義介面。**
 
---
 
### 4.1 Controller
 
Controller 是系統的外部入口，負責接收 API 請求並委派給對應的 Use Case。
 
**職責**
- 接收並解析 Request（Rq）
- 執行基本參數驗證（格式、必填欄位）
- 呼叫對應的 Use Case
- 回傳 Response（Rs）
 
**約束**
- 不得包含任何業務邏輯
- 不得直接呼叫 Service、Repository 或 Adapter
- 不得處理交易或流程判斷
 
Controller 必須保持為 Thin Adapter。
 
---
 
### 4.2 Use Case
 
Use Case 代表一個明確的使用者或系統行為，是應用層的核心。
 
**職責**
- 定義單一業務行為的執行介面
- 協調流程順序與邏輯分支
- 呼叫必要的 Service 或 Adapter
 
**規則**
- 必須為介面
- 以動詞為命名主軸
- 一個 Use Case 對應一個 API 行為
 
---
 
### 4.3 Use Case 實作
 
UseCaseImpl 是 Use Case 的具體實作。
 
**職責**
- 實作應用流程與邏輯編排
- 呼叫 Service、Repository 或 Adapter
- 將多個 Dto 組裝成 Rs 並回傳
 
**約束**
- 不得包含基礎設施實作細節
- 不得直接操作 Framework API
- 不得處理與業務邏輯無關的格式轉換（例如 JSON 序列化）
- 屬於業務邏輯一部分的物件組裝（Dto → Rs）由 UseCaseImpl 負責，允許執行
 
UseCaseImpl 是應用流程的唯一擁有者。
 
---
 
### 4.4 Assembler
 
Assembler 用於拆解過大的 UseCase，負責協調多個 Service 呼叫並將結果組裝成 Rs。
 
**位置：** `application.usecase.assembler`
 
**職責**
- 協調多個 Domain Service 的呼叫順序
- 合併多個 Dto 並組裝成 Rs
 
**使用時機**
- 當一個 UseCase 協調超過三個 Service 時，考慮抽取 Assembler
 
**約束**
- 不得包含業務決策邏輯 — 僅負責編排與組裝
- 不得直接存取 Repository 或 Adapter
 
---
 
### 4.5 Service
 
Service 承載可重複使用的業務邏輯與子流程。
 
**職責**
- 封裝業務規則
- 提供可跨多個 Use Case 共用的行為
 
**規則**
- 不得依賴 Controller 或 Use Case
- 可依賴 Repository 或 Adapter 的 Port 介面
- 不得協調跨 Use Case 的流程
 
---
 
### 4.6 Repository
 
Repository 定義資料存取的抽象介面。
 
**職責**
- 定義資料存取方法
- 隱藏資料來源與實作細節
- 提供以領域為中心的存取語意
 
**規則**
- 必須為介面
- 回傳 Domain Model — 不得暴露 JPA Entity
- 不得暴露 ORM / SQL 細節
 
---
 
### 4.7 DataGripHelper
 
DataGripHelper 處理需要合併多個 Repository 資料的情境，封裝跨表查詢與組裝邏輯。
 
**位置：** `adapter.persistence`
 
**職責**
- 注入多個 JPA Repository
- 合併多張表的查詢結果
- 將結果轉換為 Domain Model 並回傳給 Domain Service
 
**規則**
- 實作定義於 `domain.port` 的對應 Port 介面
- Domain Service 只依賴 Port 介面 — 不得直接感知 DataGripHelper
- 回傳類型必須為 Domain Model — 不得回傳 JPA Entity 或 Vo
 
**範例**
 
```
domain.port
  └── CardWalletPort（介面）→ 回傳 CardWalletModel
 
adapter.persistence
  └── CardWalletDataGripHelper implements CardWalletPort
        ├── 注入 CardJpaRepository
        ├── 注入 WalletJpaRepository
        ├── 組裝資料
        └── 轉換並回傳 CardWalletModel
```
 
---
 
### 4.8 Port
 
Port 定義與外部系統或基礎設施互動的抽象介面。
 
**職責**
- 抽象化 HTTP、RPC、MQ、Cache 等外部依賴
- 隔離第三方系統對核心業務層的影響
 
**規則**
- 必須為介面，定義於 `domain.port`
- 由 Adapter 實作
 
---
 
### 4.9 Repository 實作
 
RepositoryImpl 是 Repository 的具體實作。
 
**職責**
- 實作 ORM、SQL 與交易技術細節
- 將 JPA Entity 轉換為 Domain Model 後回傳
 
**約束**
- 不得包含業務邏輯
- 不得被 Controller 或 Use Case 直接依賴
 
---
 
### 4.10 Adapter
 
Adapter 是 Port 的具體實作。
 
**職責**
- 實作與外部系統的實際互動
- 處理通訊、序列化與錯誤轉換
 
---
 
### 4.11 資料模型
 
#### 命名慣例
 
| 類型 | 後綴 | 範例 | 位置 |
|---|---|---|---|
| API 請求 | `Rq` | `LoginRq` | `application.api.dto` / contract |
| API 回應 | `Rs` | `LoginRs` | `application.api.dto` / contract |
| JPA 實體 | `Entity` | `UserEntity` | `adapter.persistence` |
| 領域實體 | `Model` | `UserCardModel` | `domain.model` |
| 值物件 | `Vo` | `MoneyVo` | `domain.vo` |
| 資料傳輸物件 | `Dto` | `AuthByPasswdDto` | `domain.dto` |
 
#### 禁止事項
 
- `Entity` 不得作為 Rs 回傳給前端或其他服務
- `Entity` 不得出現於 `adapter.persistence` 以外的地方
- `Model` 不得直接對應表結構（不得包含 JPA 註解）
 
---
 
## 5. 依賴方向總覽
 
### 5.1 Clean Architecture 依賴規則
 
```
xxx-service
  → application（Controller / UseCase / Assembler）
  → domain（Service / Port 介面 / Model / Vo / Dto）
  ← adapter（實作 Port / Repository）
```
 
- 依賴方向一律朝內
- `domain` 層不依賴任何其他模組
- `adapter` 依賴 `domain`（以實作 Port）
 
### 5.2 BFF 完整依賴流程
 
```
Rq → Controller → UseCase → Assembler（選用）
                           → Domain Service A → Dto-A ┐
                           → Domain Service B → Dto-B ┤→ 組裝成 Rs
                 UseCase ←────────────────────────────┘
Controller ← Rs
```
 
### 5.3 Port / Adapter 依賴反轉
 
```
Domain Service → Port（介面）
                      ↑
          Adapter 實作 Port（依賴反轉）
          ├── FeignClient      （HTTP 呼叫）
          ├── Kafka Publisher  （MQ）
          ├── RepositoryImpl   （JPA）
          └── DataGripHelper   （跨表組裝）
```

下一章: [資料管理](/guideline/4-data-mgmt-zh-tw.md)
