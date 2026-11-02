# Engineering Manager Take-Home

Assumptions are stated inline as *(assumption: …)*; the longer list is in `NOTES.md`.

---

# Part A — 90-day plan

**Thesis.** The org scaled regions and brands without scaling its guardrails. Money correctness, launches and system knowledge live in a few heads and are checked by humans at the end of the pipe. This quarter I make region 4 the first launch done by the system rather than by heroes, and move money safety from "careful reviewers" to "things that can't happen".

## 1. Diagnosis — five root patterns

1. **Money safety depends on vigilance, not design.** #1 (idempotency), #4 (float) and 3 of 5 minor incidents all *passed review*. No shared primitive makes the wrong thing hard (idempotent ledger credit, decimal-only money, tenant-scoped queries), so every PR re-solves money correctness and review is the only control. PR #482 (Part B) repeats #1 and #4 and was approved by our best engineer — the control itself is failing.
2. **Critical knowledge has a bus factor of one.** S. (wallet), T. (pipeline). #10–11 are literally "T. was on leave". S.'s review style suppresses the learning that would fix it; T.'s on-call load blocks documentation. Both are burnout/flight risks.
3. **Region launches are artisanal.** #3 is copy-pasted config; region 3 overran 2.7× and nobody asked why. Nothing was learned, so region 4 inherits the same failure modes plus new providers and regulation.
4. **Quality is inspected at the end, so batches grow.** Late regressions → slipped releases → bi-weekly cadence → bigger batches → more regressions. QA isn't the bottleneck; it's where the bottleneck becomes visible. Backlog +30% QoQ is the symptom.
5. **The org cannot see or learn from itself.** No incident process, post-mortems or trusted metrics. Rising MTTR is the expected result of "whoever notices, fixes". #2 (batch job exhausting the pool) shows operability is nobody's job.

## 2. Priorities and trade-offs

Capacity split: **region 4 ≈ 50% · money safety ≈ 25% · run/support ≈ 25%.** In order:

1. **Weeks 1–2 — stop the money bleeding (cheap, high leverage).** Money-path change control (§3); S. + D. time-boxed audit of every credit/debit path for idempotency, float and tenant scoping, fix the top risks; unique constraints on provider transaction IDs; **daily ledger↔balance and provider↔deposit reconciliation per region, with alerts.** Reconciliation is what makes "zero money incidents" credible: I can't promise zero bugs, I can promise detection in hours, before money leaves via withdrawal. I agree that definition with the CEO in week 1.
2. **Week 1 onward — region 4 on the critical path.** Week 1: 90-min blameless retro of region 3 and #3; its output becomes the region-4 risk register. Local provider integrations start immediately (longest lead time: sandbox access, certification). A launch cell (4 Payments BE, 1 FE, 1 QA) with one DRI. Weekly go/no-go risk review with the CEO. I negotiate **scope, not date**: minimum viable provider set and 2–3 pilot brands at launch; the rest in weeks 11–13.
3. **Weeks 2–4 — incident process and on-call rotation** (§3).
4. **Weeks 3–10 — de-risk the two bus-factor systems** (§5).
5. **Weeks 6–10 — restore weekly releases**, gated on money-flow E2E coverage.
6. **Weeks 11–13 — stabilise region 4, quarter retro, plan Q+1.**

