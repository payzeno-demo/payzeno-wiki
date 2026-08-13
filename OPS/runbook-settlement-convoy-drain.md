---
title: "Runbook: settlement_locked convoy during a sweep"
space: OPS
space_name: "Operations & On-call"
author: Noa Migration
author_username: nmigration
author_email: noa.migration@payzeno.com
created: 2026-05-02
updated: 2026-07-18
labels: [runbook, ledger, reconciliation]
source: confluence-export
page_version: 11
---

# Runbook: settlement_locked convoy during a sweep

> **Operations & On-call** · wiki export · written by Noa Migration · last edited 2026-07-18

**Symptom:** a burst of `409 settlement_locked` from the retry route while a
reconciliation sweep is in progress, and a retryable backlog that looks stuck.

**First thing to internalise: this is not an outage.** The convoy is by construction,
and the construction is deliberate. If you take one action from this page, let it be
*no action*.

## The mechanism (from `app/services/reconciliation/retry.py`)

`RetryScheduler._claim_item` takes the batch advisory lock before the row lock —
unconditional since PAY-2056 removed the ``reconcile_batch_lock_on_retry`` kill
switch. The batch lock is NON-BLOCKING on purpose (`try_acquire_batch_lock`): a sweep
can hold the batch lock for minutes across 5,000 items, and a blocking wait would
park drain workers on pooled connections. When the try-acquire fails, the retry
returns None and the route maps it onto `409 settlement_locked` — and the console
treats that as "already settling", refetches, and shows no toast. So during a sweep,
every attempt in a drain pass fails for that batch. That burst of 409s is the system
doing what ADR 0011 says it should. See
[the decision log](../ENG/decision-log-lock-ordering.md) for why the ordering is
sufficient — and why it depends on READ COMMITTED.

## What to check

1. Is a sweep actually running for the noisy batch? (A `running`
   `reconciliation_run` row. If yes: the items the drain is skipping are being
   settled by the sweep. That is the happy path wearing an error code.)
2. Backlog depth: the `RetryableItemCount` alarm threshold is 500 (see
   [ledger alarms](oncall-ledger-alarms.md)). Below that, do nothing at all.
3. Item-level failures vs lock noise: `409 settlement_locked` is lock noise;
   repeated `retryable` outcomes with error codes are the acquirer, not the convoy.

## What NOT to do

- **Do not scale the drain.** The drain runs on one task by design (see
  [the canary](../ENG/onboarding-ledger-oncall.md)). More tasks burn their passes
  failing the same try-lock: more `settlement_locked` noise, zero extra throughput.
- **Do not shorten the sweep or drain intervals.** They are
  `RECONCILE_SWEEP_INTERVAL_SECONDS     = "900"` and
  `RETRY_DRAIN_INTERVAL_SECONDS = "60"` in `envs/production/services.tf`, and the
  terraform comment on them points back at ADR 0011 for a reason.
- **Do not add a new exclusion mechanism.** ADR 0011: "There is no third mechanism."
- The drain itself is Serial on purpose — the backoff exists to stop hammering a
  degraded acquirer. Concurrency here is not a fix, it is the 2041 incident again.

## The real fix (status)

PAY-2057 — skip items whose batch has a `running` reconciliation run in the
candidate query, instead of discovering the sweep one failed lock at a time. Per the
ADR and per the code comment in `retry.py`: PAY-2057 is the fix and it is not done.
The skip, when it lands, is an optimisation *in front of* the try-acquire, not a
replacement for it — the run-status read races the sweep start, so the lock stays.
Design questions go to Noa; the convoy history is mregression's PR #171 thread.

*(The row-claim query itself is `with_for_update(skip_locked=True)` — if you are
reading the code, that skips OTHER RETRIES, not the sweep. The sweep holds no row
locks. That distinction is the whole incident.)*
