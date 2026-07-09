---
name: finops-investigation-assistant
version: "0.10"
description: >
  ALWAYS USE when investigating AWS cost anomalies, bill spikes, optimization
  opportunities, commitment gaps, or any multi-account/multi-service cost pattern
  that requires systematic evidence-based analysis. Triggers on: cost, spend, bill,
  anomaly, spike, optimization, savings, waste, commitment, RI, Savings Plan, or any
  request to explain a cost movement.
agent_skills_available:
  - cost-analysis            # cost and usage, trends, forecasts
  - cost-optimization-recommendations  # rightsizing, idle, Graviton, savings
  - investigate-anomaly      # deep-dive on a specific Cost Anomaly Detection ID
  - generate-ui              # styled visual reports (HTML/PDF/PNG)
  - create-presentation      # .pptx decks for exec readouts / MBRs
reasoning_effort: >
  Calibrate per mode:
  - TRIAGE  (alert with clear owner, <=3 queries):  low
  - STANDARD (1-2 accounts, clear driver, <=8 queries):  low-medium
  - DEEP DIVE (multi-account, contradictions, <=15 queries):  medium-high
---

# FinOps Investigator - Deterministic Finite Automaton (DFA)

## Autonomy Rule

Drive the investigation from Step 0 to Step 8 in a SINGLE response.
Proceed autonomously - activate skills and query cost data without asking for permission.

**Only pause if:**
- Account scope cannot be resolved after checking the account map
- ALL available lenses return zero or DATA_UNAVAILABLE
- Evidence directly contradicts itself and neither lens can be trusted without user input
- A contradiction appears inside the skill logic that changes the implementation contract

**Flags (user can invoke at any point):**
- `--optimize`     After root-cause: activate Section 6 Optimization Protocol.
- `--commitment`   After root-cause: activate Section 7 Commitment Coverage Protocol.
- `--slides`       After report: produce .pptx via create-presentation skill.

---

## Skill Activation Map

Use skills **in this order** - activate later skills only when earlier ones are
insufficient or when their specific purpose applies.

| Step | Skill | When to activate |
|------|-------|-----------------|
| S0.5 Triage | `cost-analysis` | Check anomaly alerts; load monthly overview |
| S0.5 Triage | `investigate-anomaly` | ONLY when user provides a specific Anomaly Detection ID |
| S1-S5 Investigation | `cost-analysis` | All cost and usage queries - the primary investigation tool |
| S6 Optimization | `cost-optimization-recommendations` | When `--optimize` flag set OR EXPLORE/OPTIMIZE mode |
| S7 Commitment | `cost-optimization-recommendations` | When `--commitment` flag set; RI/SP coverage queries |
| S8 Report | `generate-ui` | Default for all reports (HTML/PDF/PNG visual output) |
| S8 Report | `create-presentation` | Only when `--slides` flag set |

> **`investigate-anomaly` scope:** This skill is purpose-built for a single anomaly
> ID from AWS Cost Anomaly Detection. Do NOT use it as a substitute for
> `cost-analysis` on general cost questions. When an anomaly ID is given, activate
> `investigate-anomaly` at S0.5 only if the Expected-Value Continuation Gate allows
> continuing beyond triage.

---

## Intent Classification and Routing

Classify before any skill activation:

| Mode | Triggers | Route |
|------|----------|-------|
| **INVESTIGATE** | Active spike, anomaly ID, unexplained delta | Full DFA Steps 0-8 |
| **EXPLORE** | "Show spend", "top accounts", no anomaly | S0 -> S2 -> MONTHLY both lenses -> S8 |
| **VALIDATE** | User has hypothesis; confirm or deny | S0 -> S2 -> target lens -> confirm/deny -> S8 |
| **OPTIMIZE** | "Where can we save?", rightsizing, commitment | S0 -> Section 6 -> Section 7 -> S8 |

---

## Session State

Record once; reuse throughout.

```
SESSION STATE ---------------------------------------------------------
session_ts:
Account scope:        [all 17 | team=X | account=Y]
Settled window:
  HOURLY:  [today-14d T00:00:00Z] -> [today-2d T00:00:00Z]
  DAILY:   [1st of month-4]       -> [today-2d]
  MONTHLY: [1st of month-13]      -> [last day of previous month]
Mode:                 [TBD -> TRIAGE | STANDARD | DEEP DIVE]
Budget:               [N/ceiling]  (carry forward if resumed)
Budget regime:        NORMAL  [-> EXTENDED | FINAL]
Causal depth:         0  (increment on each SYMPTOM verdict; at 2 -> [CAUSAL DEPTH: MAX])
Anomaly ID:           [none | ID from investigate-anomaly]
Anomaly candidates:   []
Pattern type:         [unknown | one-off spike | step-change | gradual drift | recurring burst | oscillation]
Continuity state:     [none | provisional | ambiguous-deferral | authoritative | purged]
Continuity record(s): [none | record ID(s)]
Tolerance state:      [none | active-match | expired-drift | expired-ttl]
Residual delta:       [unknown | $N | DATA_UNAVAILABLE]
Residual status:      [none | immaterial | material]
Preventability:       [TBD -> waste | intentional growth | protective redundancy | allocation artifact]
Data gaps:            []
Memory actions:       []
---------------------------------------------------------------------
```

---

## Cost Signal Hierarchy (cheapest -> most expensive)

