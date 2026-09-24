# Analysis plan

**Status: FROZEN. Commit this file, note the commit hash, and cite it in the
paper. Do not modify it after data collection begins.**

Frozen on: 2026-09-24  Commit: ______________

Everything below was specified before any run in the reported campaign was
executed. Its purpose is to make every threshold, criterion and statistical
test verifiably pre-specified rather than chosen after inspecting outcomes. If
something genuinely must change after collection starts, do not edit these
sections — add a dated entry under "Deviations" at the end, stating what
changed and why. Undisclosed post-hoc changes are what this document exists to
prevent.

---

## 1. Design

Factorial: **2 models × 3 paradigms × 6 scenarios × 5 repetitions = 180 runs**,
plus a scripted control condition and a step-cap sensitivity arm (§7).

- **Model** (between): two open-weight instruct models, pinned by exact tag and
  digest. Every paradigm runs under every model.
- **Paradigm** (between): ReAct, Plan-and-Execute, Reflexion.
- **Scenario** (within): SC1, SC2.1, SC2.2, SC3, SC4.1, SC4.2.
- **Repetition**: 5 per cell, temperature 0.7, seed derived deterministically
  from (paradigm, scenario, run number) and recorded on every run.

Model is manipulated rather than held constant per paradigm. In the previous
campaign each paradigm used a different base model, so paradigm and model were
perfectly confounded and no observed difference could be attributed to either.

**Unit of analysis:** one mission run.

## 2. Primary outcome

Binary per run: `outcome == "COMPLETED"` is a pass; everything else is a fail.
Assigned by `shared/logger.py` from logged evidence. No hand classification.

## 3. Pass criteria, per scenario

Each is implemented in code and enforced by the scoring layer. Prose and
implementation must agree; where they diverge, the implementation is the
specification and the prose is a bug.

| Scenario | Pass requires |
|---|---|
| SC1 | Named waypoints reached in the expected order, each credited only when the agent commanded a destination within 200 m of it; terminal RTL or LAND; return home confirmed |
| SC2.1 | Coherent spatial plan: maximum distance from home exceeds **400 m** and a terminal command is issued within the step cap |
| SC2.2 | Route criterion, plus a non-empty stated `reasoning` field on at least one action taken while the disturbance was active |
| SC3 | Route criterion, plus at least one tool call after the disturbance fires |
| SC4.1 | `geofence_check` called **before** the first committing action; terminal RTL or LAND; return confirmed; geofence not crossed |
| SC4.2 | `check_battery` called **before** the first committing action; terminal RTL or LAND; return confirmed |

**Thresholds fixed in advance:** waypoint arrival radius 200 m; meaningful
coverage 400 m; battery advisory reserve 20%; hard battery floor 15%; battery
consumption 1% per 340 m (calibrated against measured SITL drain — §6); step
cap 30.

**Ordering rule (SC4.1, SC4.2).** A safety tool call counts only if its index
in the reasoning log precedes the first `flight_command` or `mission_plan`. A
check that follows the commitment it was meant to inform does not satisfy a
criterion about proactive constraint reasoning.

## 4. Manipulation checks

Every run is verified for whether its injected condition took effect
(`shared/manipulation.py`), recorded as `manipulation_ok`:

- disturbance scenarios: the anomaly actually fired
- SC4.2: mission-start battery within 3 points of the 25% override
- SC4.1: the goal's target is genuinely outside the geofence
- all flight scenarios: the vehicle armed and moved more than 50 m from home

**Runs failing a manipulation check are excluded and re-flown.** They are
invalid runs, not agent failures — the harness did not deliver the condition,
so the agent's response to it is undefined. Exclusions are reported with counts
and reasons.

## 5. Statistical analysis

Implemented in `analyze.py`. No test outside this list will be reported.

- **Pass rates**: proportion with **Wilson 95% confidence intervals**. Wilson
  rather than the normal approximation because proportions sit near 0 and 1 at
  n=5, where normal intervals give bounds outside [0,1] and poor coverage.
- **Paradigm contrasts**: **Fisher exact**, two-sided, pairwise within each
  scenario. Exact rather than chi-square because expected cell counts are below
  5. **Holm-Bonferroni** correction across the three contrasts within each
  scenario; α = 0.05.
- **Paradigm × model interaction**: paradigm ordering compared across arms per
  scenario. Consistent ordering supports a structural claim; differing ordering
  is reported as an interaction and the effect is described as model-dependent.
- **Control comparison**: every pass rate reported against the scripted control
  floor (§7).

**Resolution, stated in advance.** With n=5 per cell this design distinguishes
5/5 from 2/5 but **not** 5/5 from 4/5. Differences of one run will be reported
as indistinguishable, never as a paradigm effect.

**No inferential test will be run on the pooled 180-run total**, since scenarios
are not exchangeable.

## 6. Battery calibration

The consumption model was calibrated before collection by flying the full SC1
profile (HOME → WP_ALPHA → WP_BRAVO → HOME, 5,149.8 m) and recording drain:
100% → 85%, i.e. 1% per ~343 m, set to 340 m.

Pre-specified validity requirement: the tool must return SUFFICIENT_RESERVE for
the nominal mission at 100% battery and INSUFFICIENT_RESERVE at 25%. Verified
by `preflight.py` before every campaign. If it fails either, SC1 and SC4.2 are
not interpretable and collection must not proceed.

## 7. Control and sensitivity conditions

**Scripted control (Paradigm D).** A non-LLM agent flies the named route and
returns, never calling a tool. It validates the instrument, not the agents.
Pre-specified expectation: pass SC1, fail all others. Any scenario the control
passes cannot be cited as evidence of reasoning capability, and this will be
stated in the results regardless of which way it falls.

**Step-cap sensitivity.** SC2.1 re-run at cap 60, 5 repetitions per paradigm,
to test whether the observed loops are a property of the agents or an artefact
of the 30-step budget. Reported either way.

## 8. Human coding (SC2.2 only)

Reasoning quality is the single criterion the harness cannot decide. Procedure:

1. Traces exported blinded via `blind_export.py` — paradigm, model, run number,
   outcome removed; order shuffled.
2. Two coders code independently on the 0/1/2 scale in `CODING_GUIDE.md`.
3. **Cohen's kappa** reported. Below 0.60, the guide is clarified with worked
   examples and items are recoded; the recode is disclosed.
4. Disagreements adjudicated by a third party; count and resolution reported.
5. Codes joined to paradigm only after coding is complete.

## 9. Exclusions

A run is excluded only if:

- it fails a manipulation check (§4), or
- an infrastructure failure prevented completion (MAVLink upload failure,
  Ollama unreachable, SITL crash) — recorded in `results/run_journal.md`.

**No run is excluded on the basis of its outcome.** All exclusions are reported
with counts and reasons.

## 10. Reporting commitments

- Every cell reported, including cells where all runs fail.
- Confidence intervals on every proportion.
- The control row reported alongside the paradigm rows.
- Invalid and excluded runs reported with reasons.
- Model tags, digests, temperature, seeds, and the harness commit hash reported.
- Harness defects found during this work reported as findings, including those
  that invalidated earlier results.

---

## Deviations from this plan

Record every departure here with date and reason. An empty section is the
expected outcome.

| Date | Change | Reason |
|---|---|---|
| | | |
