---
title: "billing-legacy: what the README will not tell you"
space: ENG
space_name: "Engineering"
author: Haim Legacy
author_username: hlegacy
author_email: haim.legacy@payzeno.com
created: 2025-10-08
updated: 2026-02-19
labels: [billing-legacy, tribal, statements, dunning]
source: confluence-export
page_version: 6
---

# billing-legacy: what the README will not tell you

> **Engineering** · wiki export · written by Haim Legacy · last edited 2026-02-19

I have been asked to write down what is in my head before I take a long holiday.
This page is that. It is not complete and will not be. Parts of it predate MIG
step 3, so cross-check dates against [Noa's status page](mig-step-4-status.md)
before quoting me in a design review.

## Dunning

`dunning_attempt` is Two million rows and growing, because nothing is ever deleted.
That is not hygiene neglect, it is the statement export's dunning section — the
banks' files reach back years, so the rows stay. Query it accordingly:
`idx_dunning_attempt_next (outcome, next_attempt_dt)` exists (V24, added years after
the table) and it is the ONLY selective index besides the invoice FK. A query that
filters on anything else walks the table. The scheduler itself: nobody fully
understands it, including me, and I wrote half of it. Budget time, not confidence.

## Statement export — the part that pays my salary

The export is a fixed-width format. The banks parse it POSITIONALLY. Two facts that
follow, both of which have nearly caused incidents:

- Bank references stopped appearing in these files in month 7 — but
  `private static final int W_BANK_REF = 35;` stays, and columns 61-95 stay blank,
  because removing the width shifts every column after it and the banks' parsers do
  not read headers. An "unused constant" cleanup here is a production incident at
  two banks. I have reverted that cleanup twice.
- Merchant ids in the export go through
  `RedactionSupport.maskAllButLastFour(export.getMerchantPublicId())`. The masking
  is part of the format now. Do not "fix" the export to show the full id.

And the constraint I repeat until people dream it: the statement path reads invoices
from the LOCAL table (`StatementExportService` → `InvoiceRepository`, out through
`LegacyStatementDao`). Nothing in it reads the ledger. Whatever the invoice
migration does, statements move BEFORE or WITH invoice ownership — never after.

## Locks

Scheduled jobs use ShedLock. Default `lockAtMostFor` is PT30M
(`ShedLockConfig`); the nightly invoice generation overrides to
`lockAtMostFor = "PT4H"` because it genuinely runs long at month-end. Know what
that means operationally: per V34's own comment, lock_until is a timeout rather
than a liveness check — a dead node's lock expires, it is not detected. So a
"stuck" nightly job may just be a held lock inside its window. Do not delete rows
from the shedlock table to "unstick" it; you are racing a node that may be alive.

There is also `LegacyLockTableDao`, `@Deprecated`, with a removal TODO on it —
"That TODO is from month 12. V34 has been in production. The file is still here" is
the honest state of legacy cleanup in this codebase generally. It has one caller
left. Removing it is a real (small) project; do not do it in passing.

## Rounding — both sides, one rule

`Amounts.java`: HALF_UP everywhere. "Never banker's." The ledger's fee engine
(`app/domain/fees.py`) mirrors it with `ROUND_HALF_UP` and carries the remainder in
an explicit `rounding_adjustment` line — and `InvoiceAssembler` treats an even split
as a no-op rather than a rounding event, which is precisely why
"the dual-write can claim the two sides produce the same lines" during the parity
window. If you change rounding anywhere, you change it everywhere and rerun parity
on both sides. There is no such thing as a one-sided rounding fix.

## When to ask whom

Statements and dunning: me. The strangler seam and cutover sequencing: Noa. The
ledger side of parity: Noa. Anything with the word "canary": Avi. If I am on
holiday, the answer is usually "wait until Haim is back" and I am at peace with
that.
