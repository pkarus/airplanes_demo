# Handoff Briefing - RelationalAI A-CDM Decision Hub Demo

**For:** the next Claude picking up this work
**From:** the previous Claude in this conversation
**Project owner:** Piotr (piotr.kraus01@gmail.com)
**Status:** Talk track delivered, synthetic data delivered. Open items at the end.

Read this document first. Everything you need to be productive on this project is here. The other files in the folder are the deliverables; this is the orientation.

---

## TL;DR

Piotr is building a customer demo for **RelationalAI** (decision intelligence platform on Snowflake) in the **A-CDM aviation operations** domain (Airport Collaborative Decision Making per the ICAO Generic Milestones procedure). Audience is **aviation customers** - airport authorities, ANSPs, airlines. Delivery format is a **live talk track** that walks through four reasoning patterns (rules, graph, predictive, prescriptive) on a synthetic Amsterdam Schiphol (EHAM) dataset. The synthetic data is built to be **defensible to aviation domain experts** - a Schiphol duty manager should look at it and say "that's a Tuesday at Schiphol", not "this is fake".

## What's been done

1. **Talk track** at `A-CDM-Decision-Hub-Talk-Track.md` - ~50KB, 18-22 min demo script with 5 acts (rules / graph / predictive / prescriptive / superalignment), Q&A prep, glossary, and data fabrication brief.
2. **Synthetic data** in `synthetic-data/` - Python generator producing Snowflake DDL, reference INSERTs, flight CSV (320 flights with all 16 ICAO milestones), load script, and validation queries. Data anchored to EHAM Schiphol on Tue 14 Oct 2026.
3. **Self-validation** of the data: confirms Act 1 returns KLG 7 / AGS 5 / DNATA 3 / MENZIES 2 TOBT violations, Act 3 returns the 4+1 MS5 candidates with the right stands, Act 4 has exactly 47 flights in the TSAT window, Act 5 finds KL691 to RJTT with 137 pax connections.

## Project context

**RelationalAI** sells "decision intelligence" via four reasoners:
- **Rules-based** (GA) - derived properties, classification, validation
- **Graph** (GA) - centrality, community, reachability, similarity
- **Predictive** (early access) - forecasts, churn, anomaly
- **Prescriptive** (preview) - optimization with HiGHS / Gurobi / Ipopt / MiniZinc backends

All four sit on a **semantic model (PyRel ontology)** instantiated inside Snowflake via the RelationalAI Native App. The pitch is "high-stakes decisions deserve frontier intelligence" and the differentiator is one model serving all four reasoners with grounded outputs.

**Why A-CDM is a near-perfect fit:** the ICAO Generic A-CDM Milestones Procedure (Chapter 4, Attachment 4.8) defines 16 milestones (MS1-MS16) tracking a flight from filed flight plan through landing, turnaround, and take-off. Each milestone is a timestamp shared across stakeholders (TWR, ACC, AODB, ATFM, A-SMGCS, GHA, airlines). Every A-CDM decision (TOBT discipline, pre-departure sequencing, gate assignment, runway sequencing) is data fusion + rules + graph + optimization. RelationalAI's four reasoners map 1:1.

**Why EHAM (Schiphol):** reference A-CDM implementation in Europe, public AIP for defendable infrastructure data, real STAR fixes (ARTIP, SUGOL, RIVER, NIRSI), real runways (18R Polderbaan with famous long taxi-in, 18C Zwanenburgbaan, 18L Aalsmeerbaan, 06/24 Kaagbaan), real pier layout (B/C/D/E/F/G/H/M/R), real carrier mix (KL 52% dominance, SkyTeam, low-cost). A Schiphol-anchored dataset is the most defendable choice for a European customer pitch. If the customer is African (the ICAO doc came from WACAF), swap to HAAB Addis or DNMM Lagos using the same method.

## Key decisions made (and why)

| Decision | Why |
|---|---|
| **EHAM Schiphol** as the data anchor | Reference A-CDM implementation, public AIP, known carrier mix. Highest defendability for European pitch. |
| **320 flights** for one Tuesday in Oct 2026 | Realistic daytime slice. Full EHAM day is ~1,400 movements but we time-box and exclude GA. |
| **All four reasoners, shallow coverage** | One query per reasoner. Shows full RAI surface in 18-22 min vs deep-dive on one. |
| **Live talk track format** (not slides or working code as primary) | Customer pitch context. Speaker reads it while clicking through a real demo. |
| **Synthetic data with disclosure** | Lie about realness = lose the room. Disclosed synthetic anchored to real EHAM = fine. |
| **22% CTOT regulation rate**, **78% slot adherence** | Matches published Eurocontrol numbers. Domain experts will check. |
| **75% TOBT compliance**, **15% late ready, 10% early ready** | Industry-standard distribution. Schiphol publishes ~80%. |
| **Polderbaan (18R) taxi-in 11-13 min** | Famous Schiphol detail. Get this wrong and a controller spots it. |
| **Schengen / non-Schengen pier split modeled** | F/G/M non-Schengen, B/C/D-north/E/H Schengen. Real EHAM rule. |
| **KL widebody fleet realistic** | B737/B738/B739/E-jets/A330/B777/B787/A350. Excludes A380 (KL retired pre-2020). |