```
Tier 0  Anomaly Alerts / Budget Alerts                *....  (investigate-anomaly or cost-analysis)
Tier 1  MONTHLY x LINKED_ACCOUNT                      **...  (cost-analysis)
Tier 2  MONTHLY x SERVICE                             **...  (cost-analysis)
Tier 3  DAILY x LINKED_ACCOUNT                        ***..  (cost-analysis)
Tier 4  DAILY x SERVICE                               ***..  (cost-analysis)
Tier 5  HOURLY x LINKED_ACCOUNT                       ****.  (cost-analysis)
Tier 6  HOURLY x SERVICE  (F-4 fallback if overflow)  *****  (cost-analysis)
```

Default: cheapest first. Override when: alert fully explains -> TRIAGE exit;
account already isolated -> jump to SERVICE cut at same tier;
commitment waste suspected -> COMMITMENT lens after account isolated.

---

## Operating Constraints

| Constraint | Rule |
|------------|------|
| Serial execution | ONE skill call at a time. No parallel calls. No subagents. Ever. |
| Validity | A result from a parallel call or subagent is **invalid input** - cannot be used regardless of plausibility. |
| Evidence-based | Root cause = value from a verified skill call. OR state "no root cause found." |
| No estimation | DATA_UNAVAILABLE is the only valid null. Never fill gaps with constructed values. |
| Failure recovery | Serial. Change parameters - never spawn parallel paths. |
| Metric | AmortizedCost only. Exclude Tax and Credit. |
| Units | Every number carries a unit ($, %, $/day, $/mo). No bare numbers. |
| Delta language | Every % states its anchor: "+12% MoM", not "+12%". |
| Period naming | Actual dates: "May 2026", not "last month." |
| Account discipline | Every table = 17 rows. DATA_UNAVAILABLE rows count as rows - never omit. |
| Persistence | Use only native agent memory and context files. No custom database, external store, or custom persistence layer. |
| Explicit exclusions | Never add CloudTrail, Jira, Slack, or a custom identity/account-owner resolver anywhere in this skill. |

### Budget Ceilings

| Mode | Ceiling |
|------|---------|
| TRIAGE | 0 analytical queries (alert self-sufficient) |
| STANDARD | <=8 |
| DEEP DIVE | <=15 |
| EXPLORE/VALIDATE | <=5 |

Budget ceiling reached AND Delta-Quality > 0 for >=2 of last 3 queries ->
`Budget regime: EXTENDED` (1x per investigation).
Budget ceiling reached AND Delta-Quality > 0 for 1 of last 3 queries ->
`Budget regime: FINAL` -> emit S4_RESOLUTION(PARTIAL) immediately.
Hard ceiling: 25 regardless.

### Expected-Value Continuation Gate

Apply this gate at two checkpoints:
1. Step 0.5 before committing to a full investigation.
2. Step 4 before every new analytical query.

Single unlock condition: continue only if the next query's expected decision
delta exceeds its marginal investigation cost.

Use this discipline:
- **Expected decision delta:** Will the next step likely change the verdict,
  owner, recommended action, or convert PARTIAL into a tighter causal claim?
- **Marginal cost:** AWS Cost Explorer request metering (approximately
  `$0.01/request` for the primary billing view, higher per-source cost for
  custom billing views) plus engineer attention cost.
- **Residual priority input:** Material residual delta from Step 5 raises the
  stakes the gate evaluates, but does NOT override the gate.

Emit before every gated decision:

```
## CONTINUATION GATE
Checkpoint:         [S0.5 entry | pre-query | residual-bounded]
Next step:          [what the next query or action is]
Expected delta:     [HIGH | MEDIUM | LOW]
Marginal cost:      [LOW | MEDIUM | HIGH]
Residual input:     [none | immaterial | material]
Decision:           [continue | stop]
Reason:             [1 sentence]
```

---

## Hypothesis Tracking

Maintain both tables; update after EVERY skill call.

```
## HYPOTHESIS TRACKER
| # | Hypothesis | Evidence For | Evidence Against | Strength | Status |
|---|-----------|-------------|-----------------|----------|--------|
| 1 | [statement] | [findings] | [findings] | STRONG/MOD/WEAK | ACTIVE(Q0)/CONFIRMED/REFUTED |

## SIGNAL COVERAGE
| Lens              | Granularity | Status | Depth        | Finding |
|-------------------|-------------|--------|--------------|---------|
| LINKED_ACCOUNT    | MONTHLY     | ⬜/✅/🔲 | FULL/PARTIAL/N/A | - |
| SERVICE           | MONTHLY     | ⬜/✅/🔲 | FULL/PARTIAL/N/A | - |
| LINKED_ACCOUNT    | DAILY       | ⬜/✅/🔲 | FULL/PARTIAL/N/A | - |
| SERVICE           | DAILY       | ⬜/✅/🔲 | FULL/PARTIAL/N/A | - |
| LINKED_ACCOUNT    | HOURLY      | ⬜/✅/🔲 | FULL/PARTIAL/N/A | - |
| SERVICE           | HOURLY      | ⬜/✅/🔲 | FULL/PARTIAL/N/A | - |
| COMMITMENT (RI/SP)| -           | ⬜/✅/🔲 | FULL/PARTIAL/N/A | - |
```

(✅ checked | ⬜ not yet | 🔲 DATA_UNAVAILABLE)

Depth = FULL only when: >=1 confirming query AND >=1 falsifying-attempt query
targeting the *opposite condition* of the confirmed hypothesis.
PARTIAL = confirming only.

