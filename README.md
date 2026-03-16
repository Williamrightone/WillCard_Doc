# WillCard

> An Electronic Payment Wallet System

<div align="center">

**Language / 語言**

[**English**](README.md) | [繁體中文](i18n/zh-TW/README.md)

</div>

WillCard is a **microservices-based** electronic payment wallet system built on **Java Spring Boot**, inspired by the core design principles of Taiwan Banking, Credit Card and e-payment. It integrates three primary capabilities: credit card transaction authorization, e-wallet top-up, and points rewards.

The system settles exclusively in New Taiwan Dollar (TWD). All foreign currency transactions are converted to TWD at the time of the transaction using a locked exchange rate. No foreign currency positions are held in user accounts.

## Core Features

- **Virtual Credit Card (WillCard)**: Issuing virtual cards with support for credit card authorization and OTP two-factor verification
- **E-Wallet**: TWD points balance management, supporting top-up via credit card and wallet balance payments
- **Points Rewards**: Consumer spending rewards calculated in a T+N batch job; points become available after the settlement date
- **Foreign Currency Support**: Real-time exchange rate locking at transaction time; FX conversion to TWD before debit
- **Dual Reconciliation**: T+0 real-time reconciliation + T+1 batch settlement with full audit trail

## Technology Stack

| Layer | Technology |
| --- | --- |
| Backend Framework | Java 17 + Spring Boot 3.4.5 |
| Architecture Pattern | Microservices + BFF + Saga Orchestration |
| Container Platform | Kubernetes (kubeadm) |
| Primary Database | MySQL 8 + Spring Data JPA (Hibernate) |
| Cache | Redis 7 |
| Message Queue | Kafka (event bus) + RabbitMQ (notification buffering) |
| Batch Scheduling | XXL-Job + Spring Batch |
| Observability | Jaeger + Prometheus + Grafana + EFK Stack |
| Security | Spring Security (BFF only) + JWT + AES-256 (card encryption) |

## Table of Contents

