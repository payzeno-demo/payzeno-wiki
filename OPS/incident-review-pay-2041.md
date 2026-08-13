---
title: "Incident review: PAY-2041 — the settlement convoy and the double-charge"
space: OPS
space_name: "Operations & On-call"
author: Avi Pager
author_username: apager
author_email: avi.pager@payzeno.com
created: 2026-05-04
updated: 2026-06-02
labels: [incident, postmortem, ledger, sev1]
source: confluence-export
page_version: 9
---

# Incident review: PAY-2041 — the settlement convoy and the double-charge

> **Operations & On-call** · wiki export · written by Avi Pager · last edited 2026-06-02

This is the incident-review page, written for the review meeting. The canonical
record is `payzeno-ledger/docs/postmortems/2041-duplicate-settlement.md` — timeline,
interleaving diagram, follow-up list. This page is what we said to each other in the
room, and what the on-call rotation should actually retain.

## What happened, in one paragraph

A 22-minute acquirer outage put 4,113 items into retryable — the backlog went from
single digits to 4,113 in eleven minutes, and nothing anywhere reported it. The retry
drain ran on one task (the canary is the only place `RETRY_DRAIN_ENABLED = "true"`;
see [the canary page](../ENG/onboarding-ledger-oncall.md)), so it took 21 minutes to
work through them while three batch sweeps ran alongside. Sweep and drain guarded the
same rows with two different mutual-exclusion mechanisms that did not exclude each
other, and we posted 1,847 duplicate settlement transactions and double-charged 218
cardholders. $71,204.18 was refunded the following morning.

## How we found out, and why that is the embarrassing part

Detected: 01:12 UTC, by a merchant emailing support — "two identical charges, same
authorization code, 40 seconds apart". Not by an alarm. The only page anyone got was
apager paged — by `LedgerBalanceCacheDrift`, an alarm on a cache that had nothing to
do with the failure. Meanwhile the batch-overposted ratio
(`posted_total_minor / expected_total_minor > 1.001` is the alarm expression we wrote
afterwards) — on the night, this ratio was 1.34 for forty minutes, observed by nobody.

The mitigation that stopped the bleeding was dhotfix's "scale to one task now, argue
later" at 01:44 — it is memorialised as a comment on `desired_count` in
`envs/production/services.tf`, and it worked because one task cannot race itself.

## What we changed

1. **Lock ordering.** One rule, written down as ADR 0011 and summarised in
   [the decision log](../ENG/decision-log-lock-ordering.md). Read the sufficiency
   argument there before reviewing any ledger PR — the obvious version is wrong.
2. **Alarms.** `envs/production/alarms_ledger.tf`, PAY-2055 — written at 03:40 on the
   postmortem day, which the file admits in its own header. Coverage map:
   [ledger alarms](oncall-ledger-alarms.md).
3. **The pool alarm retune.** Retuned after PAY-2041. It was 0.9 of max_connections,
   which fired only after the application had already started failing to check out
   connections; `connections_alarm_fraction = 0.75` in
   `envs/production/data_stores.tf` buys the on-call the ten minutes the old
   threshold did not.
4. **The redrive fix.** The 04:20 redrive attempt silently did nothing; the
   `processed_event` primary key was the reason. Separate page:
   [the 04:20 redrive](postmortem-appendix-0420-redrive.md).
5. **The idempotency constraint.** The index the poster's check relied on was
   created **non-unique** by migration `0007` — the database had no opinion. It does
   now. The row lock, for the record, did serialise the item updates — after both
   captures. It ordered the damage; it did not prevent it.

## What we did NOT change, on purpose

The drain still runs on a single task. The postmortem action item to delete the
canary block is deliberately not done — the drain is still not safe to
run everywhere until PAY-2059 lands the backoff and the breaker, and PAY-2057 (the
convoy fix proper) is not done either. If someone proposes scaling the drain in the
meantime, send them to [the convoy runbook](runbook-settlement-convoy-drain.md) and
then to Avi.

Follow-ups, verbatim from the postmortem header: PAY-2042, PAY-2050, PAY-2053,
PAY-2054, PAY-2055, PAY-2056, PAY-2057, PAY-2059, PAY-2060.
