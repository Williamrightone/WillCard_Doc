# WillCard

> An Electronic Payment Wallet System

<div align="center">

**Language / 語言**

[English](../../README.md) | [**繁體中文**](docs/zh-TW/README.md)

</div>

WillCard 是一套基於 Java Spring Boot 微服務架構的電子支付錢包系統，參考台灣業界的網路銀行, 信用卡以及電子支付系統的核心設計理念，整合信用卡交易授權、電子錢包儲值、點數回饋三大功能。

系統以台幣為唯一結算幣別，所有外幣消費於交易當下即時換算為台幣，帳戶內不持有外幣部位。

## 核心功能

- **虛擬信用卡（WillCard）**：發行虛擬卡，支援信用卡授權交易與 OTP 二次驗證
- **電子錢包**：台幣點數餘額管理，支援刷卡儲值與餘額消費
- **點數回饋**：消費 T+N 批次核算回饋點數，發放後可供消費使用
- **外幣支援**：國外消費即時鎖定匯率，換算台幣後扣款，FX 為無狀態計算工具
- **雙向對帳**：T+0 即時對帳 + T+1 批次清算，完整稽核軌跡

## 技術選型

| 層次 | 技術 |
|---|---|
| 後端框架 | Java 17 + Spring Boot 3.4.5 |
| 架構模式 | Microservices + BFF + Saga Orchestration |
| 容器平台 | Kubernetes（kubeadm） |
| 主要資料庫 | MySQL 8 + Spring Data JPA (Hibernate) |
| 快取 | Redis 7 |
| 訊息佇列 | Kafka（事件匯流排）+ RabbitMQ（通知削峰） |
| 批次排程 | XXL-Job + Spring Batch |
| 可觀測性 | Jaeger + Prometheus + Grafana + EFK Stack |
| 安全 | Spring Security（BFF 層）+ JWT + AES-256-GCM（卡片加密） |

## 目錄

1. [系統架構](/guideline/1-system-architecture-zh-tw.md)
2. [基礎設施配置](/guideline/2-infrastructure-zh-tw.md)
3. [專案架構](/guideline/3-project-architecture-structure-zh-tw.md)