---
title: "billing-legacy: the standing constraints the README does not state"
space: ENG
space_name: "Engineering"
author: Haim Legacy
author_username: hlegacy
author_email: haim.legacy@payzeno.com
created: 2025-10-08
updated: 2026-06-30
labels: [billing-legacy, tribal, statements, dunning]
source: confluence-export
page_version: 7
---

# billing-legacy: the standing constraints the README does not state

> **Engineering** · wiki export · written by Haim Legacy · last edited 2026-06-30

The standing constraints of `payzeno-billing-legacy` that live in no README,
written down so they survive holidays. Parts of this page predate MIG step 3 —
dates cross-check against [Noa's status page](mig-step-4-status.md).

> Context (Haim): "I have been asked to write down what is in my head before I
> take a long holiday. This page is that. It is not complete and will not be."

## Dunning

`dunning_attempt` is Two million rows and growing, because nothing is ever
deleted. Retention is a feature, not neglect: the statement export's dunning
section reaches back years for the banks' files, so the rows stay.
`idx_dunning_attempt_next (outcome, next_attempt_dt)` (V24, added years after the
table) is the ONLY selective index besides the invoice FK — a query that filters
on anything else walks the table.

> Context (Haim): "The scheduler itself: nobody fully understands it, including
> me, and I wrote half of it. Budget time, not confidence."

## Statement export — positional, and therefore frozen

The export is a fixed-width format and the banks parse it POSITIONALLY. Two
consequences, both load-bearing:

- Bank references stopped appearing in these files in month 7 — but
  `private static final int W_BANK_REF = 35;` stays, and columns 61-95 stay blank,
  because removing the width shifts every column after it and the banks' parsers
  do not read headers. An "unused constant" cleanup here is a production incident
  at two banks; that cleanup has been reverted twice.
- Merchant ids in the export go through
  `RedactionSupport.maskAllButLastFour(export.getMerchantPublicId())`. The masking
  is part of the format. The export never shows the full id, and must not.

The sequencing constraint that follows from the read path: the statement path
reads invoices from the LOCAL table (`StatementExportService` →
`InvoiceRepository`, out through `LegacyStatementDao`). Nothing in it reads the
ledger. Whatever the invoice migration does, statements move BEFORE or WITH
invoice ownership — never after.

## Locks

Scheduled jobs use ShedLock. Default `lockAtMostFor` is PT30M
(`ShedLockConfig`); the nightly invoice generation overrides to
`lockAtMostFor = "PT4H"` because it genuinely runs long at month-end. The
operational meaning, per V34's own comment: lock_until is a timeout rather than a
liveness check — a dead node's lock expires, it is not detected. A "stuck" nightly
job may therefore be a held lock inside its window; deleting rows from the
shedlock table to "unstick" it races a node that may be alive.

`LegacyLockTableDao` is `@Deprecated` with a removal TODO on it —
"That TODO is from month 12. V34 has been in production. The file is still here"
is the honest state of legacy cleanup in this codebase generally. It has one
caller left; removing it is a real (small) project, not a drive-by.

## Rounding — both sides, one rule

`Amounts.java`: HALF_UP everywhere. "Never banker's." The ledger's fee engine
(`app/domain/fees.py`) mirrors it with `ROUND_HALF_UP` and carries the remainder
in an explicit `rounding_adjustment` line — and `InvoiceAssembler` treats an even
split as a no-op rather than a rounding event, which is precisely why
"the dual-write can claim the two sides produce the same lines" during the parity
window. A rounding change anywhere is a rounding change everywhere, plus a parity
rerun on both sides. There is no such thing as a one-sided rounding fix.

## Ownership

Statements and dunning: Haim. The strangler seam and cutover sequencing: Noa. The
ledger side of parity: Noa. Anything with the word "canary": Avi.

> Context (Haim): "If I am on holiday, the answer is usually 'wait until Haim is
> back' and I am at peace with that."
