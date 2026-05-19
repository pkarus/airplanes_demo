# DEMO_QUESTIONS.md - The 5 acts as plain-English questions

These are the questions an A-CDM operator would type into a Snowflake
Intelligence search box. They map 1:1 to the `agent/queries.py` functions
exposed to the Cortex agent.

---

## Q1. Act 1 (Rules) - TOBT compliance audit

**"Show me every flight in the last four hours where actual ready time
deviated from TOBT by more than five minutes, broken down by ground
handler. These should have triggered automatic TOBT/TSAT removal per the
MS12 procedure."**

| Why it matters | Per ICAO MS12, ARDT must stay within +/-5 min of TOBT. Outside that band, A-CDM systems are supposed to remove the TOBT/TSAT and recompute. This audit shows which handlers are letting violations through. |
|----------------|---|
| Reasoner       | Rules (logic). The `TOBTViolation(Flight)` derived concept defines the rule once at the ontology layer. The query reuses it under a 4-hour filter. |
| Expected shape | KLG dominates (7), AGS next (5), then DNATA (3, negative deviation = calling ready early), MENZIES (2). |

---

## Q2. Act 2 (Graph) - Rotation cascade

**"KL1234 inbound from KJFK is 35 minutes late at final approach. Trace the
rotation impact across the next 6 hours. Which outbound flights, which
gates, which crew, which ATFM slots are at risk?"**

| Why it matters | Existing AODBs do inbound-outbound pairing for one aircraft. They don't propagate the delay across stand conflicts to OTHER carriers' flights using the same gate area. |
|----------------|---|
| Reasoner       | Graph. The cascade is the union of three edge types over the same ontology: `feeds_callsign` (rotation), `slot_blocks` (pushback contention), and `shares_stand` (stand-occupancy overlap). RAI's Graph reasoner walks reachability. |
| Expected shape | 6 flights at risk: KL1235 (rotation), KL0407 (stand), KL1402 (rotation+stand), KL1601 (rotation^2), HV5821 (slot_block, different carrier), AF1241 (slot_block, different alliance). Three carriers, two terminals. |

---

## Q3. Act 3 (Predictive heuristic) - MS5 gate-conflict ranking

**"Looking at all inbound flights with TLDT in the next 30 minutes, rank
them by probability of arrival gate conflict at MS5."**

| Why it matters | Stand planners don't want a yes/no alarm on every false positive. They want a ranked queue so they can intervene on the top 1-2 only. |
|----------------|---|
| Reasoner       | Deterministic heuristic in PyRel (Predictive reasoner is preview). Score = time-pressure + pax-connection weight + pier-bonus - WTC-mismatch penalty. All weights are PyRel-derived properties; the ranking is `.select(...ms5_score...).sort_values()`. |
| Expected shape | 11 candidates; talk-track 5 (KL1234, DL0036, KL0691, AF1641, BA0432) all surface. KL1234 is the cascade seed from Act 2. |

---

## Q4. Act 4 (Prescriptive) - TSAT re-sequence under storm

**"It's 14:30. Forecast says a thunderstorm cell crosses 18C/27 from 15:00
to 17:00, dropping arrival capacity 40% and forcing single-runway
departures off 18L. We have 47 outbound flights with TSAT in that window.
Constraints: each flight in exactly one slot, single-runway capacity, no
push before SOBT, at most 2 simultaneous pushes per pier. Objective:
minimize total weighted delay vs SOBT, weighted by pax connections and
ATFM penalty risk. Solve it."**

| Why it matters | Manual re-sequencing of 47 flights with 5 constraints isn't tractable. The current handoff between AMAN/DMAN and the duty manager is "best effort". |
|----------------|---|
| Reasoner       | Prescriptive. Binary decision `FlightSlot.assign_base` over 47 flights x 120 1-min slots = ~5,600 binaries. HiGHS solves in seconds. |
| Expected shape | Most flights at 0 delay (already inside their CTOT window). The flights that DO get delayed are those competing for the same SOBT minute. |

---

## Q5. Act 5 (Persistent rule) - Operator-provided preservation rule

**"Never delay KL flights with more than 80 connecting pax by more than 8
minutes from SOBT for runway re-sequencing. We absorb the delay
elsewhere."**

| Why it matters | This is *institutional knowledge* that today lives in a senior controller's head, a runbook PDF, or a spreadsheet override. By writing it as a derived concept on the ontology, the rule is stored alongside the data and respected by every reasoner that touches the same model — not just on the next re-solve, but for as long as it lives in the ontology. The next operator who joins inherits the rule automatically. |
|----------------|---|
| Reasoner       | Rules + Prescriptive. The rule becomes a `PreservedFlight(Flight)` derived concept and an aggregate-delay constraint `sum(assign * delay) <= 8` per preserved flight. Re-solving picks up the constraint automatically because the constraint reads from the ontology. |
| Expected shape | KL691 to RJTT (137 pax) and 16 other KL flights are preserved. Other flights absorb tiny delays to make room. |
