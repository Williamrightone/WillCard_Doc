# Points Settlement Job — Wallet Service Spec

> **Service:** wallet-service
> **Component Type:** Scheduled Batch Job
> **Last updated:** 2026-04

---

## 1. Overview

Nightly batch job that settles all `PENDING` point reward batches created on the previous business day (T day), transitioning them to `CONFIRMED` and crediting `wallet_account.available_balance`.

Points remain `PENDING` on transaction day T and become spendable on T+1, after this job completes. Each batch is settled in its own DB transaction to isolate failures.

---

## 2. Job Definition

| Field | Value |
|-------|-------|
| Class | `wallet.service.job.PointsSettlementJob` |
| Trigger | `@Scheduled(cron = "0 0 2 * * *", zone = "Asia/Taipei")` |
| Run window | Daily at 02:00 CST |
| Scope | All `point_reward_batch` rows where `status = 'PENDING'` AND `DATE(created_at) < CURRENT_DATE` (Asia/Taipei) |
| Concurrency guard | Redis distributed lock `job:lock:points-settlement` |

> **Scope note:** The query targets `created_at < CURRENT_DATE` (not strictly yesterday) to self-heal any batches missed by a failed or skipped prior run.

---

## 3. Job UseCase Flow

**Job:** `PointsSettlementJob.run()`

| Step | Type | Description | Failure Handling |
|------|------|-------------|-----------------|
| 1 | `[REDIS WRITE]` | SETNX `job:lock:points-settlement`, TTL 10 min | Lock held → log WARN, abort this run |
| 2 | `[DB READ]` | SELECT all `point_reward_batch` WHERE `status = 'PENDING'` AND `DATE(created_at, 'Asia/Taipei') < CURRENT_DATE`; collect as batch list | DB error → release lock, log ERROR, abort |
| 3 | `[DOMAIN]` | For each batch: open individual DB transaction (isolation: READ COMMITTED) | — |
| 4 | `[DB WRITE]` | UPDATE `point_reward_batch` SET `status = 'CONFIRMED'`, `updated_at = NOW()` WHERE `batch_id = :batchId` AND `status = 'PENDING'` | 0 rows updated → batch already settled by concurrent process; skip, count as SKIPPED |
| 5 | `[DB WRITE]` | UPDATE `wallet_account` SET `available_balance = available_balance + :issuedAmount`, `updated_at = NOW()` WHERE `user_id = :userId` | DB error → rollback this batch's transaction, log ERROR with `batch_id`, count as FAILED; continue to next batch |
| 6 | `[DOMAIN]` | Commit transaction; count as SETTLED | — |
| 7 | `[REDIS WRITE]` | Release `job:lock:points-settlement` | — |
| 8 | `[RETURN]` | Log settlement summary: `{ totalFound, settled, skipped, failed }` | If `failed > 0` → log ERROR-level alert for manual review |

> **Steps 4–6 execute in a single DB transaction per batch.** A failure in Step 5 rolls back the `CONFIRMED` status update (Step 4) so the batch remains `PENDING` and is eligible for the next run.
>
> **Idempotency:** The `WHERE status = 'PENDING'` guard in Step 4 prevents double-settlement if the job is re-run manually. Zero rows updated = already processed; job moves on without error.
>
> **`wallet_account.available_balance` semantics:** This column is a denormalized cache (see `wallet_db.md`). The authoritative balance is the sum of `point_reward_batch.remaining_balance` across `CONFIRMED` non-expired batches. The UPDATE in Step 5 keeps the cache in sync atomically within the same transaction as the batch confirmation.

---

## 4. Database

### Tables Read

| Table | Operation | Condition |
|-------|-----------|-----------|
| `point_reward_batch` | SELECT | `status = 'PENDING'` AND `DATE(created_at) < CURRENT_DATE` |

### Tables Written

| Table | Operation | Description |
|-------|-----------|-------------|
| `point_reward_batch` | UPDATE | `status: PENDING → CONFIRMED` |
| `wallet_account` | UPDATE | `available_balance += issued_amount` |

### Redis Keys

| Key | Operation | TTL | Description |
|-----|-----------|-----|-------------|
| `job:lock:points-settlement` | SETNX / DEL | 10 min | Distributed run-once lock |

---

## 5. Error Handling

| Condition | Behavior |
|-----------|----------|
| Lock held on startup | Skip this run; log WARN; next scheduled run will pick up |
| Batch already `CONFIRMED` (Step 4 returns 0 rows) | Count as SKIPPED; continue |
| DB error on UPDATE (Step 5) | Rollback this batch; count as FAILED; log ERROR with `batch_id`; continue to next |
| `wallet_account` row missing for `user_id` | Rollback this batch; count as FAILED; log ERROR — indicates data inconsistency; requires manual review |
| `failed > 0` after all batches processed | Log ERROR-level summary; trigger alert via EFK; no automatic retry |

---

## 6. Changelog

### v1.0 — 2026-04 — Initial spec