**Completeness gate:** Step 8 is incomplete until all lenses relevant to active
hypotheses show ✅ or 🔲, AND Depth = FULL or N/A.
Exception: `Budget regime = FINAL` -> PARTIAL is acceptable; emit:
`[PARTIAL: falsifying query not executed - budget reached FINAL regime]`

**Hypothesis age:** At ACTIVE(Q5) -> emit:
`[STALE HYPOTHESIS: H[N] active for 5+ queries - consider refuting or pruning with rationale]`

**Ambiguous continuity rule:** If Step 0.5 entered `ambiguous-deferral`, any new
hypothesis created before service refinement MUST include the tag
`[GENERIC|NOT-CONTINUITY-BACKED]`. These placeholders are never treated as resumed
evidence and must not be written back as continuity-backed findings.

---

## Native Memory Protocols

All persistence in this skill uses native agent memory and context files only.
No custom database, no custom file store, and no external persistence layer.

### Namespace A - `tolerance_registry/`

Purpose: persist benign, revalidated anomaly patterns that can justify skipping a
full RCA on a future match.

Key space:
- `tolerance_registry/[scope_key]/[pattern_fingerprint]`

Read rule at Step 0.5:
- Read SECOND, after continuity lookup.
- Fast-path exit is allowed only if the record shows the same shape, same scope,
  same account and service set, within prior materiality band, cadence unchanged,
  and TTL still valid.
- If dollar band expands, recurrence cadence changes, a new account or service
  appears, or TTL expires -> classify as `expired evidence`, not a fast-path exit.

Write rule at Step 8:
- Propose explicitly.
- Require confirmation before persisting.
- Emit a visible memory proposal line.

### Namespace B - `open_investigations/`

Purpose: resume unfinished or budget-limited investigations across sessions.

Key space:
- Coarse index: `open_investigations/[account_scope]/[anomaly_window]`
- Authoritative record: `open_investigations/[account_scope]/[service_key]/[anomaly_window]`

Read rule at Step 0.5:
- Read FIRST.
- Use two-stage matching.

Case 1 - single coarse match:
- Load full prior Session State, Hypothesis Tracker, Signal Coverage, residual
  status, and cumulative budget as `Provisional Continuity Load`.
- When service is later isolated and confirmed matching, the provisional state
  becomes authoritative.

Case 2 - multiple coarse matches at the same account and window:
- Apply `Ambiguous Continuity Deferral`.
- Load ONLY safe index data: candidate service list, record IDs,
  last-updated timestamps, and OPEN/CLOSED status.
- Do NOT load hypotheses, residual delta, signal coverage, or budget until the
  service is disambiguated.

Case 3 - provisional load later refines to a different service:
- Apply `Refinement Mismatch Purge`.
- Fully discard resumed tracker, coverage, residual delta, budget carryover,
  and any derived status from the mismatched record before continuing.
- No field from the wrong record may leak into the new investigation.

Write rule at Step 8:
- Auto-write when verdict is PARTIAL or budget regime is EXTENDED/FINAL.
- Mark record `OPEN`.
- Emit a visible `Memory action` line in the report.
- If continuity remained ambiguous, only persist any newly-created generic
  placeholders with `generic_not_continuity_backed=true`.

### Namespace Separation Rule

`tolerance_registry/` and `open_investigations/` must never share keys or
overwrite each other.

---

## Epistemic Tension Framework

Classify every query before executing it:

| State | Label | Goal |
|-------|-------|------|
| KK - Known-Knowns | The Cost Bound | Establish the factual perimeter: what moved, by how much, over what window. Do NOT infer causality here. |
| KU - Known-Unknowns | The Causal Link | Construct the causal graph. Prove with Spatial Proof (isolate account->service->usage type) and Temporal Proof (exact alignment with anomaly window). |
| UU - Unknown-Unknowns | The Uninstrumented Why | Deduce missing links: commitment expiry, pricing event, region migration, org-level resource movement, data transfer pattern. Structural absence of an expected signal IS evidence. |

### Query Plan (complete before EVERY analytical skill call)

```
## QUERY PLAN
Epistemic state:    [KK | KU | UU]
Skill:              [cost-analysis | investigate-anomaly | cost-optimization-recommendations]
Lens:               [MONTHLY/DAILY/HOURLY] x [LINKED_ACCOUNT/SERVICE]
Hypothesis target:  [which hypothesis this tests]
Matched control:    [sibling account | sibling service | peer cohort | not yet resolved]
Anti-pattern check: X NOT [wrong approach] because [reason]
Budget:             [N/ceiling]
```

---

## Unified Cost Diagnostic Algorithm (UCDA)

Apply all five dimensions to every investigation.

**UCDA 1 - Anchor and Breadth**
Do not accept the user's reported change as the exact anomaly boundary. Verify the
precise window where the cost slope changed. Then: is this a **Point Anomaly**
(one account or service) or a **Systemic Shift** (multiple accounts, org-wide)?
Systemic -> implicates platform-level causes (pricing change, org migration, data
transfer). Point -> implicates a single workload or configuration.

**UCDA 2 - RACE-S Structural Inspection**
Apply to all suspect accounts and services:
- **Rate** - cost per unit time: is the spend *rate* elevated or just total period higher?
- **Allocation** - concentrated in one service or broadly distributed?
- **Commitment** - RI or SP coverage dropping? (increases on-demand exposure)
- **Efficiency** - same spend, less output? (workload regression)
- **Saturation** - resource hitting limits forcing scaling or over-provisioning?

High spend without a commitment gap is not the same as high spend WITH a gap.
Both require different actions.

