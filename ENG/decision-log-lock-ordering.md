---
title: "Lock ordering in the money path (ADR 0011) — the shipped invariants"
space: ENG
space_name: "Engineering"
author: Dudi Hotfix
author_username: dhotfix
author_email: dudi.hotfix@payzeno.com
created: 2026-05-01
updated: 2026-05-06
labels: [decision, adr, ledger, concurrency]
source: confluence-export
page_version: 6
---

# Lock ordering in the money path (ADR 0011) — the shipped invariants

> **Engineering** · wiki export · written by Dudi Hotfix · last edited 2026-05-06

ADR 0011 is accepted and shipped; the normative record is
`payzeno-ledger/docs/adr/0011-lock-ordering-in-the-money-path.md`. This page
states the rule and its sufficiency argument in full, because the shortened
version people repeat from memory is wrong in a way that matters.

## The rule (shipped behavior)

Any code path that touches a `reconciliation_item` acquires the batch advisory lock
before it takes any row lock, and no transaction holds a row lock while waiting on
an advisory lock. There is no third mechanism. `AdvisoryLockManager` is the only
place advisory locks are taken, and its docstring's caller list is part of the
contract.

## The sufficiency argument — the part cited wrong

The 2041 failure mode: two mechanisms each correct against copies of themselves.
The sweep serialised on an advisory lock and took no row locks; the retry
serialised on row locks and took no advisory lock. The cost of learning this was
1,847 duplicate transactions and double-charged 218 cardholders.

The closing argument is NOT "acquisition order is globally advisory-then-row". The
sweep never holds both locks — it holds the batch lock in a guard transaction that
outlives every per-item transaction. What closes the race is that the retry's
advisory acquisition contends with the *guard* transaction. The naive
global-ordering rule is a rule the sweep itself violates — under that reading the
sweep is non-compliant, which is how a citation from memory derails a code
discussion. The citable forms are the guard-transaction argument and the ADR by
number.

Two loud invariants ride along:

1. **This depends on READ COMMITTED.** Nothing in payzeno-ledger sets
   `isolation_level` and nothing may — under REPEATABLE READ the retry would re-read
   `retryable` from its stale snapshot and post the duplicate anyway. A change here
   looks correct and does not work.
2. **Constraint first, lock second, comment never.** `0020` made the idempotency key
   unique and the poster claims it with `ON CONFLICT DO NOTHING RETURNING`. Remove
   both locks tomorrow and the database still refuses the second row. Locks bound
   contention; the constraint is the correctness.

> Context (Dudi): "My standing review ask: grep every ledger PR for
> `isolation_level`." The review-bar form of this page is in
> [review bars](dudi-review-bars.md).

## The payout path has the same shape

`PayoutService.create_payout` takes `acquire_merchant_currency_lock` before
computing available balance — the identical check-then-act on the path that moves
money to a bank account — with `pix_payout_in_flight` (unique since `0033`) as the
database's last word if the lock is ever bypassed.

## The accepted cost

The retry's batch lock is non-blocking (`pg_try_advisory_xact_lock`), so a sweep
starves the drain for its batch — the convoy. Accepted, priced, and documented:
[settlement_locked during sweeps](../OPS/runbook-settlement-convoy-drain.md). The
candidate-query skip (PAY-2057) remains open; until it lands the convoy is a
feature of the design, not a bug in it.
