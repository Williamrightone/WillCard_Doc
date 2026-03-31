# 交易授權事件消費者 — points-service Spec

> **服務：** points-service
> **元件類型：** Kafka Consumer
> **Topic：** `txn.card.authorized`
> **最後更新：** 2026-03

---

## 1. 概述

消費 Transaction Orchestrator 發布的 `txn.card.authorized` 事件。依卡片類型的基礎回饋率計算回饋點數，再呼叫 wallet-service 的內部 credit API 建立 `PENDING` 狀態的點數批次。

所有處理透過 `point_issuance_log` 的 `uk_pil_source_txn_id` 保障冪等性。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 事件來源（ORCH） | [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md) |
| 下游 credit 呼叫 | [wallet-service/points-credit-zh-tw.md](../wallet-service/points-credit-zh-tw.md) |

---

## 3. 訊息消費者定義

### Kafka Binding

| 欄位 | 值 |
|------|---|
| Topic | `txn.card.authorized` |
| Consumer group | `points-service` |
| Consumer class | `points.service.consumer.TxnAuthorizedConsumer` |

### 消費的事件欄位

| 欄位 | 型別 | 用途 |
|------|------|------|
| txnId | String | 冪等鍵（`source_txn_id`）；同時作為 wallet credit 的 `sourceTxnId` |
| userId | Long | credit 呼叫的錢包帳戶持有人 |
| cardType | String | 查詢 `reward_plan` 費率的鍵 |
| txnTwdBaseAmount | Long | 回饋計算基礎（台幣分） |
| isOverseas | Boolean | 決定使用 `overseas_reward_rate` 或 `domestic_reward_rate` |
| txnTimestamp | String | ISO-8601；決定回饋批次到期月份 |
| pointsToUse | Long | 本次折抵點數——**不從回饋計算基礎中扣除**（見說明） |

---

## 4. Service UseCase Flow

**UseCase：** `TxnAuthorizedConsumer.consume(TxnCardAuthorizedEvent)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DB READ]` | SELECT `point_issuance_log` WHERE `source_txn_id = event.txnId`；存在 → **ACK**（已處理） |
| 2 | `[DB READ]` | SELECT `reward_plan` WHERE `card_type = event.cardType` |
| 3 | `[DOMAIN]` | 選取費率：`rate = isOverseas ? overseas_reward_rate : domestic_reward_rate` |
| 4 | `[DOMAIN]` | 計算：`reward_points = floor(event.txnTwdBaseAmount × rate)`；若 `reward_points == 0` → Step 5a |
| 5a | `[DB WRITE]` | （回饋 = 0）INSERT `point_issuance_log { source_txn_id, user_id, issued_amount: 0, status: 'SKIP' }`；**ACK** |
| 5b | `[DOMAIN]` | （回饋 > 0）計算 `expiresAt`：txnTimestamp 所在月份一年後同月最後一刻（例：`2026-03-15` → `2027-03-31T23:59:59.999`） |
| 6 | `[FEIGN]` | `WalletFeignClient.credit(PointsCreditRq { userId, sourceTxnId: txnId, issuedAmount: reward_points, expiresAt })` → `POST /wallet-service/internal/points/credit`；見 [wallet-service/points-credit-zh-tw.md](../wallet-service/points-credit-zh-tw.md) |
| 7 | `[DB WRITE]` | INSERT `point_issuance_log { source_txn_id, user_id, batch_id: response.batchId, issued_amount: reward_points, status: 'SUCCESS' }` |
| 8 | `[RETURN]` | **ACK** |

**credit API 失敗時（Step 6 拋出例外）：**

| 步驟 | 類型 | 說明 |
|------|------|------|
| E1 | `[DB WRITE]` | INSERT `point_issuance_log { source_txn_id, user_id, issued_amount: 0, status: 'FAILED' }` |
| E2 | `[RETURN]` | **ACK**（記錄失敗；不 NACK——業務性失敗不使用 DLQ） |

> **E1 僅處理真正的系統性失敗：** wallet credit API 已改為冪等——重複 `sourceTxnId` 呼叫將回傳 HTTP 200 及既有 `batchId`，不會拋出錯誤。因此，E1 路徑僅在真正的基礎設施故障時觸發（例如：網路逾時、DB 中斷），而非重複投遞。

> **回饋基礎不扣除折抵點數（`pointsToUse`）：** 回饋以完整 `txnTwdBaseAmount` 計算——點數折抵不縮減回饋基礎。這是刻意的業務規則（WillCard 以交易總額給予回饋）。
>
> **冪等守衛（Step 1）：** 相同 `txnId` 的重複投遞一律跳過，無論先前狀態為何。`FAILED` 狀態同樣冪等——重複投遞後靜默跳過（需人工 DLQ 重處理）。

---

## 5. 資料庫

**讀取表：**

| 表 | 用途 |
|----|------|
| `point_issuance_log` | 冪等查詢（Step 1） |
| `reward_plan` | 依 `cardType` 查詢費率（Step 2） |

**寫入表：**

| 表 | 操作 | 條件 |
|----|------|------|
| `point_issuance_log` | INSERT | 必定執行（SKIP、SUCCESS 或 FAILED） |

---

## 6. 回饋計算參考

| 卡片類型 | 國內費率 | 國外費率 |
|---------|---------|---------|
| CLASSIC | 1% | 0% |
| OVERSEAS | 0% | 5% |
| PREMIUM | 2% | 5% |
| INFINITE | 2% | 5% |

> 費率從 `reward_plan` 表讀取。使用 `floor()` 捨棄小數分。

---

## 7. 錯誤處理

| 情況 | 行為 |
|------|------|
| `txnId` 已存在 `point_issuance_log` | 立即 ACK（冪等跳過） |
| `cardType` 在 `reward_plan` 找不到 | 記錄 ERROR，INSERT FAILED，ACK |
| `reward_points == 0` | INSERT SKIP，ACK |
| wallet credit API 失敗 | INSERT FAILED，ACK（不使用 DLQ；需人工審查） |

---

## 8. Changelog

### v1.1 — 2026-03 — 說明 E1 適用範圍（wallet credit 冪等性修正後）
- 新增備注：E1 路徑僅處理真正的系統性失敗；重複 `sourceTxnId` 不再觸發錯誤（wallet credit API 已改為 HTTP 200 冪等回應）。

### v1.0 — 2026-03 — 初始版本