## User preferences (CRITICAL - read carefully)

Piotr has explicit preferences. Violate these and you'll waste his time:

- **Truth and precision over approval.** Never agree just to be agreeable. If your position holds, restate it with evidence.
- **Brutally honest, direct.** Can be provocative, pointed. No fluff, no fillers, no "as an AI" disclaimers, no hedging.
- **Challenge his assumptions.** Lead with the strongest counterargument.
- **He relies on you for awareness.** "I don't know what I don't know" - proactively surface related topics, alternatives, gaps he might miss.
- **Ask clarifying questions** when the task is ambiguous. Use the AskUserQuestion tool when in Cowork mode.
- **NEVER use em-dash (—). Use standard hyphen (-) instead.** This is non-negotiable.
- **No emojis** unless he asks or uses them first.

He works in **Cowork mode** (Claude desktop app feature). Folder access: `/Users/piotrkraus/Documents/Claude/Projects/RAI demo`. He has the RelationalAI Snowflake Native App set up and can actually execute PyRel code.

## File map

```
RAI demo/
├── HANDOFF_BRIEFING.md                          <-- THIS FILE
├── A-CDM-Decision-Hub-Talk-Track.md             <-- The talk track (read this second)
└── synthetic-data/
    ├── README.md                                <-- How to load into Snowflake
    ├── build_eham_demo_data.py                  <-- Reproducible generator (seed=42)
    └── out/
        ├── eham_demo_ddl.sql                    <-- CREATE DATABASE/SCHEMA/TABLE
        ├── eham_demo_reference.sql              <-- INSERT for dim tables
        ├── eham_demo_aircraft.sql               <-- INSERT for dim_aircraft
        ├── eham_demo_flights.csv                <-- 320 flights, 16 milestones each
        ├── eham_demo_load.sql                   <-- CREATE STAGE + COPY INTO
        └── eham_demo_validation.sql             <-- Queries proving the data fits the script
```

## Key facts you'll need (memorize these)

### The 16 ICAO A-CDM milestones (in order)

| # | Code | What it is |
|---|------|---|
| MS1 | EOBT, SOBT | ATC flight plan filed (EOBT vs scheduled SOBT) |
| MS2 | CTOT | Calculated Take-Off Time issued (only if regulation active) |
| MS3 | ATOT_up | Take-off from upstation (departure from origin) |
| MS4 | FIR entry | Crosses FIR boundary into destination airspace |
| MS5 | TLDT, ELDT | Final approach / TMA entry. AMAN refines TLDT here. |
| MS6 | ALDT | Actual touchdown on runway |
| MS7 | AIBT | Aircraft on stand (in-block) |
| MS8 | ACGT | Ground handling starts (= AIBT for normal turn) |
| MS9 | TOBT | Operator commits to off-block time. **+/-5 min compliance window.** |
| MS10 | TSAT | ATC issues start-up approval target |
| MS11 | ABDT | Boarding starts |
| MS12 | ARDT | Pilot calls ready. **+/-5 min vs TOBT is the compliance threshold.** |
| MS13 | ASRT | Start-up request to ATC |
| MS14 | ASAT | Start-up approval granted |
| MS15 | AOBT | Off-block (push starts) |
| MS16 | ATOT | Actual take-off |

**Required:** MS1, MS6, MS7, MS9, MS10, MS15, MS16. CTOT (MS2) is required only if integrated with ATFM.

### RelationalAI PyRel essentials

```python
from relationalai.semantics import Model, String, Integer, Float, DateTime
from relationalai.semantics.std.datetime import datetime as dt_fn
from relationalai.semantics.std import aggregates as agg
from relationalai.semantics.reasoners.prescriptive import Problem

m = Model("MyModel")

# Concept with identity
Flight = m.Concept("Flight", identify_by={"callsign": String, "sobt": DateTime})

# Single-valued attribute = Property
Flight.tobt = m.Property(f"{Flight} TOBT is {DateTime}")

# Multi-valued or n-ary = Relationship
Flight.feeds = m.Relationship(f"{Flight} feeds outbound {Flight:outbound}")

# Boolean flag = unary Relationship
Stand.is_contact = m.Relationship(f"{Stand} is a contact stand")

# Derived concept membership
TOBTViolation = m.Concept("TOBTViolation", extends=[Flight])
m.where(
    dt_fn.diff("minutes", Flight.ardt, Flight.tobt) > 5
).define(TOBTViolation(Flight))

# Self-join with .ref()
F1 = Flight.ref()
F2 = Flight.ref()
m.where(F1.stand == F2.stand, F1.callsign < F2.callsign, ...)

# Query with aggregation
df = m.where(...).select(
    Flight.handler.code.alias("handler"),
    agg.count(Flight).per(Flight.handler).alias("violations"),
).to_df()

# Prescriptive workflow
p = Problem(m, Float)
p.solve_for(Flight.x_assign(Slot, x), populate=True, type="bin", lower=0, upper=1)
p.satisfy(m.require(constraint))
p.minimize(objective_expression)
p.solve("highs")
```

