---
title: "How the payments DLQ redrive path works"
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

# How the payments DLQ redrive path works

> **Operations & On-call** · wiki export · written by Avi Pager · last edited 2026-07-23

The payments DLQ redrive path is safe by construction since the consumer-key
migration, and its safety rests on one property of payzeno-api's dedupe model.
This page states that property, the ownership boundary around redrives, and the
signals that distinguish a real redrive from a silent no-op. History:
[the 04:20 redrive that did nothing](postmortem-appendix-0420-redrive.md).

## How the dedupe behaves

`processed_event` in payzeno-api carries primary key (event_id, consumer) — the
dedupe claim is per consumer, not per event. That key is what makes a redrive
through `DeadLetterConsumer` (whose handled set is "everything, for redrive") able
to do work at all: before the consumer-key migration
(`1725900000000-AddProcessedEventConsumerKey.ts`), any event already processed by
`LedgerEventsConsumer` collided on `event_id` alone and the redrive silently did
nothing. A dedupe table's key IS its semantics — a key that omits the consumer
makes every second consumer a no-op that reports success.

## The ownership boundary

Payments-path redrives are approved by Avi. The convention is load-bearing, not
ceremonial: redriven events feed the ledger, and the ledger's idempotency posture
(see [the lock-ordering page](../ENG/decision-log-lock-ordering.md)) is the only
thing standing between a duplicate event and a duplicate posting — so every
redrive names the guarantee it leans on before it runs. Events also dead-letter
for a reason: a redrive that precedes its diagnosis replays the original failure
at higher volume.

## What a completed redrive looks like

The redrive tool's exit status carries no information — the 04:20 redrive exited
0 having done nothing. The evidence that a redrive happened is its downstream
effects: the consumed count matches the redriven count, the downstream writes
landed (for ledger events, the postings exist), and DLQ depth fell while the
primary consumer's error rate stayed flat. A redrive that "worked" in seconds
with zero downstream writes is the 04:20 pattern —
[the appendix](postmortem-appendix-0420-redrive.md) is the post-incident record
of exactly that shape.
