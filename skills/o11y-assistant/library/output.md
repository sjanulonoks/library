# Output Format Templates

> Loaded by Step 8 after mode selection. Read ONLY the section matching the selected output mode.

---

## TRIAGE Format

```
<output_contract: TRIAGE>
Required sections (in order): status-line, root-cause, immediate-actions, ruled-out
Length: ≤8 lines total. No prose.
Root cause MUST reference: alert name + state + labels (alert-confirmed evidence; CAUSAL VALIDATION not required on this fast-path).
⚠️ TRIAGE assumes alert is causally sufficient. If the user asks "why" or challenges the verdict → escalate to STANDARD investigation (alerts describe symptoms, not always root causes).
Omit: contributing-factors, evidence-appendix, resolution-queries (→ use DEEP DIVE if needed).
</output_contract>

🔴 [Service]: [what happened in 1 sentence]
Root cause: [1 sentence] → grounded in: alert [name] state=[state] labels=[labels]
Immediate action: [1–2 bullets]
Ruled out: [hypotheses refuted]
```

---

## STANDARD Format

```
<output_contract: STANDARD>
Required sections (in order): summary, timeline, evidence, root-cause, immediate-action, ruled-out, unmapped-anomalies
Root cause section MUST embed the CAUSAL VALIDATION block verbatim (not by reference). Format:
  ## Root Cause
  [CAUSAL VALIDATION block verbatim — all fields including Verdict]
  Evidence summary: [1 sentence]
Optional: contributing-factors (≤2 bullets; include ONLY if ≥1 MODERATE finding is distinct from root cause and actionable), involved-architecture (list specific nodes discovered via breadcrumbs).
Omit: evidence-appendix (→ use DEEP DIVE for 3+ backend incidents).
ruled-out (required): for each REFUTED hypothesis, name the specific evidence that refuted it — 1 line max per hypothesis.
unmapped-anomalies (required): list any verified anomalies that DO NOT logically fit into the confirmed root cause or contributing factors.
</output_contract>
```
1. **Summary** — What / Impacted / Window
2. **Timeline** (chronological reconstruction from all evidence)
   ```
   [T-15m] Normal baseline (Metrics: error rate 0.1%)
   [T-5m]  Deployment detected (Annotation: deploy v2.3.1)
   [T-2m]  Error rate spike (Metrics: 0.1% → 4.7%)
   [T-0]   Alert fires (Grafana: "High Error Rate" → firing)
   ```
3. **Evidence** — Per-backend findings with Evidence Strength grades
   **Known unknowns:** [black boxes from CAUSAL VALIDATION Step 4.5 not resolved by investigation]
4. **Root Cause** — Evidence-based with causal validation (Step 4.5)
5. **Immediate Action** — 1–2 bullets (ONLY if root cause confirmed)
6. **Unmapped Anomalies** — List any observed anomalies that are NOT explained by the root cause. If none, state "None".

---

## DEEP DIVE Format

```
<output_contract: DEEP DIVE>
Required sections (in order): summary, timeline, evidence, root-cause, contributing-factors, involved-architecture, evidence-appendix, ruled-out, unmapped-anomalies
Optional: further-investigation (include ONLY if SPECULATIVE findings exist — state the signal that would confirm each one).
Contributing factors MUST have Evidence Strength grade.
ruled-out (required): for each REFUTED hypothesis, name the specific evidence that refuted it — 1 line max per hypothesis.
unmapped-anomalies (required): list any verified anomalies that DO NOT logically fit into the confirmed root cause or contributing factors.
Multi-service: if root cause is in upstream service X, prefix output header: "⛓️ [TargetService] ← upstream: [RootCauseService]"; root-cause section describes [RootCauseService].
[context-override: synthesis-breadth > retrieval-precision for 3+ backend investigations]
Retain full Signal Coverage Map + complete Hypothesis Tracker in output.
</output_contract>
```
Standard format PLUS:
7. **Contributing Factors** — What conditions enabled the failure? (with Evidence Strength grade)
8. **Evidence Appendix** — Queries used, raw findings, deeplinks
9. **Ruled Out** — Each REFUTED hypothesis with the specific evidence that refuted it (1 line each)
10. **Unmapped Anomalies** — List any observed anomalies that are NOT explained by the root cause. If none, state "None".
11. **Further Investigation** (optional) — SPECULATIVE findings only; for each, state what signal would confirm it.

---

## NO ANOMALY Format

```
<output_contract: NO ANOMALY>
Required sections (in order): status-line, checked, possible-explanations, ruled-out
Minimum query gate (MANDATORY): NO ANOMALY is FORBIDDEN if Signal Coverage shows ⬜ for any backend present in Signal Landscape.
  All available backends must show ✅ (checked healthy) or 🔲 (confirmed unavailable) — no ⬜ (unchecked) permitted.
  If any ⬜ remains → query the backend first OR explicitly mark it 🔲 with rationale before emitting NO ANOMALY.
Variant — INSTRUMENTATION_GAP: If Signal Coverage shows 🔲 for ≥50% of available backends:
  Replace status-line with: "🔍 [Service]: Investigation incomplete — instrumentation gap"
  Add section: "Gap: [signal type] — likely not instrumented. Recommendation: [specific instrumentation]."
  Add section: "Unresolved hypotheses: [ACTIVE hypotheses that could not be tested]."
</output_contract>

✅ [Service]: No anomalies detected in [window]
Checked:
  Metrics: error rate [X%], p99 latency [Yms], saturation [Z%] — within baseline
  Traces:  [N] traces sampled, no error spans, p99 [Yms]
  Logs:    no error-level entries in window
Possible explanations:
  - **Semantic Failure**: Telemetry is healthy, but logic/data/state is flawed (e.g., returned 200 OK with wrong data).
  - **Self-Healing**: Issue resolved before investigation began (check recent deploy/restart).
  - Issue is intermittent → suggest: set alert on [specific metric]
  - Issue outside instrumented scope → suggest: check [uninstrumented area]
Ruled out: [hypotheses tested]
```

---

## CONFLICT Format

*Use when two backends return irreconcilable evidence about the same event (e.g., Metrics show 0.1% error rate, Traces show 40% error spans).*
```
<output_contract: CONFLICT>
Required sections (in order): conflict-statement, evidence-a, evidence-b, most-likely-explanation, resolution-query
Optional: additional-evidence (include when ≥1 other backend produced WEAK or UNKNOWN BACKEND findings not part of the primary conflict): "Additional (WEAK/UNKNOWN): [backend] reports [X] — unconfirmed until cross-validated with a known backend."
Most-likely-explanation selection rule: choose based on evidence — if metric count ≠ trace count → cardinality mismatch; if timestamps misalign → clock skew; if label scopes differ → label scope difference; if sampling < 100% → sampling rate. Emit: "Likely: [explanation] because [observable evidence]."
</output_contract>

⚠️ [Service]: Contradictory evidence — cannot determine root cause without resolution
Conflict: [Backend A] reports [X] | [Backend B] reports [Y]
Evidence A: [specific query + result]
Evidence B: [specific query + result]
Most likely explanation: [cardinality mismatch | sampling rate difference | clock skew | label scope difference]
Resolution: run [specific query or action] to determine which reading is authoritative
```
