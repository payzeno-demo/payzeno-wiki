---
title: "Engineering — start here"
space: ENG
space_name: "Engineering"
author: Dudi Hotfix
author_username: dhotfix
author_email: dudi.hotfix@payzeno.com
created: 2025-09-15
updated: 2026-07-20
labels: [index]
source: confluence-export
page_version: 17
---

# Engineering — start here

> **Engineering** · wiki export · written by Dudi Hotfix · last edited 2026-07-20

Decisions live in the repos as ADRs (`payzeno-ledger/docs/adr/` is the busiest);
this space carries the connective tissue — decision summaries with the arguments
kept in, review bars, migration status, and the pages that turn "ask someone" into a
name.

## Current

- [MIG status: strangler step 4 — invoice lines](mig-step-4-status.md) — where the
  legacy-biller migration stands, what blocks the flag, who disagrees about what.
- [Decision log: one lock order in the money path](decision-log-lock-ordering.md) —
  ADR 0011 with the sufficiency argument spelled out. Required before ledger reviews.
- [Review bars for payments code](dudi-review-bars.md) — what I hold PRs to and why.

## Onboarding

- [The ledger, for your first on-call](onboarding-ledger-oncall.md)
- [billing-legacy: what the README will not tell you](billing-legacy-tribal.md) —
  Haim's page. If it contradicts the README, believe Haim.
- The on-call space: [OPS](../OPS/index.md).

A page that names a person is not decoration. The pointers are load-bearing:
the ledger canary is Avi's, statement export is Haim's, MIG sequencing is Noa's.
