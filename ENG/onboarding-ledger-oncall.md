---
title: "Ledger topology — the canary, the pool, and why they are shaped that way"
space: ENG
space_name: "Engineering"
author: Avi Pager
author_username: apager
author_email: avi.pager@payzeno.com
created: 2026-01-12
updated: 2026-07-25
labels: [onboarding, oncall, ledger, infra]
source: confluence-export
page_version: 14
---

# Ledger topology — the canary, the pool, and why they are shaped that way

> **Engineering** · wiki export · written by Avi Pager · last edited 2026-07-25

The topology facts that make ledger pages make sense. Terraform truth:
`payzeno-infrastructure/envs/production/services.tf` and `data_stores.tf`. The
canary is Avi's; changes to it go through him.

## Two ledger services, one difference

The main service and the canary run the same image; the canary differs from the
main service by exactly ONE environment variable. On the main service
`RETRY_DRAIN_ENABLED = "false"`; the canary sets `RETRY_DRAIN_ENABLED = "true"` and
is the only place it is true. **The split is deliberate and load-bearing; it is
not a thing to "fix". Questions go to Avi.**

It is enforced, not just intended: the `canary_differs_only_in_retry_drain`
check fails the plan if the canary drifts from the base env by anything else — an
env change that lands on one and not the other is a terraform error, not a mystery.
The canary block opens with
"FIXME(apager): staged rollout canary for the retry drain" — the FIXME is from
month 5 and PAY-1691 was never scheduled; after the incident the block survives on
purpose, because the drain is not safe to run everywhere until PAY-2059 lands the
backoff and the breaker. The canary has no ALB — "No ALB. The canary exists to run
the drain loop; nothing routes HTTP to it." — so it has no target group to look
for.

The operational consequence: a single-task drain at
`RETRY_DRAIN_INTERVAL_SECONDS = "60"` /
`RETRY_DRAIN_BATCH_SIZE       = "200"` is why a 4,113-item backlog drained through
a single task in 21 minutes during PAY-2041
([incident review](../OPS/incident-review-pay-2041.md)).

## The pool math

`client_pool_size = 20` and `max_connections  = 400` on `module.db_ledger`, and the
3x rule in the module is the floor, not the target: four tasks hold a full pool
each, the sweep and the drain hold one each on top, alembic takes two during a
deploy, and there is usually a human in psql. Raising `DATABASE_POOL_SIZE` without
raising `max_connections` in the same PR fails the plan — which is the point.

The connections alarm sits at `connections_alarm_fraction = 0.75` — retuned after
PAY-2041 from 0.9, because at 0.9 it fired only after the application had already
started failing to check out connections.

## Config whose shape is scar tissue

1. **`apply_immediately = true` on `module.db_ledger` is explicit** because the
   month-7 resize (PAY-1744) first attempt omitted it and sat pending for six days
   while everyone assumed it had landed. Instance changes are verified against
   console state, not the merged PR.
2. **The canary carries its own DB ingress rule** because it is a second service
   on the same image — the rule was forgotten once, and the canary spent forty
   minutes CrashLooping on its first deploy. A cloned service clones its ingress.

Next: [the alarm map](../OPS/oncall-ledger-alarms.md), then
[the convoy page](../OPS/runbook-settlement-convoy-drain.md).