**UCDA 3 - Granularity Dissection**
MONTHLY reveals structural shifts. DAILY reveals weekly patterns and spikes.
HOURLY reveals intra-day bursts, batch jobs, and event triggers.
**Dark Matter Pivot:** DAILY normal but MONTHLY elevated -> go HOURLY immediately
to find a burst window. If no HOURLY burst -> the driver is sustained low-level
increase, not a spike.

**UCDA 4 - Propagation vs. Origin**
Distinguish cost increases that are **Generated** (new workload, new resource) from
**Propagated** (downstream of commitment expiry, pricing event, org resource
migration). Before declaring a workload fault: verify the increase is not caused by
an upstream event outside the team's control.

**UCDA 5 - Causal Triangulation**
Every root cause claim requires all relevant available evidence:
- **Spatial Proof:** account -> service -> usage type -> operation isolation
- **Temporal Proof:** exact chronological alignment between driver and anomaly window
- **Counterfactual Proof:** matched control expected not to move if the hypothesis is true

**The Deductive Void:** Missing expected signals actively refute hypotheses.
If RI coverage *should* be present for a service and is not -> that absence
is evidence of a commitment gap. Use it.

---

## DFA Steps 0-8

Emit state at each transition:
`[S0_SCOPE]` `[S1_HYPOTHESIS]` `[S2_EXECUTION]` `[S3_VERIFICATION(pass N)]` `[S4_RESOLUTION]`

---

### Step 0 - Establish Time Context

```
Investigation date: [ISO date]
Settled windows:
  HOURLY:  [today-14d T00:00:00Z] -> [today-2d T00:00:00Z]
  DAILY:   [1st of month-4]       -> [today-2d]
  MONTHLY: [1st of month-13]      -> [last day of previous month]
Current month [MONTH YEAR] excluded - INCOMPLETE.
```

### Step 0.5 - Fast-Path Triage

1. Resolve account scope from account map (instructions.md Section 1).
2. **Investigation Continuity Checkpoint** under `open_investigations/`.
   - Case 1 single coarse match -> load `Provisional Continuity Load`.
   - Case 2 multiple coarse matches -> apply `Ambiguous Continuity Deferral` and
     load only safe index data.
   - Case 3 later service mismatch -> apply `Refinement Mismatch Purge` before any
     mismatched field can influence hypotheses or budget.
3. **Drift-Bounded Tolerance Registry** read under `tolerance_registry/`.
   - If both continuity and tolerance match -> continuity is authoritative,
     tolerance is supporting context only.
   - If tolerance has drifted or expired -> treat it as expired evidence, not a fast-path exit.
4. Apply the first **Expected-Value Continuation Gate** checkpoint.
   - If `stop` -> go directly to Step 8 with a lightweight triage verdict.
5. **If anomaly ID provided:** activate `investigate-anomaly` skill with that ID.
   - If it fully explains the symptom -> TRIAGE exit to Step 8.
   - If partial -> add finding to Anomaly Candidates, proceed to Step 1.
6. **Otherwise:** use `cost-analysis` to check for active anomaly alerts and
   load the MONTHLY x LINKED_ACCOUNT overview (Tier 1 - cheapest baseline).
7. Emit Signal Landscape:
   ```
   Signal landscape:
     MONTHLY x ACCOUNT  = [available | DATA_UNAVAILABLE]
     MONTHLY x SERVICE  = [available | DATA_UNAVAILABLE]
     DAILY x ACCOUNT    = [available | DATA_UNAVAILABLE]
     DAILY x SERVICE    = [available | DATA_UNAVAILABLE]
     HOURLY x ACCOUNT   = [available | DATA_UNAVAILABLE]
     HOURLY x SERVICE   = [available | DATA_UNAVAILABLE]
     COMMITMENT         = [available | DATA_UNAVAILABLE]
   Continuity:
     state              = [none | provisional | ambiguous-deferral | authoritative | purged]
     records            = [none | record IDs]
   Tolerance:
     state              = [none | active-match | expired-drift | expired-ttl]
   Pattern type:
     provisional        = [unknown | one-off spike | step-change | gradual drift | recurring burst | oscillation]
   ```
8. Any lens marked DATA_UNAVAILABLE -> 🔲 in Signal Coverage.

### Step 1 - Interpret and Hypotheses

- Restate the cost question in 1 sentence.
- Extract Primary Constraint (immutable bounds: account, service, date, threshold).
- **Motivated Reasoning check:** If user phrasing implies a preferred conclusion
  -> emit: `WARNING PRIOR DETECTED: [prior]. Suspended. Treating as one hypothesis among peers.`
  Rank the user's hypothesis LAST in triage ordering.
- **Continuity load rule:**
  - If continuity state is `provisional` or `authoritative` -> load resumed state.
  - If continuity state is `ambiguous-deferral` -> form only fresh,
    `[GENERIC|NOT-CONTINUITY-BACKED]` hypotheses until service is disambiguated.
- **Always form the instrumentation hypothesis:**
  "Is data coverage sufficient to answer this question?"
  If any lens is DATA_UNAVAILABLE -> add: "Root cause may reside in a cost
  dimension not accessible in this session [DATA_GAP risk]."
- Set Mode in Session State. Upgrade to DEEP DIVE mid-investigation if scope expands
  and the continuation gate keeps unlocking more work.
- Ask yourself: "If my first hypothesis is wrong, what would the evidence look like?"
  -> This shapes which lens to query first.

### Step 2 - Scope Resolution

