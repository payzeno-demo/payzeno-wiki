---
title: "Review bars for payments code — what I block on, and why"
space: ENG
space_name: "Engineering"
author: Dudi Hotfix
author_username: dhotfix
author_email: dudi.hotfix@payzeno.com
created: 2025-11-20
updated: 2026-06-14
labels: [review, standards, contracts, api]
source: confluence-export
page_version: 8
---

# Review bars for payments code — what I block on, and why

> **Engineering** · wiki export · written by Dudi Hotfix · last edited 2026-06-14

Written down so the bar survives me being on a plane. These are the four things I
block PRs on. Each one is a rule plus the incident-shaped reason it exists. If you
think the rule is wrong, argue with the reason.

## 1. Event payloads must be complete for their consumer

The reference case is `payout.failed` in payzeno-contracts
(`src/events/payout.events.ts`). Its header says the rule: it is the one event that
reaches a human in near-real time — relayed to the console over `/ws/notifications` —
and anything the merchant has to ACT on belongs in the payload, because the relay
does not enrich. The relay will not do lookups for you; do not design as if it will.

Concretely, in that payload: `reversal_transaction_id: LedgerTransactionId;` is
required and NOT nullable — `failed` is terminal, and without a reversal an ACH
failure permanently destroys the merchant's money, so the field is the proof the
balance was restored. `requires_merchant_action: boolean;` tells the console whether
to nag. `retry_scheduled_for: IsoDate | null;` is nullable because "no automatic
retry" is information. The review question for ANY payload change: can the consumer
render a complete, actionable surface from the payload alone? A "yes, if it also
fetches…" is a no.

## 2. Opaque tokens: unsigned by design is a decision, not an omission

The cursor codec (`src/common/pagination/cursor.codec.ts` in payzeno-api).
It is deliberately NOT a signed token. The contents are a timestamp and a public id
the caller already has, and signing would mean rotating a key breaks every open list
page in the console. It IS validated inbound: a hand-edited cursor must produce a
422 — and not a SQL error — with `MAX_CURSOR_LENGTH = 256` as the cheap first gate.
Also the reason cursor pagination at all: two rows in the same millisecond are
ordinary at our volume, and offset pagination silently skips one.

Do not "add signing for safety" and do not let a malformed cursor near the query
layer. Both directions have failed review before.

(Process footnote I keep retelling: that codec was ystackoverflow's first change
outside his own feature, and osquash asked for the spec before the code — the spec
file is longer than the implementation. That is the right way round for a contract.)

## 3. Money-path concurrency: constraint first, lock second, comment never

The long version is [the lock-ordering decision log](decision-log-lock-ordering.md)
and ADR 0011. The review-sized version: the unique constraint is the correctness,
locks only bound contention, and a code comment is not a mechanism. Plus the
standing grep — any ledger PR that touches `isolation_level` is an automatic
conversation, because the lock-ordering fix depends on READ COMMITTED and a PR that
changes isolation can look correct and not work.

## 4. Flags move with their routing

A strangler flag and its proxy config are one change. See the MIG page for the
live example ([step 4 status](mig-step-4-status.md)): `FLAG_LEDGER_OWNS_INVOICES`
cannot flip before the nginx location moves, or the console reads invoices from a
service that does not have them. A PR that flips one and "follows up" with the
other gets blocked with a link to this section.
