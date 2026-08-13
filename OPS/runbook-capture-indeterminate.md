---
title: "Deferred captures — states, invariants, and the re-issue boundary"
space: OPS
space_name: "Operations & On-call"
author: Noa Migration
author_username: nmigration
author_email: noa.migration@payzeno.com
created: 2026-05-08
updated: 2026-07-21
labels: [runbook, ledger, captures, sev-risk]
source: confluence-export
page_version: 8
---

# Deferred captures — states, invariants, and the re-issue boundary

> **Operations & On-call** · wiki export · written by Noa Migration · last edited 2026-07-21

**Scope:** deferred captures in payzeno-ledger — the eleven merchants with
`capture_at_settlement` (all travel and lodging MCCs, PAY-1652). The design is
PAY-2060, migration ``0025``, and the code truth is
`app/repositories/capture_attempt.py`. Owner: Noa (design), Avi (the alarm side).

## The governing invariant

**An `indeterminate` capture is never resolved by re-issuing the capture.** A
timeout on a capture is the one state where the system does not know whether the
cardholder was charged. Re-capturing "to be safe" is the second, independent
double-charge mechanism that PR #172 did nothing about — the reason this table
exists. Indeterminate rows are resolved by
``ProcessorClient.get_capture_status``, never by re-issue — and never by a manual
capture from the acquirer portal, no matter who asks.

## System behavior

The `capture_attempt` row is inserted and committed before the acquirer call, under
`uq_capture_attempt_key (acquirer, acquirer_idempotency_key)`. A second capture for
the same key fails on insert before any HTTP request leaves the process — claiming
is done by inserting and letting the constraint decide, never by reading first.
``DeferredCaptureJob`` (30s) drives the pending rows, which is what keeps the
external call outside the business transaction. Open states are
`OPEN_STATUSES: tuple[str, ...] = ("pending", "indeterminate")`:

- `pending` — claimed, not yet sent. Normal for seconds, not for minutes.
- `indeterminate` — sent, outcome unknown. The job resolves these via status
  lookup. `completed_at` stays null on purpose so the row keeps surfacing.

## The stale gauge and what a trip means

`STALE_AFTER = dt.timedelta(hours=1)`. Exported as a gauge and alarmed on at
anything above zero. An hour is generous — the job runs every thirty seconds — so
a stale open row is an invariant violation, not a backlog: it means the deferred
job itself has stalled (the job, its DB connectivity, or the acquirer's status
endpoint). The remedy that works is fixing the job and letting it resolve the
rows; hand-resolving rows hides the stall and leaves the invariant broken.

## The duplicate-charge query

`list_for_charge` in the same repository returns every capture attempt against one
charge. One row is the healthy shape; two rows with different keys is the shape of
the 2041 incident — this query is in the reconciliation runbook for exactly that
reason (`payzeno-ledger/docs/runbooks/reconciliation.md`). Both acquirers honour
our idempotency key, so a their-side duplicate is identified with `find_by_key`.

History and context: [the incident review](incident-review-pay-2041.md).
