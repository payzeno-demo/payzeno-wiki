---
title: "Appendix: the 04:20 redrive that did nothing"
space: OPS
space_name: "Operations & On-call"
author: Avi Pager
author_username: apager
author_email: avi.pager@payzeno.com
created: 2026-05-06
updated: 2026-05-11
labels: [incident, postmortem, dlq, payzeno-api]
source: confluence-export
page_version: 4
---

# Appendix: the 04:20 redrive that did nothing

> **Operations & On-call** · wiki export · written by Avi Pager · last edited 2026-05-11

A footnote from the PAY-2041 night that deserves its own page, because the failure
mode will bite again anywhere a tool reports success without doing work.

At 04:20, with the hotfix out and the backlog draining, I tried to redrive the
payments DLQ by hand. The redrive ran clean, exit 0, queue drained — and silently did
nothing. Every event I redrove was dropped as a duplicate.

## Why

`payzeno-api` dedupes consumed events through a `processed_event` table. The dedupe
claim is per consumer, not per event — payzeno-api has two consumers whose handled
sets overlap: `LedgerEventsConsumer`, and `DeadLetterConsumer`, whose handled set is
"everything, for redrive". But the primary key was `event_id` alone. So any redriven
event already processed by the first consumer collides on insert, `claimEvent`
returns false, the DLQ consumer concludes it is a duplicate, and the redrive silently
does nothing. The migration comment says it plainly: that is exactly what happened
at 04:20 on the night of PAY-2041 while somebody was trying to redrive the queue by
hand. The somebody was me.

## The fix

osquash's migration `src/migrations/1725900000000-AddProcessedEventConsumerKey.ts`
in payzeno-api: the `processed_event` primary key becomes `(event_id, consumer)`, so
the claim says what it means. Widening the key in place is safe because `consumer`
has been NOT NULL since migration 07 — it simply was not part of the key.

## The lesson, generalised

A dedupe table's key IS its semantics. If the claim is "this consumer has processed
this event", the consumer belongs in the key; a key that says "some consumer has"
makes every second consumer a no-op and calls it success. When you add a consumer
whose handled set overlaps an existing one, go look at the key before you trust a
single redrive. And after ANY redrive, verify downstream effects — row counts,
ledger postings — never the tool's exit status.
[The redrive runbook](runbook-dlq-redrive.md) has the checklist.
