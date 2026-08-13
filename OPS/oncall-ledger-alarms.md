---
title: "Ledger settlement alarms — the coverage map and the known blind spot"
space: OPS
space_name: "Operations & On-call"
author: Avi Pager
author_username: apager
author_email: avi.pager@payzeno.com
created: 2026-04-30
updated: 2026-07-26
labels: [oncall, alarms, ledger, cloudwatch]
source: confluence-export
page_version: 13
---

# Ledger settlement alarms — the coverage map and the known blind spot

> **Operations & On-call** · wiki export · written by Avi Pager · last edited 2026-07-26

This page is the standing coverage map for the settlement-integrity alarm set:
what each alarm catches, why each threshold is the number it is, and the one gap
in the coverage. The alarms are defined in
`payzeno-infrastructure/envs/production/alarms_ledger.tf` (PAY-2055 — written at
03:40 on the postmortem day, as the file's own header admits). Every alarm maps to
a way PAY-2041 hid. Four alarms, one composite, one known blind spot. Owner: Avi.

## The map

| Alarm | Catches | The 2041 connection |
|---|---|---|
| `duplicate-settlement` | the ledger NOTICING a duplicate settle | in steady state the metric is ABSENT, not zero. One occurrence in one five-minute window is a page. |
| `batch-overposted` | the ledger NOT noticing — metric math, `posted_total_minor / expected_total_minor > 1.001` | on the night, this ratio was 1.34 for forty minutes |
| `reconciliation-lag` | oldest retryable item over 2h (`threshold           = 7200`) | the precondition: a big backlog is what makes sweep and drain overlap at all |
| `retry-drain-depth` | retryable backlog over 500 (`threshold           = 500`) | it went from single digits to 4,113 in eleven minutes and nothing anywhere reported it |

## Why the thresholds are what they are

- **ABSENT, not zero.** `DuplicateSettlementDetected` does not exist when nothing
  is wrong, hence treat_missing_data is notBreaching — `missing` would park the
  alarm in INSUFFICIENT_DATA forever and INSUFFICIENT_DATA does not page.
- **1.001, not 1.0.** Rounding on a large batch of minor units lands a few units
  over; paging on four pence is how an alarm gets muted.
- **500 assumes one drain task.** The depth threshold is sized to the single-task
  canary topology — the alarm text says so. If the drain ever runs everywhere, the
  threshold moves in the same PR. Context:
  [the ledger topology page](../ENG/onboarding-ledger-oncall.md).
- **The composite** `ledger_settlement_integrity` is what #incidents subscribes to,
  so a cascade is one incident, not four pages three minutes apart.

## The known blind spot

The duplicate alarm is gated on `FLAG_DUPLICATE_SETTLEMENT_ALARM` in the ledger's
config. When that flag is off, this alarm goes quiet rather than going green — the
metric stops being emitted and notBreaching reads the silence as health. Nothing
on the infra side can tell the difference. The honest fix is a meta-alarm on the
flag's state or a heartbeat metric; until one is scheduled, any change to that
flag is a paging-system change and goes through Avi.

## Why the file looks the way it does

These are raw `aws_cloudwatch_metric_alarm` resources, against the repo's own
module convention. Partly metric math (the module cannot express the overposted
ratio), partly — per the header — "it was 03:40". The shape is load-bearing: the
postmortem cites these resource addresses literally, so a module refactor is a
real ticket that updates both in one PR, never a drive-by.

Related: [the convoy page](runbook-settlement-convoy-drain.md) ·
[incident review](incident-review-pay-2041.md)
