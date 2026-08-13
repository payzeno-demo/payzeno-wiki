---
title: "MIG status: strangler step 4 — invoice lines"
space: ENG
space_name: "Engineering"
author: Noa Migration
author_username: nmigration
author_email: noa.migration@payzeno.com
created: 2026-03-15
updated: 2026-07-27
labels: [migration, strangler, billing-legacy, status]
source: confluence-export
page_version: 21
---

# MIG status: strangler step 4 — invoice lines

> **Engineering** · wiki export · written by Noa Migration · last edited 2026-07-27

Living status page for the MIG arc — strangling `payzeno-billing-legacy` per ADR 0007
(`payzeno-ledger/docs/adr/0007-strangle-billing-legacy.md`, status: accepted,
step 4 of 4). Kept honest on purpose; if this page says something is done and it is
not, tell Noa, loudly.

## Where we are

Steps 1–3 (fees, balances, payouts) are done. Step 4 — invoice **lines** — is
**in flight**, and "in flight" deserves its asterisks: the machinery exists end to
end and has never processed a production row. Migration `0022` created the staging
table, the route is live, and `MigratedInvoiceService` on the Java side calls it —
but `invoice_line_staging` has no production rows, because `FLAG_LEDGER_OWNS_INVOICES` is off
everywhere and always has been. On the api side the terraform says
`FLAG_LEDGER_OWNS_INVOICES = "false"` with a comment worth reading whole: the flag
and the proxy_pass have to move together or the console reads invoices from a
service that does not have them.

## What actually blocks the flip

Not code. The invoice-numbering disagreement, still open, positions unchanged:

- **Mine:** a ledger that stages lines but cannot issue an invoice number is a
  half-migration that will sit for a year.
- **Dudi's:** invoice numbering is a statutory sequence per jurisdiction;
  `SequentialInvoiceNumberGenerator` has a per-country ruleset nobody has written
  down, and the ledger has no business owning a legal artifact it cannot validate.

Both positions are in ADR 0007 verbatim. Until one of us moves, step 4 stays staged.

## Hard constraints on the cutover (from Haim — treat as law)

1. **Statements move BEFORE or WITH invoice ownership, never after.**
   `StatementExportService` reads invoices from the LOCAL table via
   `InvoiceRepository` (and writes through `LegacyStatementDao`); nothing in the
   statement path reads the ledger. Flip the flag first and the SFTP banks get
   stale statements for migrated merchants — and the integration tests will not
   catch it, because they seed the legacy tables directly. More from Haim:
   [billing-legacy tribal knowledge](billing-legacy-tribal.md).
2. **Drain the dual-write backlog before any cutover window.** The seam
   (`MigratedInvoiceService`, "The strangler seam") stages lines via
   `stageInvoiceLines` as a dual-write, and its declared failure mode is
   "local invoice stands, ledger is behind" — TODO(PAY-1905) tracks the
   reconciliation job that is supposed to make that healable. Cut over on top of an
   undrained backlog and the two sides disagree from day one.

## Scar tissue worth knowing about

Step 1 left `legacy_blended_fee` in the ledger on purpose — kept bit-for-bit
compatible with the Java calculator so the parity suite can run both sides against
the same fixtures. Anyone touching fees runs the parity suite on both sides; the
rounding contract behind it is on [Haim's page](billing-legacy-tribal.md).

And a reminder of why the strangler and not a rewrite: the legacy biller has a
jacoco threshold of 55% and roughly nobody left understands the dunning scheduler.
That sentence is from the ADR, and it is still true — which is the argument for
finishing step 4 rather than getting comfortable.
