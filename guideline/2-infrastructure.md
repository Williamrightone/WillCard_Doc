# Infrastructure Configuration

This chapter describes the infrastructure configuration of WillCard,
covering the physical host specification, VM layout, resource allocation,
and the observability stack.

WillCard's entire service landscape runs on a single Windows Server 2019
physical host using VMware Workstation as the hypervisor.

## Physical Host Specification

| Item | Specification |
| --- | --- |
| OS | Windows Server 2019 |
| CPU | AMD Ryzen 5 3600X (6 cores / 12 threads, 3.8 GHz) |
| RAM | 128 GB |
| SSD | 2 TB |
| Hypervisor | VMWare workstation |

**Allocatable resources** (after Host OS reservation):

| Resource | Allocatable |
|---|---|
| vCPU | 10 vCPU |
| RAM | ~116 GB |
| Disk | ~900 GB |

### VM Layout (6 VMs)

| VM | Role | vCPU | RAM | Disk | Services |
| --- | --- | --- | --- | --- | --- |
| `vm-k8s-master` | k8s control plane | 2 | 8 GB | 60 GB | API Server, etcd, Scheduler, Controller-manager |
| `vm-worker-1` | k8s worker (lightweight) | 2 | 16 GB | 80 GB | BFF, Card svc, Points svc, FX svc, Notification svc |
| `vm-worker-2` | k8s worker (core) | 3 | 32 GB | 80 GB | Txn Orchestrator, Wallet svc, Ledger svc, Reconciliation, Audit log |
| `vm-mw-1` | Message middleware | 2 | 24 GB | 300 GB | Kafka (heap 16 GB), ZooKeeper, RabbitMQ |
| `vm-mw-2` | Cache + DB | 1 | 16 GB | 200 GB | Redis (maxmemory 6 GB), MySQL 8 (InnoDB buffer 8 GB) |
| `vm-observe` | Observability | 1 | 12 GB | 100 GB | Elasticsearch (heap 6 GB), Kibana, Prometheus, Grafana, Jaeger |
| **Total** | | **10 vCPU** | **108 GB** | **820 GB** | |

> Host OS + Hypervisor reserves approximately 20 GB RAM; ~8 GB headroom remains.

### Typology

```mermaid
graph TB
    subgraph Host["Host: Windows Server 2019 (VMware Workstation)"]

        subgraph AdminTools["管理工具 (Host)"]
            WINSCP[WinSCP<br/>SFTP Client]
            DBEAVER[DBeaver<br/>DB GUI]
            REDISINSIGHT[RedisInsight<br/>Redis GUI]
            BROWSER[Browser<br/>管理者]
        end

        subgraph VM_Master["vm-k8s-master (Ubuntu 22.04)"]
            K8S_CP[k8s Control Plane<br/>API Server · etcd<br/>Scheduler · Controller-manager]
        end

        subgraph VM_Worker1["vm-worker-1 (Ubuntu 22.04) — k8s worker"]
            BFF[BFF<br/>JWT · Rate Limit · Idempotency]
            CARD[Card Service<br/>vCard · OTP · combineKey]
            POINTS[Points Service<br/>Earn · Redeem]
            FX[FX Service<br/>Rate · Lock · Calculate]
            NOTIFY[Notification Service]
        end

        subgraph VM_Worker2["vm-worker-2 (Ubuntu 22.04) — k8s worker"]
            ORCH[Transaction Orchestrator<br/>Saga · Compensate]
            WALLET[Wallet Service<br/>Balance · Top-up]
            LEDGER[Ledger Service<br/>Double-entry · Immutable]
            RECON[Reconciliation<br/>T+0 · T+1 Batch]
            AUDIT[Audit Log]
        end

        subgraph VM_MW1["vm-mw-1 (Ubuntu 22.04) — Message Middleware"]
            KAFKA[(Kafka<br/>heap 16 GB)]
            ZK[ZooKeeper]
            RABBITMQ[(RabbitMQ)]
        end

        subgraph VM_MW2["vm-mw-2 (Ubuntu 22.04) — Cache + DB"]
            REDIS[(Redis<br/>maxmemory 6 GB)]
            MYSQL[(MySQL 8<br/>InnoDB buffer 8 GB)]
        end

        subgraph VM_OBS["vm-observe (Ubuntu 22.04) — Observability"]
            ES[(Elasticsearch<br/>heap 6 GB)]
            KIBANA[Kibana]
            PROM[Prometheus]
            GRAFANA[Grafana]
            JAEGER[Jaeger]
        end
    end

    EXT[外部使用者<br/>Mobile / Web / POS]

    %% External access
    EXT -->|HTTPS 443| BFF

    %% BFF → Domain / Core
    BFF -->|X-User-Id, X-User-Role| CARD
    BFF -->|X-User-Id, X-User-Role| WALLET
    BFF -->|X-User-Id, X-User-Role| POINTS
    BFF -->|X-User-Id, X-User-Role| FX
    BFF -->|X-User-Id, X-User-Role| ORCH

    %% Orchestrator → Services
    ORCH --> CARD
    ORCH --> WALLET
    ORCH --> FX
    ORCH --> LEDGER
    ORCH -->|publish| KAFKA

    %% Kafka consumers
    KAFKA --> POINTS
    KAFKA --> LEDGER
    KAFKA --> RECON
    KAFKA --> NOTIFY

    %% Notification
    NOTIFY --> RABBITMQ
    CARD --> RABBITMQ

    %% Middleware
    CARD --- MYSQL
    WALLET --- MYSQL
    LEDGER --- MYSQL
    ORCH --- MYSQL
    WALLET --- REDIS
    CARD --- REDIS

    %% Observability
    PROM -.->|scrape /actuator/prometheus| BFF
    PROM -.->|scrape| CARD
    PROM -.->|scrape| ORCH
    GRAFANA -.->|query| PROM
    GRAFANA -.->|query| ES
    KIBANA -.->|query| ES

    %% Admin tools
    WINSCP -->|SFTP 22| VM_Master
    DBEAVER -->|3306| MYSQL
    REDISINSIGHT -->|6379| REDIS
    BROWSER -->|3000| GRAFANA
    BROWSER -->|16686| JAEGER
```

### VM Operating System

All VMs run **Ubuntu 22.04 LTS**. Fluentd is deployed as a DaemonSet on the two worker nodes and does not consume resources on `vm-observe`.

Next Chapter: [Project Architecture](/guideline/3-project-architecture-structure.md)
