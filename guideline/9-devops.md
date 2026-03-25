# DevOps Pipeline

This chapter describes the WillCard DevOps workflow, covering branching strategy, CI Pipeline design, and CD Pipeline design.

---

## 1. Branching Strategy

The project uses a simplified GitHub Flow with the following branch types:

| Branch | Description |
|---|---|
| `main` | Primary branch, always releasable. Direct push is prohibited; merges via PR only. |
| `dev` | Testing branch (may be split into `sit`, `uat` as needed). |
| `feature/{name}` | Feature development branch, created from `main`. |
| `hotfix/{name}` | Emergency fix branch, created from `main`. Same workflow, different naming and priority. |

### Development Workflow

```
main → feature/xxx → (commit & push) → PR to dev → SIT testing
                                                        ↓ passes
                                         PR to main → merge → tag → CD
```

1. Create `feature/{name}` or `hotfix/{name}` from `main`.
2. Develop, commit, and push.
3. Open a PR targeting `dev`; this triggers the CI Pipeline.
4. If CI fails, fix on the same branch and push again.
5. Once CI passes, deploy to dev environment for testing.
6. After testing passes, open a PR targeting `main`; CI runs again.
7. After merge to `main`, create a version tag to trigger the CD Pipeline.

```bash
# Create a feature branch
git checkout main
git checkout -b feature/my-feature

# Push after development
git add .
git commit -m "feat: implement my feature"
git push -u origin feature/my-feature
```

---

## 2. CI Pipeline

### 2.1 Trigger Conditions

| Event | Target Branch | Path Filter | Jobs Triggered |
|---|---|---|---|
| `push` | `main` | `willcard-backend/**` | Build & Analyze |
| `pull_request` opened / synchronize | `main` | `willcard-backend/**` | Build & Analyze |

> CI only triggers when files under `willcard-backend/` change, preventing doc-only or unrelated commits from running unnecessary builds.
> CI must pass before a PR to `main` can be merged (Required Status Check).

### 2.2 Runner Configuration

A Self-hosted Runner is used to prevent source code and credentials from leaving the internal network.

| Item | Value |
|---|---|
| Runner Label | `arm-self` |
| Hardware | Mac Mini M4 (Apple Silicon) |
| OS | macOS |
| Java | OpenJDK 17 via Homebrew |
| JAVA_HOME | `/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home` |
| Maven | System path (`mvn`) |
| Docker | Installed (used by Testcontainers for integration tests) |
| SonarQube Host | `SONAR_HOST_URL` (from GitHub Secrets) |
| Harbor | `harbor.willcard.internal` |

### 2.3 Pipeline Stages

CI uses a single Job design, merging compile, test, coverage gate, and static analysis into one execution:

```
┌────────────────────────────────────────────────────────┐
│  Build & Analyze                                       │
│  (mvn verify + sonar:sonar, with JaCoCo Coverage Gate) │
└────────────────────────────────────────────────────────┘
```

Steps within the Job:

1. **Restore dependency caches** (Maven + SonarQube cache)
2. **Compile + unit/integration tests + JaCoCo Coverage Gate** (`mvn verify`)
3. **SonarQube static analysis** (`sonar:sonar`, same mvn command)

Benefits of combining `mvn verify sonar:sonar`:
- Avoids redundant compilation, saving Runner resources.
- JaCoCo reports are directly available to SonarQube without artifact hand-off.

### 2.4 Caching

`actions/cache@v4` is used to cache dependencies and accelerate subsequent builds.

| Cache Target | Path | Cache Key |
|---|---|---|
| Maven local repository | `~/.m2` | `${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}` |
| SonarQube analysis cache | `~/.sonar/cache` | `${{ runner.os }}-sonar` |

- Maven cache key is derived from all `pom.xml` hashes; any dependency change produces a new key.
- SonarQube cache is keyed by OS to accelerate incremental SonarQube analysis data updates.

### 2.5 Testcontainers

Integration tests use Testcontainers to spin up real DB / Redis containers. The following environment variables are required:

| Environment Variable | Value | Purpose |
|---|---|---|
| `DOCKER_HOST` | `unix:///var/run/docker.sock` | Points to the Docker daemon on the Runner |
| `TESTCONTAINERS_RYUK_DISABLED` | `true` | Disables Ryuk cleanup container to avoid macOS permission issues |

> **Why disable Ryuk?** Ryuk is the Testcontainers daemon responsible for cleaning up leftover containers. On macOS Self-hosted Runners, socket permission issues can cause test failures. Disabling Ryuk relies on the Job's natural teardown for cleanup instead.

### 2.6 SonarQube

- SonarQube analysis is merged with `mvn verify` in a single mvn command; no separate Job is needed.
- `SONAR_TOKEN` and `SONAR_HOST_URL` are stored in GitHub Secrets and never hard-coded in YAML.
- SonarQube project key: `willcard-mobile-bff`.
- `fetch-depth: 0` (full git history) is required for SonarQube blame functionality.
- Quality Gate results are informational for PRs; **a failing Quality Gate does not block the pipeline** (left to Code Reviewer judgment).

### 2.7 Coverage Gate

- `mvn verify` is executed with the JaCoCo Maven Plugin to measure instruction coverage.
- Coverage tracking scope follows the [8-unit-test](8-unit-test.md) guidelines, excluding Controllers, DTOs, Bootstrap classes, etc.
- **Threshold: instruction coverage ≥ 80%** — the pipeline fails and blocks merging if this is not met.

JaCoCo exclusion configuration (centralized in `pom.xml`):

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

The following is the actual `.github/workflows/ci.yml` in use:

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

> To be designed. Expected scope: Docker Image Build → Push to Harbor → Kubernetes Rolling Update.
>
> Anticipated trigger: tag push (version tag) to `main`, targeting deployment to the k8s cluster.

---