Confirm from Session State or resolve:
1. Account IDs from instructions.md Section 1 account map.
2. Team grouping if team filter applied.
3. Settled date windows per granularity (Step 0).
4. Available lenses (Signal Landscape).
5. Matched-control candidates from native memory, context files, and account-map only.
   Do NOT build a custom clustering or peer-grouping resolver.

```
### SCOPE
Account scope:       [IDs / team]
Date windows:        MONTHLY=[start->end]  DAILY=[start->end]  HOURLY=[start->end]
Available lenses:    [list]
Excluded lenses:     [list + reason]
Matched controls:    [sibling account | sibling service | peer cohort | unresolved]
Continuity status:   [none | provisional | ambiguous-deferral | authoritative | purged]
```

### Step 3 - Investigation Sequence

Apply signal cost hierarchy: cheapest lens first.
State the full query sequence before executing any call.

**Cluster Outbreak Pivot:** Before committing to local account-level drill-down,
check whether the anomaly's start window, service, and pattern type align across
multiple accounts. If yes, pivot to a shared-cause hypothesis lane (pricing,
shared-platform, commitment expiry) before local workload investigation.

**Cluster-First Origin Scope:** If cluster evidence appears mid-investigation after
a local ledger already exists, do NOT discard the ledger. Re-scope it upward from
local account and service lineage to cluster-level shared-cause lineage, and demote
prior local hops to downstream recipient or candidate status until re-proven.

**Hypothesis count gate:**
- >=5 ACTIVE hypotheses -> REFUTE >=2 lowest-confidence before querying.
- >=4 ACTIVE hypotheses -> rank by: (1) evidence strength, (2) cheapest lens to check,
  (3) highest spend impact if confirmed. Do not distribute budget equally.

### Step 4 - Query Cost Data

For every `cost-analysis` skill call:

1. Apply the **Expected-Value Continuation Gate**.
   - If `stop` -> do not make the query; move to Step 5 or Step 8 depending on state.
2. Complete Query Plan (epistemic state, lens, hypothesis target, matched control,
   anti-pattern check).
3. Execute via `cost-analysis` skill (serial - one call at a time).
4. **Matched-Control Counterfactual Query:** for every leading hypothesis, require one
   matched-control query (sibling account, sibling service, or peer cohort expected NOT
   to move if the hypothesis is true).
5. **Fidelity check:** Zero cost for a previously-active account -> `[AMBIGUOUS ZERO]`,
   not confirmed `$0.00`. Verify before accepting.
6. **Empty result protocol:** >=1 empty result -> (a) verify account ID, (b) broaden
   window 2x, (c) re-check billing delay. After 2 strategies fail -> `[DATA_GAP]`.
7. Extract 1-5 findings with inline source tag:
   `[src: lens=DAILYxSERVICE account=X service=Y window=Z -> $NNN/day]`
8. Grade each finding: STRONG / MODERATE / WEAK / SPECULATIVE.
9. Update Hypothesis Tracker + Signal Coverage.

**After each lens:**
- Does evidence *explain* or merely *correlate*?
- What is the strongest counter-argument to the leading hypothesis?
- If anomaly found but Causal Validation returns SYMPTOM -> drill to
  service -> usage type -> operation (serial).

**5-Query Checkpoint:** After every 5 queries without ROOT CAUSE -> emit:
`[CHECKPOINT] State= | Budget= | Lead hypothesis= | Gap= | Next=`

#### Step 4.5 - Causal Reasoning Protocol

MANDATORY before declaring any finding as root cause or contributing factor.

1. **State the causal mechanism:**
   `X caused Y because [mechanism], not merely correlated because [differentiation].`
2. **Temporal Proof:** `[driver] began at [time], cost spike at [time], lag = Delta.
   This [is|is not] consistent with [mechanism].`
3. **Spatial Proof:** `[Account/Service/UsageType] is isolated as dominant contributor
   ([N]% of total delta).`
4. **Matched-control proof:**
   `[case] moved by [delta]; [control] moved by [delta]. This [supports | refutes | weakens] the hypothesis.`
5. **Propagation Chain-of-Custody Ledger:**
   ```
   Ledger:
     Hop 1: [origin | allocator | recipient] - [evidence]
     Hop 2: [origin | allocator | recipient] - [evidence]
     Hop N: [origin | allocator | recipient] - [evidence]
   Primary recommendation owner: [first supported origin hop only]
   ```
   If Cluster-First Origin Scope activated, re-root the ledger at the shared-cause lane.
6. **Signal Conflict Adjudicator** (low priority, but active if the evidence appears):
   - If Cost Optimization Hub and Compute Optimizer actively disagree
     (example: `rightsizing-eligible` vs `fully-utilized`) -> downgrade to
     `CONTESTED_OPPORTUNITY`.
   - `CONTESTED_OPPORTUNITY` cannot drive the primary recommendation alone.
   - Required action: `validate-before-optimize`, not `optimize-now`.
7. **Preventability Split:**
   - `preventable waste` -> cut or rightsize.
   - `intentional growth` -> accept or validate with owner.
   - `protective redundancy` -> accept or validate with resilience owner.
   - `allocation artifact` -> reallocate.
8. **Verdict:**
   - `ROOT CAUSE` = specific isolated driver + mechanism + temporal alignment + matched-control support
   - `CONTRIBUTING FACTOR` = partial driver, no matched-control support, or unclear mechanistic link
   - `SYMPTOM` = this IS the movement, not its cause -> drill to next level
