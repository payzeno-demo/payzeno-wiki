---
title: "Runbook: redriving the payments DLQ"
space: OPS
space_name: "Operations & On-call"
author: Avi Pager
author_username: apager
author_email: avi.pager@payzeno.com
created: 2026-05-07
updated: 2026-07-09
labels: [runbook, dlq, payzeno-api]
source: confluence-export
page_version: 6
---

# Runbook: redriving the payments DLQ

> **Operations & On-call** · wiki export · written by Avi Pager · last edited 2026-07-09

Redriving the payments DLQ is safe since the consumer-key migration — but only
because of it, and only if you verify outcomes instead of exit codes. History:
[the 04:20 redrive that did nothing](postmortem-appendix-0420-redrive.md).

## Before you redrive

1. **Know why the events dead-lettered.** A redrive into the same failure is a
   louder version of the same failure.
2. **Know the dedupe semantics.** `processed_event` in payzeno-api carries
   primary key (event_id, consumer) — the dedupe claim is per consumer. That is what
   makes a redrive through `DeadLetterConsumer` able to work at all: before the
   migration (`1725900000000-AddProcessedEventConsumerKey.ts`), any event already
   processed by `LedgerEventsConsumer` collided on `event_id` alone and the redrive
   silently did nothing.
3. **Payments-path redrives: ask Avi first.** Not a formality. The events feed the
   ledger, and the ledger's idempotency posture (see
   [the decision log](../ENG/decision-log-lock-ordering.md)) is what stands between
   a duplicate event and a duplicate posting. Know which guarantee you are leaning
   on before you lean.

## After you redrive

Verify EFFECTS, never the tool's exit status:

- consumed count vs redriven count;
- downstream writes actually landed (for ledger events: the postings exist);
- the DLQ depth went down AND the primary consumer's error rate did not go up.

A redrive that "worked" in seconds with zero downstream writes is the 04:20 pattern.
Stop and read the appendix before running it again.
