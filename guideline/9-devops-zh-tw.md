# DevOps Pipeline

本章說明 WillCard 的 DevOps 流程，涵蓋分支策略、CI Pipeline 設計與 CD Pipeline 設計。

---

## 1. 分支策略

本專案採簡化的 GitHub Flow，分支共為以下幾種：

| 分支 | 說明 |
|---|---|
| `main` | 主要分支，隨時可上線的版本。禁止直接 push，僅能由 feature/hotfix 分支經 PR 合併。 |
| `dev` | 測試分支（可依需求細分為 `sit`、`uat`）。 |
| `feature/{name}` | 功能開發分支，自 `main` 建立。 |
| `hotfix/{name}` | 緊急修復分支，自 `main` 建立，命名與優先順序有別，流程相同。 |

### 開發流程

```
main → feature/xxx → (commit & push) → PR to dev → SIT 測試
                                                        ↓ 測試通過
                                         PR to main → merge → tag → CD
```

1. 自 `main` 建立 `feature/{name}` 或 `hotfix/{name}`。
2. 開發完成後 commit & push。
3. 建立 PR，目標為 `dev`，觸發 CI Pipeline 進行驗證。
4. 若 CI 失敗，於同一分支修正並重新 push。
5. CI 通過後部署至 dev 環境進行測試。
6. 測試通過後，建立 PR 目標 `main`，再次觸發 CI。
7. Merge 至 `main` 後打版號（tag），觸發 CD Pipeline 部署。

```cmd
# 建立功能分支
git checkout main
git checkout -b feature/my-feature

# 開發完成後推送
git add .
git commit -m "feat: implement my feature"
git push -u origin feature/my-feature
```

---

## 2. CI Pipeline

### 2.1 觸發條件

| 事件 | 目標分支 | Path Filter | 觸發的 Jobs |
|---|---|---|---|
| `push` | `main` | `willcard-backend/**` | Build & Analyze |
| `pull_request` opened / synchronize | `main` | `willcard-backend/**` | Build & Analyze |

> 僅當 `willcard-backend/` 目錄下有變更時才觸發，避免文件或其他子目錄的修改觸發不必要的 CI。
> PR to `main` 的 CI 結果為合併前必要條件（Required Status Check）。

### 2.2 Runner 配置

使用 Self-hosted Runner，避免原始碼與憑證離開內部網路。

| 項目 | 值 |
|---|---|
| Runner Label | `arm-self` |
| Hardware | Mac Mini M4（Apple Silicon） |
| OS | macOS |
| Java | OpenJDK 17 via Homebrew |
| JAVA_HOME | `/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home` |
| Maven | 系統路徑（`mvn`） |
| Docker | 已安裝（供 Testcontainers 整合測試使用） |
| SonarQube Host | `SONAR_HOST_URL`（來自 GitHub Secrets） |
| Harbor | `harbor.willcard.internal` |

### 2.3 Pipeline 階段

CI 採單一 Job 設計，將編譯、測試、覆蓋率門檻與靜態分析合併執行：

```
┌────────────────────────────────────────────────────────┐
│  Build & Analyze                                       │
│  (mvn verify + sonar:sonar，含 JaCoCo Coverage Gate)   │
└────────────────────────────────────────────────────────┘
```

Job 內部步驟：

1. **相依快取還原**（Maven + SonarQube cache）
2. **編譯 + 單元/整合測試 + JaCoCo Coverage Gate**（`mvn verify`）
3. **SonarQube 靜態分析**（`sonar:sonar`，同一 mvn 指令執行）

`mvn verify sonar:sonar` 合併執行的優點：
- 避免重複編譯，節省 Runner 資源。
- JaCoCo report 可直接被 SonarQube 讀取，無需 artifact 傳遞。

### 2.4 Caching

使用 `actions/cache@v4` 快取相依資料，加速後續 build。

| 快取目標 | 路徑 | Cache Key |
|---|---|---|
| Maven local repository | `~/.m2` | `${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}` |
| SonarQube analysis cache | `~/.sonar/cache` | `${{ runner.os }}-sonar` |

- Maven cache key 由所有 `pom.xml` 的 hash 決定，任何依賴變更即產生新 key。
- SonarQube cache 按 OS 區分，加速 SonarQube 分析資料的增量更新。

### 2.5 Testcontainers

整合測試使用 Testcontainers 啟動真實 DB / Redis 容器，需設定以下環境變數：

| 環境變數 | 值 | 說明 |
|---|---|---|
| `DOCKER_HOST` | `unix:///var/run/docker.sock` | 指向 Runner 上的 Docker daemon |
| `TESTCONTAINERS_RYUK_DISABLED` | `true` | 停用 Ryuk 清理容器，避免 macOS 上的權限問題 |

> **為何停用 Ryuk？** Ryuk 是 Testcontainers 用於清理殘留容器的守護程式，在 macOS Self-hosted Runner 上可能因 socket 權限問題導致測試失敗，因此停用並依賴 Job 結束後的自動清理。

### 2.6 SonarQube

- SonarQube 分析與 `mvn verify` 合併於同一 mvn 指令，不另開 Job。
- `SONAR_TOKEN` 與 `SONAR_HOST_URL` 均存放於 GitHub Secrets，不硬編碼於 YAML。
- SonarQube project key：`willcard-mobile-bff`。
- `fetch-depth: 0`（完整 git history）為 SonarQube blame 功能的必要條件。
- Quality Gate 結果作為 PR 的參考資訊；**Quality Gate 失敗不強制阻斷 pipeline**（由 Code Reviewer 判斷）。

### 2.7 Coverage Gate

- 執行 `mvn verify`，由 JaCoCo Maven Plugin 統計 instruction coverage。
- Coverage 追蹤範圍依照 [8-unit-test](8-unit-test-zh-tw.md) 規範，排除 Controller、DTO、Bootstrap 等。
- **門檻：instruction coverage ≥ 80%**，未達門檻則 pipeline 失敗並阻止合併。

JaCoCo 排除設定（在 `pom.xml` 中統一配置）：

```xml
<excludes>
  <exclude>**/application/api/controller/**</exclude>
  <exclude>**/application/api/dto/**</exclude>
  <exclude>**/domain/model/**</exclude>
  <exclude>**/domain/vo/**</exclude>
  <exclude>**/domain/port/**</exclude>
  <exclude>**/common/**</exclude>
  <exclude>**/bootstrap/**</exclude>
</excludes>
```

### 2.8 CI Workflow YAML

以下為實際使用的 `.github/workflows/ci.yml`：

```yaml
name: Backend CI

on:
  push:
    branches: [ main ]
    paths:
      - 'willcard-backend/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'willcard-backend/**'

jobs:
  build-and-test:
    runs-on: arm-self

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Cache SonarQube packages
        uses: actions/cache@v4
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build & Analyze
        env:
          JAVA_HOME: /opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          DOCKER_HOST: unix:///var/run/docker.sock
          TESTCONTAINERS_RYUK_DISABLED: "true"
        run: |
          mvn -f willcard-backend/pom.xml -B verify sonar:sonar \
            -Dsonar.projectKey=willcard-mobile-bff \
            -Dsonar.projectName='willcard-mobile-bff'
```

---

## 3. CD Pipeline

> 待設計。將涵蓋：Docker Image Build → Push to Harbor → Kubernetes Rolling Update。
>
> 預計觸發條件：push tag（版號）至 `main` 後觸發，目標部署至 k8s cluster。

---

下一章: [交易流程規範](/guideline/10-txn-flow-zh-tw.md)