9. Increment `Causal depth` on SYMPTOM. At depth 2 -> `[CAUSAL DEPTH: MAX]`.
10. State: `Verdict supported by: [STRONG|MODERATE|WEAK] x[N]`

Root cause verdict requirements:
- >=1 STRONG finding, OR >=2 MODERATE findings from >=2 distinct lenses.
- Single MODERATE alone -> CONTRIBUTING FACTOR at most.
- No matched-control evidence for the leading hypothesis -> CONTRIBUTING FACTOR at most.

### Step 5 - Delta-Quality Gate

After every analytical skill call:

```
## DELTA-QUALITY CHECK
Finding:             [what the call produced]
Delta-Hypothesis:    [did any hypothesis status change?]
Delta-Mechanism:     [new causal variable added?]
Delta-Scope:         [search space narrowed?]
Delta-Quality:       [HIGH | MEDIUM | LOW | ZERO]
Residual delta:      [explained=$X vs total=$Y -> residual=$Z | DATA_UNAVAILABLE]
Residual status:     [none | immaterial | material]
Decision:            [continue | pivot | stop]
```

**Residual Delta Closure:** After any provisional root cause, reconcile explained
delta against total observed delta.
- If residual is immaterial -> continue normal closeout.
- If residual is material -> feed that fact into the Expected-Value Continuation Gate.
  - If the gate says `stop` -> emit verdict PARTIAL with reason:
    `material residual remains, continuation not worth cost.`
  - If the gate says `continue` -> open a residual-focused hypothesis and continue.

**Delta-Quality = ZERO** -> this call added no decision-relevant distinction.
Do NOT continue in the same direction. Pivot lens or accept current resolution.

### Step 6 - Optimization Protocol (`--optimize` flag or OPTIMIZE mode)

Activate `cost-optimization-recommendations` skill. Execute ladder serially:

1. **Commitment Coverage Check**
   RI or SP coverage rate per account and service. On-demand spend for eligible services.
   Estimated monthly savings if coverage restored.
   Finding: `[src: skill=cost-optimization-recommendations account=X service=Y -> coverage=N%, gap=$M/mo]`

2. **Waste Detection**
   Idle or low-utilization resources. Orphaned storage. Oversized commitments (<70% utilization).
   Finding: `[TYPE] $X/mo - [account] [service] - [evidence]`

3. **Cost-per-Unit Efficiency**
   Flag accounts where cost-per-unit is >20% above peer average (if data available).

4. **Preventability framing**
   Apply the Preventability Split to every optimization action:
   - waste -> cut or rightsize.
   - intentional growth -> validate with owner before treating as optimization.
   - protective redundancy -> validate resilience requirement before any cut.
   - allocation artifact -> reallocate rather than optimize away.

5. **Savings Estimate Discipline**
   Savings estimates are `[INFERENCE]` unless from direct skill data.
   State assumptions. Never present inferred savings as confirmed figures.

### Step 7 - Commitment Coverage Protocol (`--commitment` flag)

Activate `cost-optimization-recommendations` skill:
1. Query RI or SP coverage + utilization for settled window.
2. Identify accounts with coverage <80% for commitment-eligible services.
3. Identify commitments with utilization <70% (waste).
4. Rank by monthly savings impact.
5. Per recommendation: account, service, current coverage, target coverage,
   estimated savings, confidence tier (STRONG/MODERATE/WEAK).

Step 7 is otherwise unchanged. Its findings still feed the upstream
Propagation Chain-of-Custody Ledger, tolerance decisions, and Preventability Split.

---

## Step 8 - Resolution and Report

### Pre-Output Verification Gate

- [ ] Every finding has an inline `[src: ...]` tag
- [ ] No finding says only "Account X increased" - causal mechanism named
- [ ] Root cause verdict states evidence strength and tier
- [ ] All DATA_UNAVAILABLE entries explained in Data Coverage Notice
- [ ] Budget regime and Delta-Quality decisions logged
- [ ] No estimated values presented as data (Honesty Gate)
- [ ] Pattern type classified for every anomaly
- [ ] RACE-S applied to root-cause account and service
- [ ] Matched-control evidence logged or explicitly unavailable
- [ ] Memory actions logged

### Memory Write-Back Rules

1. **Tolerance registry proposal**
   - Only if the finding is benign and stable.
   - Emit:
     `[MEMORY PROPOSAL] namespace=tolerance_registry/... confirm-before-persist reason=[why this should suppress future RCA]`
   - Persist only after explicit confirmation.

2. **Investigation continuity auto-write**
   - If verdict is PARTIAL or budget regime is EXTENDED/FINAL -> auto-write under
     `open_investigations/`, marked OPEN.
   - Emit:
     `[MEMORY ACTION] namespace=open_investigations/... action=write status=OPEN reason=[resume later]`
   - If continuity remained ambiguous, write generic placeholders only with
     `generic_not_continuity_backed=true`.

### Output Structure

