# SNOWSIGHT_DEMO.md - 3-question Snowflake Intelligence talk track

This is the 5-minute version of the demo, delivered directly inside
Snowflake Intelligence. After `agent/deploy.py deploy`, an `acdm` agent
appears in the SI picker. Three questions land the value:

> **One-click visualisation.** Each agent response that returns tabular
> data shows a small chart icon next to the result. Click it and Snowsight
> auto-picks a chart type from the table shape. For this demo the agent
> also includes an `chart_hint` field (`{type, x, y, title}`) in its tool
> response and mentions the suggested chart in its text reply — that's the
> cue for the speaker to click the chart button and land the visual. The
> chart-friendly query variants (`*_chart` in `agent/queries.py`) return
> tighter 2-3 column DataFrames specifically shaped for Snowsight's auto
> charter.

> **Expected latency** (engines already warm via `prep_demo.py`):
> the rules / graph / heuristic questions return in ~60-75 s end-to-end
> (LLM round-trip + sproc + tool result); the two prescriptive solves
> (Q4 and the persistent-rule re-solve) take ~2-3 min each. Budget the
> longer wait for narrative — the talk track explicitly calls out the LP
> as the moment to NOT take questions.

---

## Question 1 - The rules layer

**Type:**
> Show TOBT violations by handler in the last four hours.

**Expected answer:** A table or chart with KLG (7), AGS (5), DNATA (3),
MENZIES (2). The agent will reach into `tobt_violations_by_handler` in
`agent/queries.py`, which delegates to the `TOBTViolation(Flight)` derived
concept in the ontology.

**Speaker beat (15 seconds):**
> "The rule is encoded once - 'ARDT deviates from TOBT by more than 5
> minutes' - and queryable for any time window. It composes with the next
> question."

---

## Question 2 - The graph layer

**Type:**
> Trace the rotation cascade if KL1234 is late.

**Expected answer:** A list of 6 downstream flights at risk: KL1235, KL0407,
KL1402, KL1601, HV5821, AF1241. The agent delegates to
`rotation_cascade_from_kl1234`, which runs the Graph reasoner over the
union of `feeds_callsign`, `slot_blocks`, and `shares_stand` edges.

**Speaker beat (20 seconds):**
> "Three carriers, two terminals, one ATFM slot at risk - traced from a
> single late ALDT. Notice the agent didn't write SQL; it composed a
> graph query over the same model that defined the rule above."

---

## Question 3 - The prescriptive layer

**Type:**
> Re-sequence the storm-window TSATs to minimize weighted delay.

**Expected answer:** A table of 47 (callsign, minute_offset, delay_min)
rows. KL691 stays at 0 delay (its pax-conn weight protects it). The
agent delegates to `tsat_resequence_under_storm`, which builds and
solves a MIP with HiGHS via the Prescriptive reasoner.

**Speaker beat (25 seconds):**
> "About 5,600 binary variables, four constraint families, solved in
> seconds. The same model. Now watch what happens when we add a rule."

---

## (Optional) Question 4 - The persistent-rule moment

**Type:**
> Re-solve, but never delay KL flights with more than 80 connecting pax by
> more than 8 minutes from SOBT.

**Expected answer:** Same 47-flight assignment, but every KL flight with
pax_connections > 80 has `delay_min <= 8`. The agent delegates to
`tsat_resequence_with_preservation`, which uses the
`FlightSlot.assign_preserved` decision and adds the cap constraint.

**Speaker beat (15 seconds):**
> "The operator's instinct just became a derived concept on the ontology.
> Every reasoner that touches the same model from now on respects it -
> not just this solver, not just this conversation. Tomorrow morning's
> controller inherits it automatically."

---

## What to NOT say

- Don't say "this is just a database query." It is, and it isn't - the agent
  picks the right query from a catalog *and* the underlying queries
  compose graph + LP + rules across one model. The point isn't that any one
  of these is impossible. The point is that they all sit on one semantic
  layer.
- Don't say "the LP is the special part." The rules + graph + LP
  composition is the special part. Anyone with HiGHS can solve a MIP.
- Don't say "we replace your AODB." We don't. We sit on top of it as a
  decision layer.
