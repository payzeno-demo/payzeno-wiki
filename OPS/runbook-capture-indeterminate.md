---
title: "Runbook: capture attempts stuck pending or indeterminate"
space: OPS
space_name: "Operations & On-call"
author: Noa Migration
author_username: nmigration
author_email: noa.migration@payzeno.com
created: 2026-05-08
updated: 2026-06-25
labels: [runbook, ledger, captures, sev-risk]
source: confluence-export
page_version: 7
---

# Runbook: capture attempts stuck pending or indeterminate

> **Operations & On-call** · wiki export · written by Noa Migration · last edited 2026-06-25

**Scope:** deferred captures in payzeno-ledger — the eleven merchants with
`capture_at_settlement` (all travel and lodging MCCs, PAY-1652). The design is
PAY-2060, migration ``0025``, and the code truth is
`app/repositories/capture_attempt.py`.

**The one rule, before anything else: an `indeterminate` capture is NEVER resolved by
re-issuing the capture.** A timeout on a capture is the one state where you do not
know whether the cardholder was charged. Re-capturing "to be safe" is the second,
independent double-charge mechanism that PR #172 did nothing about — the whole reason
this table exists. Indeterminate rows are resolved by
``ProcessorClient.get_capture_status``, never by re-issue. Not by you either: no
manual capture from the acquirer portal, no matter who asks.

## How the mechanism works (so the states make sense)

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

## When the stale gauge trips

`STALE_AFTER = dt.timedelta(hours=1)`. Exported as a gauge and alarmed on at
anything above zero — an hour is generous, the job runs every thirty seconds, so a
stale open row is an invariant violation, not a backlog. It means the deferred job
itself has stalled (look at the job, its DB connectivity, the acquirer's status
endpoint) — the response is fixing the job and letting it resolve the rows, not
resolving rows by hand.

## Support asks "was this charged twice?"

`list_for_charge` in the same repository: every capture attempt against one charge.
Should be one row; two rows with different keys is the shape of the 2041 incident —
this query is in the reconciliation runbook for exactly that reason
(`payzeno-ledger/docs/runbooks/reconciliation.md`). Both acquirers honour our
idempotency key, so a their-side duplicate is identified with `find_by_key`.

History and context: [the incident review](incident-review-pay-2041.md).
Questions: Noa (design), Avi (the alarm side).