**Explicitly not doing this quarter:**
- No wallet rewrite, service extraction or platform migration — the biggest temptation at the worst time.
- No new money-moving features that bypass the new controls (PR #482 waits until it passes them).
- No bug-backlog burn-down: triage only money, regulatory and player-blocking bugs; park the rest visibly.
- CRM roadmap mostly frozen to protect T. — stakeholders hear it in week 2, not week 9.
- No squad reorg, no metrics tooling purchase, no broad test-automation programme.

## 3. Process changes (four)

| Change | Attacks | Working if |
|---|---|---|
| **Money-path change control.** CODEOWNERS on wallet/payments/ledger/reporting; 2 approvals, one from a 3-person money-reviewer rotation (S. trains D. + one other); 8-item checklist (tenant scope, idempotency, decimal, transaction/locking, ledger entry, retry behaviour, reconciliation impact, rollback). Then encode it: lint bans `parseFloat`/`Number`/`FLOAT` in money modules; one `ledger.credit/debit(tx, idempotencyKey)` primitive. | #1, #4, #5–9 | Money-path escaped defects → 0; review turnaround < 1 working day (no new bottleneck); S. no longer sole approver on most money PRs. |
| **Incident process.** SEV1–3 (any money discrepancy = SEV1); incident-commander role; cross-squad on-call with runbooks; blameless post-mortem within 5 days for SEV1/2; actions tracked in a weekly 30-min ops review. | Rising MTTR, recurrence, #10–11 | MTTR trending down; ≥80% of actions closed on time; no repeat root cause. |
| **Region launch playbook.** Region config as code, schema-validated, per-region secrets in a vault (never copied); automated go-live gate: real sandbox deposit + withdrawal per provider and a clean reconciliation run; go/no-go checklist owned by the DRI. | #3, region-3 overrun | Region 4 live with no SEV1 in its first 2 weeks; region 5 plan reuses the playbook. |
| **Shift quality left; small release train.** QA joins refinement to write test plans; engineers own unit + integration tests (real Postgres on money paths); QA automates E2E for the ~10 critical flows; release checklist ≤ 10 items, risk-tiered; weekly train by week 8. | Late regressions, slipping cadence | Regressions found in release QA ↓; 4 consecutive weekly releases. |

## 4. Metrics — six, weekly with leads, monthly with the CEO

1. **Money incidents (count, value) + reconciliation breaks detected.** The board commitment; breaks are the leading indicator — more breaks early means detection works.
2. **Change failure rate** (deploys causing hotfix/rollback/incident). What review and testing actually catch.
3. **MTTR by severity.** Whether incident process and runbooks work.
4. **Deploy frequency / lead time.** Batch size — the cure for the late-regression spiral.
5. **Region-4 milestone confidence.** Burn-up on concrete milestones (provider certified, config validated, smoke green) with weekly RAG. Replaces the untrusted burndown.
6. **Bus factor.** People able to independently handle a wallet or pipeline incident; target ≥ 3 each by day 90.

Not measured: velocity, story points, individual output — gameable and silent on money safety.

## 5. People

- **S. (staff, wallet)** — biggest asset, biggest risk. Week 1: listen — load, frustrations, retention risk, comp. Then direct feedback with concrete examples: harsh reviews are *why* nobody else learns the wallet. Reframe his scope as **owner of money correctness, measured by others' capability**: trains two money reviewers, writes the wallet invariants doc, designs the ledger primitive. Protect him from reviewing everything; challenge him on #482 (Part B).
- **D. and two mids** — D. becomes the second wallet expert through deliberate pairing and the reviewer rotation. One mid is region-4 launch DRI — a stretch, with weekly support from me. Visible growth paths keep mids.
- **Three juniors** — no solo money-path changes; paired with mids; shadow on-call. Mentoring set up with L.
- **T. (CRM & Data)** — protect. Pipeline runbooks pair-written in week 2; off solo on-call by week 3 (data engineer + one BE shadowing); I triage the CRM backlog with product and say "no" for him.
- **L. (Web lead)** — formalise the mentoring they already do (engineering-wide onboarding/mentoring owner) and discuss whether group lead is their next step. Co-design the release checklist with them — L.'s buy-in moves the room. Web lends one FE to region 4 (localisation, back office).
- **QA (2)** — the problem is process, not headcount (1:10 is normal). Shift from gatekeepers to quality coaches and E2E owners; one embedded in the launch cell.
- **Two open headcounts** — I own hiring personally and first find why they sat 3 months (bar, comp, sourcing). Hires won't land in time for region 4, so hire for structure: **(1) senior BE, Payments** — S.'s second; **(2) senior BE/data engineer** — T.'s second. If integration capacity is short by week 3, a time-boxed contractor with local-provider experience, never on wallet core.
- **Me** — weeks 1–2: 1:1s with all 22, skip-levels with juniors, CEO, compliance, support and product. Publish this plan in week 3, revised with their input.

---

# Part B — PR #482 review

## B1. Review as posted

> **Requesting changes — please don't merge tomorrow.**

> D., thanks — clearly structured and quick to review. I'm blocking because this job moves real money and, as written, it can pay members multiple times, pay members of other brands, and overwrite concurrent balance changes. Each is a money incident. None is hard to fix; happy to pair on it today. Items 1–5 block merge; 6–8 block the first production run.

1. Red **`cashbackService.ts:78/86/101` — `accountId` is never used.** `Member.findAll()`, the bets and wallet queries all span every brand in the region. Ops runs this once per brand, so every member gets paid once *per brand*. Scope all three queries by `accountId`, with a test.
2. Red **No idempotency.** `weekStart` is a rolling `now − 7d`, and nothing prevents a second run (cron retry, HTTP timeout + retry, double click) from paying again — the same class as our duplicate-callback incident. Use a fixed period (`week_start` = period start in an agreed timezone), `UNIQUE (account_id, member_id, week_start)`, and treat a conflict as "already paid".
3. Red **`:106–119` — not atomic, lost update.** Read-modify-write of `balance` outside a transaction: a bet settling between `findOne` and `save()` is overwritten. If the payout or ledger insert fails after `save()`, the balance has moved with no ledger entry, and the `catch` hides it. Balance, payout row and `WalletTx` in one transaction with a row lock or `balance = balance + :amount` — ideally via the existing wallet credit path.
4. Red **Float money.** `parseFloat`, `* 0.05`, `FLOAT` column and `String(cashback)` give binary rounding and can write values like `"1e-7"` to the ledger — the class of incident #4. `dec()` end to end, `DECIMAL(36,18)`, and explicit rounding per currency (agreed with finance).
5. Red **`/admin` has no visible authN/authZ.** It needs auth and an audit record (who, when, which brand).
6. Orange **Unsettled bets.** If `payout` is null on open bets, `parseFloat(null)` is `NaN`, `NaN <= 0` is false, and we write `"NaN"` into a balance (Postgres `numeric` accepts it). Use settled bets only, by settlement time; confirm void/cancelled handling.
7. Orange **Eligibility.** Self-excluded, closed or blocked members? Cash or wagering-locked bonus, per region? I'll get compliance sign-off.
8. Orange **Execution model and failures.** A synchronous request doing 3–4 queries × 40k members will exceed timeouts (→ retry → #2) and hold DB connections — the shape of the withdrawal-batch incident. Aggregate net loss in one `GROUP BY` per brand, process in chunks as a background job, return a job id, and record/alert on failures instead of `console.log` + `ok: true`.
9. Yellow **Schema.** `cashback_payouts` needs `account_id`, `wallet_id`, currency, `wallet_tx` id, index on `member_id`; `WalletTx` should reference the payout. Which wallet if a member has several currencies?
10. Yellow **Tests** mock the DB, so they can't catch 1–4. Add integration tests on real Postgres: brand scoping, double run = one payout, concurrent settlement, decimal precision, null payout.
11. Yellow **Rollout.** Feature flag plus a dry-run that outputs the payout file so finance can check totals; first real run on one brand.

> I'll re-review with a second money reviewer once 1–8 are addressed.

## B2. The conversation

**Sequence.** Before posting I message both: "Looked at #482, I have blocking concerns, posting now — can we talk at 2?" Nobody learns about a block from a notification.

**D. (author).** 30-minute walkthrough, framed as a system failure, not a personal one: two years in, D. followed the codebase's patterns and their reviewer's advice ("keep it simple"). D. owns the fix, with pairing; I ask whether promo-date pressure shaped scope. Goal: D. leaves confident and becomes a money reviewer who catches this next time.

**S. (approver).** Private 1:1, curiosity first: "What did you look at? What was your load that day?" Then direct: his approval is our last line on money; missing tenant scoping and idempotency are exactly what he holds juniors to, and the asymmetry — brutal on juniors, "LGTM" on a trusted mid — tells me he's overloaded and reviewing people, not code. Not a performance issue yet; a design flaw in how we use him. I ask him to co-write the money-review checklist and train the rotation — leverage, not gate.

**So it doesn't reach me next time.** I can't be the safety net. I change the system: (1) CODEOWNERS on money paths with two approvals, one from the rotation; (2) checklist in the PR template; (3) guardrails in code — lint ban on float in money modules, a query helper that requires `accountId`, one idempotent `ledger.credit()`; (4) Postgres integration tests in CI; (5) every payout job ships behind a flag with a dry-run. I run a blameless "how did this pass review" with the squad and track money-path escaped defects.

---

# Part C — Two situations

## C1. Saturday 01:40 — balances too high

**First 15 minutes.** Declare a SEV1 money incident. I'm incident commander: I coordinate and decide; I don't debug code I don't know. Page S. and on-call, open a channel and bridge, name a scribe (timestamped log for regulator and post-mortem).

**CEO, within 10 minutes:** "Possibly yes. Some region-2 balances are inflated and still moving. It's top severity; S. and I are on it. We may pause withdrawals in region 2 to stop money leaving. Next update 02:30, then hourly." No guessed numbers.

**Contain before diagnosing.** Inflated balances only become real losses when withdrawn, so my first decision, taken with S. in minutes, is to **pause region-2 withdrawals** — reversible, player impact acceptable against open-ended loss. Then S. checks what changed: last deploy, scheduled jobs (cashback?), provider callback rates/duplicates. Kill the trigger (feature flag, stop the job, roll back) rather than patch at 2am. Snapshot the ledger and DB; no manual balance edits overnight.

**Who does what.** S.: root cause and stop. On-call/D.: quantify — wallets affected, delta, who has already withdrawn — using a ledger-vs-balance query. Support lead: holding message and macro. Me: decisions, CEO, comms. With T. on leave, the data engineer covers pipeline/reporting questions.

**Morning (08:00).** Notify compliance — regulatory reporting deadlines may apply — and account managers for affected brands. Hand over to a fresh pair; S. sleeps once stable. Written summary to the CEO: impact, exposure, actions, unknowns.

**Remediation.** Correct balances only through auditable ledger reversal entries, reviewed by two people and signed off by finance; withdrawn amounts handled case by case with brand and legal. Lift the withdrawal pause after reconciliation is clean.

**By hour 24.** Fix verified, reconciliation clean, post-mortem scheduled within 5 days, actions tracked in the weekly ops review.

## C2. Week 6 — "process theater"

**The next 60 seconds.** No defensiveness, no rank-pulling, no debate in front of the room:

"Fair challenge, and I'd much rather hear it out loud than have it quietly ignored. The checklist exists because of incident #1 and region-2 launch day — those cost us more than any checklist will. So it stays for this release. But if items aren't earning their place, they should go. L., let's take 30 minutes today and fix it; anyone else with a specific item, bring it. Let's continue planning."

Acknowledge, hold the line, open a path to change it. "Just ignore it" can't stand in a regulated money business — but disagreement can be productive.

**After — same day, 1:1 with L.** Curiosity first: what exactly slows them down? L. may be right: a payments-grade checklist applied to front-end copy changes *is* theater. I bring data (time added per release, what it caught) and ask L. to co-own a risk-tiered revision: light path for Web/back office, full path for money. Ship the revision within the week, credited to L.

**Then the behaviour, separately and directly:** "Challenge me any time, including in public. Telling the squads to ignore a control isn't OK — bring that to me first." Agree how we disagree in future.

**Look at myself.** If my most respected lead felt unheard by week 6, I didn't involve them early enough. L. joins the monthly plan review, and I ask leads whether the plan is too heavy four weeks before launch — and cut what isn't pulling weight.

**Follow-through.** Watch whether the revised checklist gets used and whether release lead time moves. If L. keeps undermining agreed decisions after this, it becomes a performance conversation — I don't expect it to.

---

> To structure the plan, to cross-check PR #482 for defects I might have missed, and to tighten wording to the page/word limits. All recommendations were reviewed, edited and are owned by me; I can defend each in interview.
