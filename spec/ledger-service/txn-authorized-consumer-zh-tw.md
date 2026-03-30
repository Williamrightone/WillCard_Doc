# 交易授權事件消費者 — ledger-service Spec

> **服務：** ledger-service
> **元件類型：** Kafka Consumer
> **Topic：** `txn.card.authorized`
> **最後更新：** 2026-03

---

## 1. 概述

消費 `txn.card.authorized` 事件，在 `ledger_db` 建立複式記帳分錄。每筆授權交易依情境（台幣 / 外幣 / 點數折抵）產生一或多對借貸分錄。

透過 `uk_je_idempotency_key`（`{txnId}_{entrySeq}`）保障冪等性——重複的 Kafka 事件靜默忽略。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 事件來源（ORCH） | [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md) |
| 資料庫 | [ledger_db.md](../../database/ledger_db.md) |

---

## 3. 訊息消費者定義

### Kafka Binding

| 欄位 | 值 |
|------|---|
| Topic | `txn.card.authorized` |
| Consumer group | `ledger-service` |
| Consumer class | `ledger.service.consumer.TxnAuthorizedConsumer` |

### 消費的事件欄位

| 欄位 | 型別 | 用途 |
|------|------|------|
| txnId | String | 冪等鍵前綴（`{txnId}_1`、`{txnId}_2` ...） |
| cardType | String | 查詢 `fx_rate_snapshot` 的鍵 |
| txnTwdBaseAmount | Long | 所有分錄的基礎金額（台幣分） |
| isOverseas | Boolean | 決定是否建立外幣手續費分錄 |
| pointsToUse | Long | 決定是否建立點數折抵對沖分錄 |

---

## 4. 科目代碼

| 代碼 | 科目名稱 | 正常餘額 | 說明 |
|------|---------|---------|------|
| 1001 | 應收帳款 | 借方 | 持卡人欠 WillCard 的金額 |
| 2001 | 應付帳款 | 貸方 | WillCard 欠商戶的金額 |
| 3001 | 外幣手續費收入 | 貸方 | 跨境交易手續費（僅外幣交易） |
| 4001 | 點數負債 | 貸方 | 持卡人折抵的點數 |

---

## 5. 分錄情境

### 情境 A — 台幣、無點數折抵

**條件：** `isOverseas = false` 且 `pointsToUse = 0`

| 序號 | 科目 | 借方 | 貸方 |
|------|------|------|------|
| 1 | 1001 應收帳款 | `txnTwdBaseAmount` | — |
| 2 | 2001 應付帳款 | — | `txnTwdBaseAmount` |

---

### 情境 B — 台幣、有點數折抵

**條件：** `isOverseas = false` 且 `pointsToUse > 0`

| 序號 | 科目 | 借方 | 貸方 | 金額 |
|------|------|------|------|------|
| 1 | 1001 應收帳款 | ✓ | — | `txnTwdBaseAmount` |
| 2 | 2001 應付帳款 | — | ✓ | `txnTwdBaseAmount` |
| 3 | 4001 點數負債 | — | ✓ | `pointsToUse` |
| 4 | 1001 應收帳款 | — | ✓ | `pointsToUse`（對沖：持卡人現金負擔減少） |

> 分錄 3–4：點數折抵降低持卡人的淨現金義務，同時建立 WillCard 代持卡人支付的點數負債。

---

### 情境 C — 外幣、無點數折抵

**條件：** `isOverseas = true` 且 `pointsToUse = 0`

令 `fxFee = floor(txnTwdBaseAmount × fx_fee_rate)`（`fx_fee_rate` 從 `fx_rate_snapshot` 取得）

| 序號 | 科目 | 借方 | 貸方 | 金額 |
|------|------|------|------|------|
| 1 | 1001 應收帳款 | ✓ | — | `txnTwdBaseAmount` |
| 2 | 2001 應付帳款 | — | ✓ | `txnTwdBaseAmount - fxFee` |
| 3 | 3001 外幣手續費收入 | — | ✓ | `fxFee` |

---

### 情境 D — 外幣、有點數折抵

**條件：** `isOverseas = true` 且 `pointsToUse > 0`

令 `fxFee = floor(txnTwdBaseAmount × fx_fee_rate)`

| 序號 | 科目 | 借方 | 貸方 | 金額 |
|------|------|------|------|------|
| 1 | 1001 應收帳款 | ✓ | — | `txnTwdBaseAmount` |
| 2 | 2001 應付帳款 | — | ✓ | `txnTwdBaseAmount - fxFee` |
| 3 | 3001 外幣手續費收入 | — | ✓ | `fxFee` |
| 4 | 4001 點數負債 | — | ✓ | `pointsToUse` |
| 5 | 1001 應收帳款 | — | ✓ | `pointsToUse` |

---

## 6. Service UseCase Flow

**UseCase：** `TxnAuthorizedConsumer.consume(TxnCardAuthorizedEvent)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DB READ]` | SELECT `journal_entry` WHERE `txn_id = event.txnId` LIMIT 1；存在 → **ACK**（已處理） |
| 2 | `[DOMAIN]` | 依 `isOverseas` 與 `pointsToUse` 判斷情境（A / B / C / D） |
| 3 | `[DB READ]` | 若 `isOverseas = true`：SELECT `fx_rate_snapshot` WHERE `card_type = event.cardType`；計算 `fxFee = floor(txnTwdBaseAmount × fx_fee_rate)` |
| 4 | `[DOMAIN]` | 建立分錄清單：為每列產生 Snowflake `entry_id` 並指派 `entry_seq`；設定 `idempotency_key = "{txnId}_{entrySeq}"` |
| 5 | `[DB WRITE]` | Batch INSERT `journal_entry`；重複 `idempotency_key` → `ON DUPLICATE KEY IGNORE`（安全重入） |
| 6 | `[RETURN]` | **ACK** |

> **原子性：** 同一 `txnId` 的所有分錄在單一資料庫 Transaction 內批次插入。全部成功或全部 rollback。

---

## 7. 資料庫

**讀取表：**

| 表 | 條件 |
|----|------|
| `journal_entry` | 冪等查詢：`txn_id = ?` LIMIT 1 |
| `fx_rate_snapshot` | 費率查詢：`card_type = event.cardType`（僅外幣交易） |

**寫入表：**

| 表 | 操作 | 說明 |
|----|------|------|
| `journal_entry` | Batch INSERT | 所有情境分錄，單一 Transaction |

---

## 8. 錯誤處理

| 情況 | 行為 |
|------|------|
| `txnId` 已存在分錄 | 立即 ACK（冪等跳過） |
| `fx_rate_snapshot` 找不到對應 `cardType`（外幣） | 記錄 ERROR，NACK 進 DLQ |
| DB 寫入失敗 | NACK 進 DLQ（非業務性失敗；需調查） |

---

## 9. Changelog

### v1.0 — 2026-03 — 初始版本
