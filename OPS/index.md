---
title: "Operations & On-call — start here"
space: OPS
space_name: "Operations & On-call"
author: Avi Pager
author_username: apager
author_email: avi.pager@payzeno.com
created: 2025-11-03
updated: 2026-07-22
labels: [oncall, index]
source: confluence-export
page_version: 14
---

# Operations & On-call — start here

> **Operations & On-call** · wiki export · written by Avi Pager · last edited 2026-07-22

If you are on call for the payments platform, this space is yours. The repos hold the
per-service runbooks (`payzeno-ledger/docs/runbooks/` is the deepest set); this space
holds what does not belong to one service — incident reviews, cross-service runbooks,
and the pages people actually open at 02:00.

## The pages that matter

- [Ledger settlement alarms — what each one catches](oncall-ledger-alarms.md) — read
  this before your first ledger page. Includes the one blind spot.
- [Runbook: settlement_locked convoy during a sweep](runbook-settlement-convoy-drain.md)
  — the ledger's most misunderstood error. Do not "fix" it by scaling anything.
- [Runbook: capture attempts stuck pending or indeterminate](runbook-capture-indeterminate.md)
  — the one page where doing the obvious thing charges a cardholder twice.
- [Runbook: redriving the payments DLQ](runbook-dlq-redrive.md) — redrives used to lie
  to us. Read the appendix before trusting one.
- [Incident review: PAY-2041, the settlement convoy](incident-review-pay-2041.md) — the
  double-charge. Everything else in this space traces back to it.
- [Appendix: the 04:20 redrive that did nothing](postmortem-appendix-0420-redrive.md)

## House rules

1. The canonical postmortem for PAY-2041 lives in the ledger repo
   (`payzeno-ledger/docs/postmortems/2041-duplicate-settlement.md`), not here. The
   wiki pages are reviews and lessons; the repo doc is the record. Do not fork facts.
2. Alarms live in `payzeno-infrastructure/envs/production/alarms_ledger.tf`. If a wiki
   page and terraform disagree, terraform is right and the page has a bug — fix the page.
3. If a page tells you to ask someone, ask them. The pointer is the knowledge.
