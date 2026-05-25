# CLAUDE.md - Project orientation for the EHAM A-CDM demo

This file is the entry point for Claude (or anyone else) opening the repo cold.
It links the moving parts; the talk-track in
[A-CDM-Decision-Hub-Talk-Track.md](A-CDM-Decision-Hub-Talk-Track.md) is the
narrative spine.

## What this is

A RelationalAI demo of an Airport Collaborative Decision Making (A-CDM)
Decision Hub at Schiphol (EHAM). A single PyRel ontology over 320 flights
on 2026-10-14 backs four reasoners and a Snowflake Intelligence agent:

| Act | Reasoner       | Question                                                                |
|-----|----------------|-------------------------------------------------------------------------|
| 1   | Rules          | TOBT compliance audit (MS12 +/-5 min vs ARDT)                           |
| 2   | Graph          | Rotation + slot-block cascade from a late inbound                       |
| 3   | Heuristic      | MS5 gate-conflict ranking (predictive reasoner is preview)              |
| 4   | Prescriptive   | TSAT re-sequence under storm (47 flights, HiGHS MIP)                    |
| 5   | Persistent rule | Operator adds 'preserve KL high-pax' rule; ontology stores it; re-solve respects it |

The data, the rules, the graph edges, the optimization model, AND the
Cortex agent all reference the same `eham_acdm` model. Adding a rule for Act
5 takes ~10 lines of PyRel and propagates to everything downstream.

## Repository layout

```
.
├── A-CDM-Decision-Hub-Talk-Track.md     # the script (read this first)
├── DEMO_QUESTIONS.md                    # the 5 acts as plain-English Qs
├── SNOWSIGHT_DEMO.md                    # 3-question Snowsight talk track
├── HANDOFF_BRIEFING.md                  # original handoff context
├── data/                                # raw + generator + load.sql
│   ├── build_eham_demo_data.py          # seed=42 generator (do not edit live)
│   ├── load_to_snowflake.sh             # reproducible loader
│   └── out/                             # ddl, reference, aircraft, load, validation .sql
├── rai_code/
│   └── manual/
│       ├── eham_acdm.py                 # the ontology (THE source of truth)
│       ├── demo_queries.py              # 5 act queries
│       └── eham_acdm_demo.ipynb         # Plotly viz notebook
├── agent/
│   ├── deploy.py                        # CortexAgentManager wrapper
│   └── queries.py                       # QueryCatalog wrappers (plain + _chart variants)
├── build/
│   ├── generate_demo_figures.py         # produce PNG charts for RUNNING.html
│   └── figures/                         # act1..act5 PNGs (regenerate before a demo)
├── RUNNING.html                         # self-contained run map with embedded figures
└── .venv/                               # uv venv, Python 3.13
```

## Snowflake state

- Connection: `snow` profile `rai`, account `ajb85638`, role `ACCOUNTADMIN`,
  warehouse `RAI_XS`, database `ACDM_DEMO`, schema `EHAM`.
- Tables: 320-row `FLIGHT`, 8 dim tables, `STORM_SLOT` (120 rows backing the
  Act 4 LP), `SLOT_BLOCK` (2 rows backing the operational cascade in Act 2).
