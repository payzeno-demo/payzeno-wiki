---
title: "settlement_locked during reconciliation sweeps — designed behavior"
space: OPS
space_name: "Operations & On-call"
author: Noa Migration
author_username: nmigration
author_email: noa.migration@payzeno.com
created: 2026-05-02
updated: 2026-07-28
labels: [runbook, ledger, reconciliation]
source: confluence-export
page_version: 12
---

# settlement_locked during reconciliation sweeps — designed behavior

> **Operations & On-call** · wiki export · written by Noa Migration · last edited 2026-07-28

This page is the standing description of the ledger's settlement locking behavior
as shipped, and the facts that make a `409 settlement_locked` burst readable. The
convoy — a burst of 409s from the retry route while a reconciliation sweep runs —
is a property of the design, not a fault. Owner: Noa (design), Avi (the alarm side).

## System behavior: how a convoy forms

`RetryScheduler._claim_item` (in `app/services/reconciliation/retry.py`) takes the
batch advisory lock before the row lock — unconditional since PAY-2056 removed the
``reconcile_batch_lock_on_retry`` kill switch. The batch lock is NON-BLOCKING on
purpose (`try_acquire_batch_lock`): a sweep holds the batch lock for minutes across
5,000 items, and a blocking wait would park drain workers on pooled connections.
When the try-acquire fails, the retry returns None and the route maps it onto
`409 settlement_locked` — and the console treats that as "already settling",
refetches, and shows no toast. During a sweep, therefore, every attempt in a drain
pass fails for that batch: the burst of 409s is the system doing what ADR 0011 says
it should. The ordering's sufficiency argument — and why it depends on
READ COMMITTED — is in [the lock-ordering page](../ENG/decision-log-lock-ordering.md).

## Invariants that hold during a convoy

- A `running` `reconciliation_run` row for the noisy batch means the sweep is
  settling those items; the drain skipping them is the happy path wearing an
  error code.
- `409 settlement_locked` is lock noise, never an item failure. Repeated
  `retryable` outcomes with error codes are the acquirer, not the convoy.
- The backlog signal is the `RetryableItemCount` alarm at threshold 500 (see
  [the alarm map](oncall-ledger-alarms.md)); below it the convoy warrants no
  response at all.

## Standing rules — why the obvious fixes are wrong

- **The drain is not scaled.** It runs on one task by design (see
  [the ledger topology page](../ENG/onboarding-ledger-oncall.md)). Extra tasks burn
  their passes failing the same try-lock: more `settlement_locked` noise, zero
  extra throughput.
- **The intervals are fixed.** `RECONCILE_SWEEP_INTERVAL_SECONDS     = "900"` and
  `RETRY_DRAIN_INTERVAL_SECONDS = "60"` in `envs/production/services.tf`; the
  terraform comment on them points back at ADR 0011.
- **There is no third exclusion mechanism, and none gets added.** ADR 0011:
  "There is no third mechanism."
- The drain is Serial on purpose — the backoff exists to stop hammering a degraded
  acquirer. Concurrency here is not a fix; it is the 2041 incident again.

## The accepted cost, and its open follow-up

The convoy is the priced cost of the lock ordering. The candidate-query skip
(PAY-2057 — skip items whose batch has a `running` reconciliation run) remains
open; the code comment in `retry.py` is the record: "PAY-2057 is the fix and it
is not done." When it lands it is an optimisation *in front of* the try-acquire,
not a replacement — the run-status read races the sweep start, so the lock stays.

> Context (Noa): the convoy was read as an outage twice before this page existed —
> the history is mregression's PR #171 thread. Design questions come to me.

*(The row-claim query is `with_for_update(skip_locked=True)` — that skips OTHER
RETRIES, not the sweep. The sweep holds no row locks. That distinction is the
whole 2041 incident.)*
