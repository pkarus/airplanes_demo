# EHAM A-CDM Decision Hub

A RelationalAI demo of an Airport Collaborative Decision Making (A-CDM)
Decision Hub at Amsterdam Schiphol (EHAM). A single PyRel ontology over 320
flights on 2026-10-14 (`ACDM_DEMO.EHAM` on Snowflake) backs four reasoner
families plus a persistent rule, and the same model is served through a
Snowflake Intelligence (Cortex) agent. The data, the rules, the graph edges,
the optimization model, and the agent all reference one `eham_acdm` model, so
adding a rule in Act 5 takes about ten lines of PyRel and propagates to
everything downstream.

## The five acts

| Act | Reasoner        | Question                                                                |
|-----|-----------------|-------------------------------------------------------------------------|
| 1   | Rules           | TOBT compliance audit (MS12 +/-5 min vs ARDT)                           |
| 2   | Graph           | Rotation and slot-block cascade from a late inbound                     |
| 3   | Heuristic       | MS5 gate-conflict ranking (the predictive reasoner is preview)          |
| 4   | Prescriptive    | TSAT re-sequence under a storm window (47 flights, HiGHS MIP)           |
| 5   | Persistent rule | Operator adds a "preserve KL high-pax" rule; the ontology stores it and the MIP re-solves to respect it |

## Run it

```bash
# Pre-flight gate (run ~10 min before a demo): verifies the Snowflake
# connection, schema, change tracking, and talk-track numbers, resumes the RAI
# engines, runs all 5 acts, checks the SI agent, and regenerates RUNNING.html.
.venv/bin/python prep_demo.py

# Smoke-test the ontology and all 5 act queries:
.venv/bin/python rai_code/manual/demo_queries.py

# Local notebook (Plotly visualisations):
.venv/bin/python -m jupyter lab rai_code/manual/eham_acdm_demo.ipynb

# Cortex agent in Snowflake Intelligence:
.venv/bin/python -m agent.deploy deploy
.venv/bin/python -m agent.deploy chat "Show TOBT violations by handler"
```

For a no-setup overview, open [RUNNING.html](RUNNING.html) in any browser: a
self-contained run map with every act figure embedded.

## What's in here

```
A-CDM-Decision-Hub-Talk-Track.md   the speaker script (the narrative spine)
DEMO_QUESTIONS.md                  the 5 acts as plain-English questions
SNOWSIGHT_DEMO.md                  3-question Snowsight talk track
prep_demo.py                       the pre-flight gate
rai_code/manual/
  eham_acdm.py                     the PyRel ontology (source of truth)
  demo_queries.py                  the 5 act queries
  eham_acdm_demo.ipynb             local notebook
agent/
  deploy.py, queries.py            Cortex agent deploy + the query catalog
data/
  build_eham_demo_data.py          deterministic generator (seed=42)
  load_to_snowflake.sh             idempotent loader
  out/                             generated DDL / reference / validation SQL
build/generate_demo_figures.py     result figures for RUNNING.html
```

[CLAUDE.md](CLAUDE.md) is the full orientation: Snowflake state, reasoner
engine names and sizes, measured timings, and the anchored demo numbers.

## The data

320 flights for 2026-10-14, generated deterministically (seed=42) and loaded
into `ACDM_DEMO.EHAM` on the sales-engineering Snowflake account. The storm
window (Act 4) is backed by a 120-row `STORM_SLOT` table; the operational
cascade (Act 2) lives in `SLOT_BLOCK`. Anchored numbers reproduce exactly from
`data/out/eham_demo_validation.sql`. Recreate the dataset with
`data/build_eham_demo_data.py` then `data/load_to_snowflake.sh`.

## Requirements

- A `rai` connection profile in `~/.snowflake/connections.toml` (account
  `ajb85638`, warehouse `RAI_XS`) and the RelationalAI Native App installed on
  the account.
- Python 3.13 and [uv](https://docs.astral.sh/uv/); the `snow` CLI.

## Note

This is a sales-engineering demo. The flight schedule is synthetic; it is not
operational A-CDM data.
