# NOTES — assumptions and flags

Where the handover pack is silent, I assumed the following. Each one changes a recommendation if it turns out to be wrong.

## Org and context
- I report to the CTO or directly to the CEO, and I have authority over squad allocation, process, and the two open requisitions.
- There is a product function (PMs or a head of product) that owns the CRM and promo roadmaps. I can negotiate scope with them, but I don't own the roadmap myself.
- A compliance/MLRO function exists outside engineering. Regulators in each region have incident-reporting obligations. I don't know the deadlines, so C1 escalates to compliance in the morning rather than me guessing a deadline.
- "Client brands" map to `accountId` (multi-tenant per regional deployment). A brand runs in exactly one region.
- All regions run one shared codebase with per-region config and deployment. That is why config-as-code (Part A §3) fixes incident #3 for every region and not only region 4.
- Some budget flexibility exists, e.g. using unfilled-vacancy budget for a time-boxed contractor.

## Definitions I would agree with the CEO in week 1
- **Money-impacting incident**: any player or brand balance, payment, or regulatory figure that is wrong *and* reaches a player, a brand, or a regulator. A discrepancy that reconciliation catches and fixes before any of those see it counts as a **near-miss**. I track near-misses and report them, but they are not counted against the "zero" target. Without this definition, "zero" either becomes a promise nobody can keep or an incentive to hide problems.
- **Region-4 "launch"**: live with real money for the pilot brands on the minimum viable provider set. Remaining brands and providers onboard in weeks 11–13.

## Part B (PR #482) technical assumptions
- An existing, correct wallet-credit path exists in the codebase (the PR says it is "consistent with the rest of the wallet code"). If it doesn't, building that primitive becomes the first money-safety task in Part A.
- `bets.payout` is nullable for unsettled bets. The pack lists columns but not nullability. If it is non-null, finding 6 becomes "filter to settled bets by settlement time" only.
- Postgres `numeric`/`DECIMAL`, including a constrained `DECIMAL(36,18)`, accepts the special value `'NaN'`. So `String(NaN)` would be stored silently rather than rejected. Verify this against our Postgres version before quoting it in an interview.
- `/admin` auth may be enforced at a gateway or ingress layer that isn't visible in the diff. That's why the comment is phrased as a question plus a requirement for an audit trail, rather than an accusation.
- "~40k members per brand" × ~10 brands per region (30 brands / 3 regions) ≈ 400k members per unscoped run. This is the multiplier behind findings 1 and 8.

## Scope choices
- Part A splits capacity roughly 50/25/25. These are planning numbers, not timesheets.
- I don't recommend a QA hire. A ratio of 1 QA to 10 engineers is within the normal range. The bottleneck comes from *when* testing happens, not from how many testers there are. If the late-regression rate hasn't dropped by week 8, I would revisit this and ask for an SDET.