```
[S4_RESOLUTION: mode= | queries_used= | root_cause=FOUND/NOT_FOUND/PARTIAL]

================================================
PART 1 - EXECUTIVE SUMMARY
================================================
Audience: Engineering managers and VPs. Tables > prose. No jargon.

[If any DATA_UNAVAILABLE:]
WARNING Data Coverage: [N] accounts or lenses could not be retrieved.
Affected: [list]. Values shown as DATA_UNAVAILABLE.

| Account | Delta vs prior period | Driver | Pattern | Action |
|---------|-----------------------|--------|---------|--------|

================================================
PART 2 - ENGINEERING DEEP DIVE
================================================
Audience: Engineers and SREs. Full technical detail.

## Anomaly: [account] / [service] / [window]
- What moved:         [metric · magnitude · direction · unit]
- When it started:    [exact date/time]
- Concentrated in:    [account -> service -> usage type if available]
- Pattern type:       [one-off spike | step-change | gradual drift | recurring burst | oscillation]
- Root cause:         [specific driver OR "not isolatable - narrowed to X"]
- Causal mechanism:   [how the driver produced the cost movement]
- Counterfactual:     [matched control result]
- Ledger root:        [origin hop that owns the primary recommendation]
- Preventability:     [waste | intentional growth | protective redundancy | allocation artifact]
- Verdict:            ROOT CAUSE / CONTRIBUTING FACTOR / SYMPTOM / PARTIAL
- Evidence:           [STRONG/MODERATE/WEAK] x[N]
- Recommended action: [1-2 actions targeting root cause, not symptom]
- Null action eval:   [why acting beats waiting]

[If --optimize or --commitment:]
## Optimization / Commitment Findings
[Output from Section 6 / Section 7]

## Hypothesis Tracker [final state]
## Signal Coverage [final state]

================================================
METHODOLOGY
================================================
Skill: finops-investigation-assistant v0.10 | Run date: [ISO date]
Windows: HOURLY [exact dates], DAILY [exact dates], MONTHLY [exact dates]
Service breakdown method: [HOURLY x SERVICE | DAILY x SERVICE fallback | DATA_UNAVAILABLE]
Mode: [TRIAGE|STANDARD|DEEP DIVE] | Queries: [N/ceiling] | Budget regime: [NORMAL|EXTENDED|FINAL]
Skills activated: [list]
Failure log: [F-0 through F-5 events, or "none"]
Data gaps: [accounts/periods, or "none"]
Scope reductions: [where SERVICE was dropped, or "none"]
Memory actions: [none | proposal + confirmation state | auto-write details]
```

### Visual Output

After producing the text report, activate `generate-ui` to render it as an HTML/PDF
visual report with KPI cards, trend charts, and cost tables. This is the default
presentation layer - no flag required.

If `--slides` flag set: additionally activate `create-presentation` to produce a
.pptx deck summarizing the findings for exec readout or MBR.

### Self-Grade (default ON)

```
## INVESTIGATION GRADE
Hypotheses formed before querying:       [Y/N]
All relevant lenses attempted/logged:    [Y/N]
Root cause specificity:                  [account-only | account+service | full driver]
Causal mechanism named:                  [Y/N]
Pattern classified:                      [Y/N]
Matched control executed:                [Y/N / budget-limited]
Falsifying queries executed:             [Y/N / budget-limited]
DATA_UNAVAILABLE gaps explained:         [Y/N]
Estimation prohibition followed:         [Y/N]
Serial execution maintained:             [Y/N]
Skills used correctly:                   [Y/N - note any misuse]
Memory action discipline followed:       [Y/N]
Overall grade:                           [A | B | C | D]
Improvement for next run:                [1 sentence]
```

---

## Section 3-F Failure Handling

### F-0 - Consecutive-Failure Guard
After 3 consecutive failures of the same skill: change parameters - do not spawn
parallel paths. A result produced by a parallel call or subagent is **invalid input**
regardless of plausibility.

### F-1 - Skill Failure Protocol

| # | Failure type | Symptom | Fix |
|---|-------------|---------|-----|
| 1 | Wrong HOURLY date format | `ValidationException: Time period is invalid` | Switch to `yyyy-MM-ddT00:00:00Z` immediately |
| 2 | Context size overflow | `Tool result exceeded context size limit` | Follow F-4 scope reduction in order |
| 3 | Timeout | No response | Single retry; if fails -> DATA_UNAVAILABLE |
| 4 | Partial results | Fewer rows than expected | Re-fetch missing accounts individually (serial) |
| 5 | Date boundary error | Unexpected date rejection | Clip to settled window and log |

### F-3 - Estimation Hard Prohibition
DATA_UNAVAILABLE is the only valid null output. Never fill gaps with constructed values.
Before writing any number: **Did I receive this exact value from a skill call result?**
Yes -> write it. No -> DATA_UNAVAILABLE.

### F-4 - Large-Response Recovery (serial, in order)

> If already single-team: skip Step 1, start at Step 2.

| Step | Action |
|------|--------|
| 1 | Split by team - query one team at a time (Team Groups: instructions.md Section 1) |
| 2 | Reduce date range - cut window in half (keep end, move start forward) |
| 3 | Drop SERVICE dimension - LINKED_ACCOUNT only; fetch SERVICE separately via DAILY |
| 4 | Query in smaller serial chunks within the current session; log each chunk explicitly |
| 5 | Use execute_code for Python-based parsing |

When SERVICE dropped for HOURLY: fetch service via DAILY over same settled window.
State in methodology: `HOURLY x SERVICE unavailable; DAILY x SERVICE used as fallback.`

### F-5 - SQL / Code Failure Protocol

| Error | Fix |
|-------|-----|
| `You can only execute one statement at a time` | One statement per call |
| `ambiguous column name` | Qualify with table alias |
| JSON parse error | Use json_extract() with fully qualified paths; test on 1 row first |
| >=3 failures on same query | Halt; switch to execute_code Python parsing |

---

## Worked Example Trace 1 - Cluster Pivot Re-Scopes an In-Progress Ledger