**Logical operators:** `&`, `|`, `m.not_()` (NOT Python's `and`/`or`/`not`).

**Skill marketplace:** RelationalAI ships skills at `github.com/RelationalAI/rai-agent-skills`. Skills: rai-setup, rai-pyrel-coding, rai-ontology-design, rai-rules-authoring, rai-querying, rai-graph-analysis, rai-prescriptive-problem-formulation, rai-prescriptive-solver-management, rai-prescriptive-results-interpretation, rai-discovery, rai-build-starter-ontology, rai-cortex-integration, rai-health. Install via `/plugin marketplace add RelationalAI/rai-agent-skills` in Claude Code.

### EHAM essentials a domain expert will check

- **Runways:** 18R (Polderbaan), 18C (Zwanenburgbaan), 18L (Aalsmeerbaan), 06/24 (Kaagbaan), 09/27 (Buitenveldertbaan), 04/22 (Schiphol-Oost, GA only)
- **STAR fixes (IAFs):** ARTIP (east→27/18C), SUGOL (north→18C), RIVER (NW→18C), NIRSI (NW→18R)
- **Pier layout:** B/C/D/E/H Schengen; D-south, F/G/M non-Schengen
- **Polderbaan taxi-in is 11-13 minutes**
- **KL fleet at EHAM:** B737-700/800/900, E175/E190/E195, A330-200/300, B772/B77W, B789/78X, A359. No A380.
- **Turnaround mins:** KL narrowbody 40, KL widebody 90, HV 25, DL widebody 120, KL Cityhopper E-jets 30
- **TSAT window definition:** +/- 5 min per Schiphol A-CDM Manual v1.0 (Feb 2024)
- **CTOT compliance window:** TTOT in [CTOT-5, CTOT+10] per Eurocontrol

### What's deliberately NOT modeled (and own this in the demo)

- De-icing season (Oct is too early)
- Crew duty time
- Pax bag connection sequencing
- ATFM regulation prediction (CTOTs are pre-stamped)
- Cargo flights minimal (only 5X UPS briefly)
- GA on RWY 04/22

State these up front. Owning the gaps builds trust.

## The five demo acts (and what data fits each)

| Act | Reasoner | Question | Data anchor |
|---|---|---|---|
| **1** | Rules | "Show TOBT compliance violations last 4h by handler" | Curated KL7000-7006, HV7100-7104, AF8000-8002, BA8100-8101 with controlled deviations 6-12 min |
| **2** | Graph | "Trace rotation cascade from KL1234 (35 min late at MS5)" | KL1234 + KL1235 (rotation) + KL1402/KL1601/HV5821/AF1241 (downstream) + KL0641/KL0712 (stand-occupier upstreams) |
| **3** | Predictive | "Rank MS5 inbounds by gate conflict probability" | DL0036/E18, KL0691/F04, AF1641/D88, BA0432/B26 + KL1234/E18 (5 flights with TLDT 14:30-15:00, ALDT NULL) |
| **4** | Prescriptive | "Re-sequence TSATs under storm closing 18C 15:00-17:00" | Exactly 47 departures with TSAT in [15:00, 17:00). Wake categories populated. CTOT on ~22%. Pier for pushback contention. |
| **5** | Superalignment | "Add rule: KL flights >80 pax conn delay <= 8 min" | KL691 to RJTT with 137 pax conn (matches the rule), plus 1-2 others |

## Pitfalls discovered (don't repeat these mistakes)

1. **Bulk generator callsign collisions.** The bulk-flight loop increments a counter from 100 and can land on "KL1234" with a different SOBT, shadowing the curated narrative flight. Fix: maintain a `reserved_callsigns` set of curated callsigns and skip them in bulk.

2. **SOBT convention drift.** Initial generator treated SOBT for arrivals as "scheduled departure from origin" (upstream time). AODB convention is SOBT = scheduled in-block at EHAM regardless of direction. This bug put KL0641 landing at 22:49 instead of 13:10 - missing the cascade window entirely. Fixed by rewriting `synthesize_arrival` to back-compute timestamps from SOBT-as-STA.

3. **TSAT window tuner clobbering audit window.** The function that shifts flights into 15:00-17:00 to hit the 47-flight target was also shifting curated TOBT-violation flights, knocking their ARDT out of the 10:30-14:30 audit window. Fixed by excluding any flight whose ARDT is already in the audit window from the tuner's candidate pool.

4. **PyRel `datetime_diff` vs `datetime.diff`.** The docs use `from relationalai.semantics.std.datetime import datetime; datetime.diff("minutes", a, b)`, NOT `datetime_diff()`. Verified against the actual derive-facts docs.

5. **PyRel logical operators.** Use `&`, `|`, `m.not_()`. Python's `and`/`or`/`not` cannot be overloaded and will fail silently or give wrong results.

6. **Predictive reasoner is preview/early-access.** The talk track's Act 3 forecast code assumes `Forecast` constructs that may not exist in the user's RAI version. The talk track has a disclaimer pointing to a deterministic fallback rule.

## Open items / what could be next

Piotr hasn't asked for these, but these are the natural next steps:

1. **Actually run the PyRel code against the live RAI/Snowflake instance.** The user has it set up. The talk track code is talk-track-grade and needs to be syntax-verified before any live demo. Spin up a Jupyter notebook in the working directory, import `relationalai`, and run each act's code cell end-to-end. Fix syntax drift against the installed version.

2. **Build the Plotly Gantt for Act 4.** The talk track references a before/after Gantt showing the original chaotic TSAT schedule vs the optimized one. Currently no chart code exists. ~50 lines of Python.

3. **Build the Workshop-style HMI mock.** The talk track suggests a second-monitor display showing the live milestone timeline (operator's-eye view). Optional, makes the demo feel like a real ops floor. Could be a single HTML file or Streamlit app.

4. **Create the operator-feedback persistence demo.** Act 5 says the new constraint persists across sessions. Show how that actually works in PyRel (write a new derived property, commit the model, demonstrate retrieval in a new notebook session).

5. **Pre-recorded screencap as backup.** Talk track says "if any cell fails on stage, fall back to the pre-recorded screencap." That recording doesn't exist yet.

6. **Customer-specific variant.** If the customer is HAAB Addis or DNMM Lagos (per the ICAO WACAF source), re-anchor the data to that airport's AIP. Method is the same; need to pull the runway/fix/pier data for the target hub.

7. **POC scope doc.** The talk track teases a "six-week POC". Write the formal scope doc (3 deliverables, integration shape, governance approach) so it's ready when the customer asks.

## How to pick this up

If Piotr asks you to continue:

1. Re-read this briefing.
2. Read `A-CDM-Decision-Hub-Talk-Track.md` (the deliverable that defines what "done" looks like for the demo).
3. Skim `synthetic-data/README.md` so you understand the data.
4. Ask Piotr what he wants next. Likely candidates: dry-run the PyRel code, build the Gantt, build the HMI mock, prepare a POC scope doc, or pivot to a different anchor airport.
5. Before running PyRel against Snowflake, **always** load `relationalai` and check the installed version. Syntax drifts between versions. The skill `rai-pyrel-coding` (if installed) auto-generates current-version-correct code.

## Sample handoff prompt Piotr can paste to a new Claude

```
I'm continuing work on a RelationalAI demo in the A-CDM aviation domain.
There's a handoff briefing at:
/Users/piotrkraus/Documents/Claude/Projects/RAI demo/HANDOFF_BRIEFING.md

Read that file first, then the talk track and synthetic data README.
Don't restart from scratch. Continue where the previous Claude left off.

What I want next: [ <-- fill in: dry-run PyRel / build Gantt / etc. ]
```

## Important meta-notes about working with Piotr

- He's not asking for opinions on the project; he's asking for execution and challenge. Don't pad your responses.
- When in doubt, use `AskUserQuestion` (Cowork mode) instead of guessing or hedging.
- File outputs land in `/Users/piotrkraus/Documents/Claude/Projects/RAI demo/`. Use `computer://` links to share files with him.
- The temporary outputs folder (`/Users/piotrkraus/Library/Application Support/Claude/...`) is invisible to him. Always copy final deliverables to the working folder.
- If you create files, use `Write` for new ones and `Edit` for modifications. Read before Edit.
- He'll push back if you're wrong. Restate your position with evidence if it holds, capitulate only if the evidence is against you.

## End of briefing

You now have full context. The previous Claude did the research, the talk track, and the synthetic data. Your job is to extend or refine based on what Piotr asks next. Don't re-research what's already documented in this briefing or in the files. Don't second-guess the EHAM anchor or the 320-flight scope without asking - those are settled decisions.

Good luck. Be direct, be precise, no fluff.
