# 解凍卡片 — card-service Spec

> **服務：** card-service
> **Contract 模組：** card-contract
> **呼叫方：** bff → [mobile-bff/card-unfreeze-zh-tw.md](../mobile-bff/card-unfreeze-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 BFF 透過 Feign 轉發的解凍請求。
驗證卡片存在、歸屬於請求用戶，且目前為 `FROZEN` 狀態。
將卡片狀態從 `FROZEN` 轉換為 `ACTIVE`。

---

## 2. API Definition — card-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/unfreeze` |
| 實際接收路徑 | `/card/unfreeze` |
| 呼叫方 | BFF（`CardFeignClient`） |

### Request（`card.contract.dto.CardUnfreezeRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardId | String | ✅ | @NotBlank | 目標卡片的 Snowflake ID |

### Response（`card.contract.dto.CardStatusRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| cardId | String | 卡片 Snowflake ID（String） |
| status | String | `ACTIVE` |

---

## 3. Service UseCase Flow

**UseCase：** `CardUnfreezeUseCase.unfreeze(CardUnfreezeRq)`（card-service）
**UseCase Impl：** `CardUnfreezeUseCaseImpl`
**Repository：** `CardJpaRepository`（Spring Data JPA）

| 步驟 | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DB READ]` | 以 `card_id` 查詢 `card`；不存在 → 拋出 `CARD_NOT_FOUND`（CA00001） | `CardQueryService.findById()` |
| 2 | `[DOMAIN]` | 驗證 `card.user_id == X-Member-Id`；不符 → 拋出 `CARD_NOT_FOUND`（CA00001） | `CardAuthorizationService.assertOwnership()` |
| 3 | `[DOMAIN]` | 驗證 `card.status == FROZEN`；否則 → 拋出 `CARD_FROZEN`（CA00004） | `CardStateService.assertFrozen()` |
| 4 | `[DB WRITE]` | 更新 `card.status = 'ACTIVE'`，`updated_at = now()` | `CardJpaRepository.save()` |
| 5 | `[RETURN]` | 回傳 `CardStatusRs`（cardId、status = `ACTIVE`） | — |

> **注意：** CA00004（`CARD_FROZEN`）在此流程中被用於表示「卡片非 FROZEN 狀態（已是 ACTIVE 或已取消）」，BFF 將其映射為 `CARD_NOT_FROZEN`。

---

## 4. 資料庫

### MySQL

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `card` | READ | 以 `card_id` 查詢卡片 |
| `card` | WRITE | 更新 `status` 為 `ACTIVE` |

### Redis

> 本端點不進行任何 Redis 操作。

### 資料表 Schema

#### card（本操作相關欄位）

| 欄位 | 型別 | 說明 |
|------|------|------|
| card_id | BIGINT | PK |
| user_id | BIGINT | 歸屬驗證 |
| status | VARCHAR(20) | 驗證（須為 FROZEN）；更新為 ACTIVE |
| updated_at | DATETIME(3) | 由 BaseTimeEntity 更新 |

> 完整 card 資料表 Schema：請參閱 [card-apply-zh-tw.md](./card-apply-zh-tw.md#資料表-schema)。

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 404 | CA00001 | CARD_NOT_FOUND | 卡片不存在，或 `user_id` 不符合 `X-Member-Id` |
| 409 | CA00004 | CARD_FROZEN | 卡片狀態非 `FROZEN`（已是 ACTIVE 或已取消） |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
