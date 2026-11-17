# Expected Decisions — Scenario 09: Concurrent Webhook Deliveries and Worker Crash Recovery

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Targeted Hardening of Approach D (Provider-Managed Billing + Local Anniversary Quota & Lease-Backed Reservation Ledger)**.
- **Why**:
  1. **Race-Safe Webhook Processing & Self-Healing Crash Recovery (No Un-Queued `processing` Claims)**:
     - Replace the broken `SELECT`-then-`INSERT` check with a consistent processing model from `references/payment-integration.md`:
       - **Durable Asynchronous Processing (Primary Model)**: Atomically persist the verified event (`payload_json`, `status = 'pending'`) on `UNIQUE (provider, provider_event_id)` before acknowledging the provider (`202`/`200`). A worker claims the event by setting `status = 'processing'`, `lease_owner = <token>`, `lease_expires_at = now + lease_duration`, and incrementing `attempt_count`. Inside one database transaction, the worker applies the subscription state transition AND updates the event to `status = 'processed'` **only if `lease_owner` still matches and `lease_expires_at > now`** (lease fencing). If a worker crashes after enqueue or lease claim, an internal recovery sweep reclaims rows where `status IN ('pending', 'failed_transient') OR (status = 'processing' AND lease_expires_at <= now)` and completes them from `payload_json` **without relying on another provider retry**.
       - **Synchronous Single-Transaction Alternative (Option A)**: Claim the event (`status = 'processed'`) and apply the subscription state transition inside **one** atomic transaction so any crash or transient failure rolls back the event claim completely, allowing the provider's retry to insert and process the event cleanly.
     - **Never** commit a `status = 'processing'` row outside the business transaction and return `200 OK` on concurrent duplicates without durable payload storage and internal lease-recovery workers.
  2. **Subscription-Anniversary Monthly Quota Reset on Annual Plans**:
     - Do **not** scope the 20-credit counter to `current_period_start .. current_period_end` (12 months).
     - Compute the active monthly sub-period (`quota_period_start`, `quota_period_end`) anchored to `current_period_start + N months` (subscription-anniversary model).
  3. **Lease-Backed Quota Reservation & Abandoned-Job Recovery**:
     - Add `reserved_until TIMESTAMP NOT NULL` (set to `now + 5 minutes`) on `translation_reservations`.
     - **Abandoned-Job Cleanup**: Before checking available quota (or via scheduled sweep), transition any row where `status = 'reserved' AND reserved_until < now` to `status = 'released'`.
     - **Idempotent Commit & Release (`WHERE status = 'reserved'`)**:
       - Commit updates `SET status = 'consumed' WHERE id = $1 AND status = 'reserved'`.
       - If `0` rows are updated, check current status: if already `'consumed'` (duplicate callback), treat as idempotent no-op; if already `'released'` (expired lease), **reject/discard** the delayed worker result so it never consumes an already-released reservation.

## 2. Important Interview Questions
- Confirm the quota reset policy for annual plans (calendar-month vs. subscription-anniversary vs. rolling 30-day window).
- Confirm the maximum expected translation worker duration (`reserved_until` lease timeout) and desired behavior if a delayed worker finishes after lease expiry.

## 3. Mandatory Security & Reliability Tests
- **Concurrent Duplicate Webhook Test**: Fire two concurrent deliveries of the same `provider_event_id` and assert business logic executes **exactly once**.
- **Worker Crash & Lease Expiry Test**: Create a reservation, advance clock past `reserved_until`, run recovery, and verify the credit is restored.
- **Delayed Worker & Duplicate Callback Test**: Verify a delayed worker arriving after lease release cannot transition `'released' -> 'consumed'`, and duplicate completion callbacks consume the credit only once.

## 4. Unacceptable Recommendations (Automatic Fail)
- Keeping a non-atomic `SELECT` before `INSERT` for webhook deduplication.
- Leaving `processing` webhook rows permanently wedged after a transient failure with no recovery path.
- Allowing a delayed worker callback (`UPDATE ... SET status = 'consumed'` without `WHERE status = 'reserved'`) to re-consume an expired/released reservation.