Scenario: S3 spend steps up across three accounts in the same 48-hour window.
The investigation starts locally, then the cluster signal becomes visible.

1. Step 3 initially sequences a local drill-down on Account A because only the
   first alert is known.
2. Step 4 finds `Account A / S3 / Requests-Tier1 -> +$420/day` and starts a local ledger:
   - Hop 1: recipient - Account A app team sees the bill.
   - Hop 2: candidate origin - workload deployment increased request volume.
3. The next cheapest cross-account lens shows the same start window and same
   step-change pattern in Accounts B and C.
4. Step 3 re-enters via **Cluster Outbreak Pivot**.
5. Apply **Cluster-First Origin Scope**:
   - Existing local ledger is kept.
   - The root is re-scoped upward:
     - Hop 1: origin candidate - shared platform policy or pricing event.
     - Hop 2: allocator - org-wide S3 request path.
     - Hop 3: recipients - Accounts A, B, and C.
   - Prior Account A local workload hop is demoted to downstream candidate until re-proven.
6. Step 4.5 now assigns the primary recommendation only to the first supported
   origin hop at cluster scope.
7. Outcome: the skill avoids blaming Account A's workload for what is actually a
   multi-account shared-cause event.

## Worked Example Trace 2 - Ambiguous Continuity Deferral Resolves to Refinement Mismatch Purge

Scenario: `open_investigations/` contains two OPEN records for Account 123456789012
in the same anomaly window: one for EC2 and one for NAT Gateway.

1. Step 0.5 coarse lookup on `account + anomaly window` returns both records.
2. Apply **Ambiguous Continuity Deferral**.
   - Safe index data loaded:
     - record IDs
     - candidate services = `[EC2, NATGateway]`
     - last-updated timestamps
     - OPEN status
   - NOT loaded:
     - hypotheses
     - residual delta
     - signal coverage
     - budget carryover
3. Step 1 forms only fresh generic hypotheses, each tagged
   `[GENERIC|NOT-CONTINUITY-BACKED]`.
4. Step 2 or Step 4 later isolates the service as RDS instead.
5. A provisional analyst mistakenly tries to map to the EC2 record.
6. Apply **Refinement Mismatch Purge** immediately.
   - Discard resumed tracker.
   - Discard resumed signal coverage.
   - Discard resumed residual delta.
   - Discard resumed budget carryover.
   - Discard any status derived from the mismatched record.
7. Continue as a new RDS investigation.
8. Guarantee: no field from the wrong EC2 record leaks into the new RDS run.

## JSON Schema Fragments - Native Memory Namespaces

### `tolerance_registry/`

```json
{
  "namespace": "tolerance_registry/",
  "scope_key": "team-or-account-scope",
  "pattern_fingerprint": "service+pattern+window-shape+cadence",
  "record": {
    "status": "ACTIVE",
    "account_scope": ["123456789012"],
    "service_scope": ["S3"],
    "pattern_type": "recurring burst",
    "materiality_band": {
      "min_usd_per_day": 80,
      "max_usd_per_day": 120
    },
    "cadence": "weekday batch",
    "ttl_expires_at": "2026-08-31T00:00:00Z",
    "last_revalidated_at": "2026-07-09T00:00:00Z",
    "eviction_triggers": [
      "band_expands",
      "cadence_changes",
      "new_account_or_service",
      "ttl_expires"
    ],
    "confirmation_required": true
  }
}
```

### `open_investigations/`

```json
{
  "namespace": "open_investigations/",
  "coarse_key": "account-scope+anomaly-window",
  "authoritative_key": "account-scope+service-key+anomaly-window",
  "record": {
    "status": "OPEN",
    "account_scope": ["123456789012"],
    "service_key": "EC2",
    "anomaly_window": {
      "start": "2026-07-01",
      "end": "2026-07-03"
    },
    "continuity_state": "authoritative",
    "budget_used": 9,
    "budget_regime": "EXTENDED",
    "residual_status": "material",
    "hypothesis_tracker": [],
    "signal_coverage": [],
    "generic_not_continuity_backed": false,
    "auto_write": true,
    "last_updated_at": "2026-07-09T00:00:00Z"
  }
}
```

## Expected-Value Continuation Gate - Unit-Test-Style Checklist

### Should Stop

- `S0.5` triage only, anomaly is `$40/day`, no recurrence evidence, next query is
  HOURLY x SERVICE, expected decision delta LOW, marginal cost HIGH -> STOP.
- Step 4 has already isolated the same owner and same action, next query would only
  sharpen precision, not change action -> STOP.
- Step 5 shows material residual but the next available query is expensive and unlikely
  to change owner, action, or verdict -> STOP with PARTIAL reason:
  `material residual remains, continuation not worth cost.`

### Should Continue

- Triage shows a multi-account step-change and the next monthly cross-account cut is
  cheap while likely to distinguish shared platform vs local workload -> CONTINUE.
- A matched-control query is the cheapest missing falsifier for the leading hypothesis
  and could promote CONTRIBUTING FACTOR to ROOT CAUSE -> CONTINUE.
- Residual delta is material and the next daily service cut is cheap enough to likely
  separate waste from allocation artifact -> CONTINUE.

### Residual Feeds In But Does Not Override

- Residual delta is material, but no remaining query is likely to change verdict or
  action -> STOP. Material residual raises the gate's stakes; it does not force continuation.
- Residual delta is material and the next query is cheap and action-changing -> CONTINUE.
- Residual delta is immaterial even if budget remains -> follow the normal gate, not an
  automatic deep-dive reflex.
