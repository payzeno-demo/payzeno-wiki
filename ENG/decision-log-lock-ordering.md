---
title: "Decision log: one lock order in the money path (ADR 0011)"
space: ENG
space_name: "Engineering"
author: Dudi Hotfix
author_username: dhotfix
author_email: dudi.hotfix@payzeno.com
created: 2026-05-01
updated: 2026-05-06
labels: [decision, adr, ledger, concurrency]
source: confluence-export
page_version: 5
---

# Decision log: one lock order in the money path (ADR 0011)

> **Engineering** · wiki export · written by Dudi Hotfix · last edited 2026-05-06

Summary page for `payzeno-ledger/docs/adr/0011-lock-ordering-in-the-money-path.md`,
written three days after PAY-2041. The ADR is normative; this page exists because
people keep citing the rule slightly wrong in reviews, and the slightly-wrong
version is dangerous.

## The rule

Any code path that touches a `reconciliation_item` acquires the batch advisory lock
before it takes any row lock, and no transaction holds a row lock while waiting on
an advisory lock. There is no third mechanism. `AdvisoryLockManager` is the only
place advisory locks are taken, and its docstring's caller list is part of the
contract.

## The sufficiency argument — read this part twice

The incident happened because two mechanisms were each correct against copies of
themselves: the sweep serialised on an advisory lock and took no row locks; the
retry serialised on row locks and took no advisory lock. We posted 1,847 duplicate
transactions and double-charged 218 cardholders to learn that.

The fix is NOT "acquisition order is globally advisory-then-row". The sweep never
holds both locks — it holds the batch lock in a guard transaction that outlives
every per-item transaction. The argument that closes the race is that the retry's
advisory acquisition contends with the *guard* transaction. A reviewer who cites the
naive global-ordering rule is citing a rule the sweep itself violates — if that
version were the requirement, the sweep would be non-compliant and the review
conversation goes sideways from there. Cite the guard-transaction argument or cite
the ADR by number, not from memory.

Two loud invariants ride along:

1. **This depends on READ COMMITTED.** Nothing in payzeno-ledger sets
   `isolation_level` and nothing may — under REPEATABLE READ the retry would re-read
   `retryable` from its stale snapshot and post the duplicate anyway. The fix would
   look correct and not work. My standing review ask: grep every ledger PR for
   `isolation_level`.
2. **Constraint first, lock second, comment never.** `0020` made the idempotency key
   unique and the poster claims it with `ON CONFLICT DO NOTHING RETURNING`. Remove
   both locks tomorrow and the database still refuses the second row. Locks bound
   contention; the constraint is the correctness.

## The payout path has the same shape

`PayoutService.create_payout` takes `acquire_merchant_currency_lock` before
computing available balance — the identical check-then-act on the path that moves
money to a bank account — with `pix_payout_in_flight` (unique since `0033`) as the
database's last word if the lock is ever bypassed.

## The accepted cost

The retry's batch lock is non-blocking (`pg_try_advisory_xact_lock`), so a sweep
starves the drain for its batch — the convoy. Accepted, priced, and runbooked:
[settlement_locked convoy](../OPS/runbook-settlement-convoy-drain.md). The real fix
is PAY-2057 (skip items whose batch has a `running` run) and until it lands the
convoy is a feature of the design, not a bug in it.
