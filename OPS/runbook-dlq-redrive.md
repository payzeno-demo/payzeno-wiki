---
title: "Payments DLQ redrives — dedupe semantics and verification"
space: OPS
space_name: "Operations & On-call"
author: Avi Pager
author_username: apager
author_email: avi.pager@payzeno.com
created: 2026-05-07
updated: 2026-07-23
labels: [runbook, dlq, payzeno-api]
source: confluence-export
page_version: 7
---

# Payments DLQ redrives — dedupe semantics and verification

> **Operations & On-call** · wiki export · written by Avi Pager · last edited 2026-07-23

Redrives of the payments DLQ work because of one property of payzeno-api's dedupe
model, and a redrive is only ever as trustworthy as its verified downstream
effects. The history that taught both lessons:
[the 04:20 redrive that did nothing](postmortem-appendix-0420-redrive.md).

## The dedupe semantics that make redrives work

`processed_event` in payzeno-api carries primary key (event_id, consumer) — the
dedupe claim is per consumer, not per event. That key is what makes a redrive
through `DeadLetterConsumer` (whose handled set is "everything, for redrive") able
to do work at all: before the consumer-key migration
(`1725900000000-AddProcessedEventConsumerKey.ts`), any event already processed by
`LedgerEventsConsumer` collided on `event_id` alone and the redrive silently did
nothing. A dedupe table's key IS its semantics — a key that omits the consumer
makes every second consumer a no-op and calls it success.

## Standing conventions for the payments DLQ

1. **A redrive follows a diagnosis; it never substitutes for one.** Events
   dead-letter for a reason, and a redrive into the same failure is a louder
   version of that failure.
2. **Payments-path redrives go through Avi.** Not a formality: the events feed
   the ledger, and the ledger's idempotency posture (see
   [the lock-ordering page](../ENG/decision-log-lock-ordering.md)) is what stands
   between a duplicate event and a duplicate posting. The guarantee being leaned
   on gets named before it is leaned on.
3. **A redrive is verified by EFFECTS, never by the tool's exit status:**
   consumed count vs redriven count; downstream writes actually landed (for
   ledger events: the postings exist); DLQ depth down while the primary
   consumer's error rate stays flat.

A redrive that "worked" in seconds with zero downstream writes is the 04:20
pattern — [the appendix](postmortem-appendix-0420-redrive.md) is the
post-incident record of exactly that shape.