- All tables have change-tracking enabled (required for RAI's CDC).
- The Cortex agent is registered in `SNOWFLAKE_INTELLIGENCE.AGENTS` and its
  stored procedures live in `ACDM_DEMO.RAI_AGENT`.
- The Snowsight notebook lives in `ACDM_DEMO.NOTEBOOKS.EHAM_ACDM_DEMO`,
  backed by stage `ACDM_DEMO.NOTEBOOKS.ACDM_NOTEBOOK_STAGE` with the three
  files (notebook + `eham_acdm.py` + `demo_queries.py`) under a
  `planes/` subfolder so it browses cleanly as a workspace folder named
  "planes" in Snowsight's Files view.

## Reasoner engines

- `acdm_logic_l` (HIGHMEM_X64_L) - backs the rules + graph + ad-hoc queries.
- `acdm_prescriptive_m` (HIGHMEM_X64_M) - backs Acts 4 and 5.

Both are configured in `rai_code/manual/eham_acdm.py` (`_build_config()`) and
referenced by the Cortex agent at runtime, so the same warm engines back the
notebook AND the deployed agent.

## How to run

```bash
# Pre-flight gate before any demo (the one command worth typing 10 min ahead).
# Verifies Snowflake connection, schema, change tracking, talk-track numbers,
# resumes RAI engines, runs all 5 acts, confirms the SI agent is healthy, and
# regenerates RUNNING.html figures. Ends with PASS/FAIL summary.
.venv/bin/python prep_demo.py
.venv/bin/python prep_demo.py --skip-figures --skip-chat  # fast iteration
.venv/bin/python prep_demo.py --skip-snowsight            # skip stage upload
.venv/bin/python prep_demo.py --redeploy                  # force agent redeploy

# Smoke-test the ontology and all 5 queries:
.venv/bin/python rai_code/manual/demo_queries.py

# Open the demo notebook (Plotly viz):
.venv/bin/python -m jupyter lab rai_code/manual/eham_acdm_demo.ipynb

# Deploy the Cortex agent:
.venv/bin/python -m agent.deploy deploy

# Talk to the agent from the CLI:
.venv/bin/python -m agent.deploy chat "Show TOBT violations by handler"

# Tear down:
.venv/bin/python -m agent.deploy teardown

# Regenerate the figures embedded in RUNNING.html (do this whenever the
# ontology, queries, or LP objective changes):
.venv/bin/python build/generate_demo_figures.py
```

## Timing (measured on this account, 2026-05-19)

**Zero to warm.** Cold start with both RAI engines suspended:

| Stage                                            | Cold time | Warm time |
|--------------------------------------------------|-----------|-----------|
| `snow` connection test                           | <1 s      | <1 s      |
| Schema + change-tracking checks                  | ~10 s     | ~10 s     |
| Talk-track number validation (4 raw SQL queries) | ~20 s     | ~20 s     |
| Resume `acdm_logic_l` engine (HIGHMEM_X64_L)     | 60-90 s   | 0 s (already READY) |
| Resume `acdm_prescriptive_m` engine (HIGHMEM_X64_M) | 60-90 s | 0 s (already READY) |
| `demo_queries.py` end-to-end (Q1-Q5)             | ~5-6 min  | ~5 min    |
| `agent.deploy status` (or `update`)              | ~8 s      | ~8 s      |
| Live `agent.deploy chat` Q1 round-trip           | ~85 s     | ~65 s     |
| Figure regeneration (5 PNGs via kaleido)         | ~30 s     | ~30 s     |
| Snowsight notebook PUT + ALTER LIVE VERSION      | ~25 s     | ~25 s     |
| **`prep_demo.py` total**                         | **~8 min**| **~6 min**  |

The warm CLI smoke test is bottlenecked by the two LP solves (Q4, Q5 ~80 s each
including ship-to-engine and result materialisation; HiGHS itself runs in ms).

**Warm SI agent chat latencies.** End-to-end from `agent.deploy chat "..."`
(LLM round-trip + tool execution + LLM response generation):

| Act | Reasoner        | Sample query                                          | Warm chat time |
|-----|-----------------|-------------------------------------------------------|----------------|
| 1   | Rules           | *"Show TOBT violations by handler"*                   | ~65 s          |
| 2   | Graph           | *"Trace the rotation impact if KL1234 is late"*       | ~65 s          |
| 3   | Heuristic       | *"Rank arrivals by MS5 conflict risk"*                | ~75 s          |
| 4   | Prescriptive    | *"Solve the TSAT re-sequence for the storm window"*   | ~3 min         |
| 5   | Persistent rule | *"Re-solve preserving KL high-pax flights"*           | ~2-3 min       |

Roughly a third of each chat is the Anthropic/Cortex LLM round-trip; the rest
is the sproc executing the PyRel query against the warm engine. Q4/Q5 are
slower because the LP serializes 47 x 120 binary decisions and the LLM has
more rows to summarise in the response. The raw PyRel query times from the
notebook are 5-15 s per act once everything is warm.

**Demo-day cadence.** Run `prep_demo.py` ~10 minutes before the call. Once
green, every speaker action (notebook cell, SI chat) responds in seconds for
the rules/graph/heuristic queries and ~3 minutes for the LP. Budget 2-3 min
of dead air around the Act 4 / Act 5 solves; that's the moment the talk-track
recommends for narrative.

## Anchored numbers (from `eham_demo_validation.sql`)

These are the demo-promise numbers. They reproduce exactly under the
definitions in `data/out/eham_demo_validation.sql`:

| Metric                                              | Value          |
|-----------------------------------------------------|----------------|
| TOBT violations (4h before 14:30, \|ARDT-TOBT\|>5)  | KLG 7, AGS 5, DNATA 3, MENZIES 2 |
| Cascade from KL1234 (reachability)                  | 6 downstream flights (KL1235, KL0407, KL1402, KL1601, HV5821, AF1241) |
| MS5 candidates (TLDT 14:30-15:00, ALDT NULL)        | 5 named flights surface in top-7 |
| Storm-window departures (TSAT 15:00-17:00)          | 47 |
| KL deps in storm with pax_conn > 80                 | 17 |
| KL691 -> RJTT pax connections                       | 137 |
| Storm event                                         | WX_2026101415, 15:00-17:00, 18C/36C+27 |
| CTOT rate                                           | 22.2% of departures |

## Conventions and gotchas

- **No pandas in query logic.** The PyRel queries in `demo_queries.py` express
  everything via the model. Pandas appears only on the post-`to_df()` display
  side (sort, format).
- **Mirror `~/rai-repos/supply_chain_demo`** for new files. Same author, same
  layout: `rai_code/manual/<domain>.py`, `demo_queries.py`, `.ipynb`,
  `agent/deploy.py`, `agent/queries.py`.
- **Snowflake is the single source of truth.** The slot-block cascade
  dependency (Act 2) lives in `SLOT_BLOCK`, not as a pandas constant. The 120
  storm-window slots (Act 4) live in `STORM_SLOT`.
- **Engine naming matters.** Named engines persist across sessions; nameless
  engines are torn down between runs. Both reasoners use named engines.
- **Talk-track Act 3 is preview-only.** We use a deterministic fit-score
  expressed entirely as PyRel derived properties. The talk track explicitly
  authorises this fallback.

## Where to look when something breaks

- Validation queries: `data/out/eham_demo_validation.sql` (run against
  Snowflake to confirm the demo numbers are intact).
- Smoke test: `rai_code/manual/demo_queries.py` `main()` (prints all 5 act
  results).
- Agent status: `.venv/bin/python -m agent.deploy status`.
- Engine state: `.venv/bin/rai reasoners list`.
