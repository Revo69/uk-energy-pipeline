# UK Energy Balancing Mechanism Pipeline

A data engineering project that ingests, maps, reconciles, and cleans
UK electricity market data from Elexon's Balancing Mechanism Reporting Service (BMRS).

**Live dashboard:** _coming in Phase 5_

---

## What this project does

The UK electricity grid requires continuous balancing between generation and demand.
Elexon administers this process and publishes granular market data — but raw data
arrives inconsistent, with format mismatches between message types, irregular timestamps,
and BM Unit identifiers that need mapping to actual power plants.

This pipeline:
1. **Ingests** BOALF, BOD, and DISBSAD data from Elexon's public API daily
2. **Maps** BM Units to power plants, fuel types, and ETYS grid zones
3. **Reconciles** accepted bids (BOALF) against submitted offers (BOD), detecting discrepancies
4. **Cleans** time-series data — outliers, gaps, overlapping settlement periods
5. **Serves** analysis-ready datasets via a Streamlit dashboard

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    SOURCES                          │
│  Elexon BMRS API │ National Grid ESO │ OSUKED/TEC   │
└────────────┬────────────────────────────────────────┘
             │ Python grabbers (httpx + tenacity)
             ▼
┌─────────────────────────────────────────────────────────┐
│              BRONZE  (raw JSON → Supabase)              │
│  bronze.boalf_raw │ bronze.bod_raw │ bronze.disbsad_raw │
└────────────┬────────────────────────────────────────────┘
             │ dbt staging models
             ▼
┌─────────────────────────────────────────────────────┐
│              SILVER  (dbt intermediate)             │
│  int_bm_units_mapped                                │
│  int_boalf_bod_reconciled   ← core of this project  │
│  int_timeseries_cleaned                             │
└────────────┬────────────────────────────────────────┘
             │ dbt marts
             ▼
┌─────────────────────────────────────────────────────┐
│              GOLD  (analysis-ready)                 │
│  mart_balancing_actions │ mart_plant_registry       │
└────────────┬────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────┐
│              SERVING                                │
│  Streamlit dashboard │ dbt docs site                │
└─────────────────────────────────────────────────────┘

Orchestration: GitHub Actions (daily cron)
Storage: Supabase (PostgreSQL)
```

---

## Stack

| Layer | Tools |
|---|---|
| Ingestion | Python, httpx, tenacity |
| Storage | Supabase (PostgreSQL) |
| Transformation | dbt-core, dbt-postgres |
| Data quality | dbt tests, Great Expectations |
| Orchestration | GitHub Actions |
| Serving | Streamlit, Plotly |
| Local analytics | DuckDB |

---

## Project structure

```
uk-energy-pipeline/
├── ingestion/
│   ├── elexon/          # BOALF, BOD, DISBSAD grabbers
│   ├── neso/            # BM Unit reference data
│   └── reference/       # TEC register
├── dbt/uk_energy/
│   ├── models/
│   │   ├── staging/     # bronze → typed, renamed
│   │   ├── intermediate/# mapping, reconciliation, cleaning
│   │   └── marts/       # gold, analysis-ready
│   ├── seeds/           # BM unit → plant mapping CSVs
│   ├── macros/          # reusable SQL logic
│   └── tests/           # singular data quality tests
├── streamlit/           # dashboard
├── scripts/             # backfill, one-off utilities
├── docs/                # mapping decisions, edge cases
└── .github/workflows/   # CI/CD
```

---

## Data sources

| Source | URL | Notes |
|---|---|---|
| Elexon BMRS API | data.elexon.co.uk/bmrs/api/v1 | No API key required |
| OSUKED Power Station Dictionary | github.com/OSUKED/Power-Station-Dictionary | BM Unit → plant mapping |
| National Grid ESO | nationalgrideso.com | TEC register, ETYS zones |

---

## Known limitations & mapping assumptions

Documented in [`docs/mapping_decisions.md`](docs/mapping_decisions.md).

Key points:
- One physical power station can have multiple BM Units (per generating unit)
- Settlement periods are 30 minutes (48/day), except DST transitions (46 or 50)
- BOALF and BOD will not always perfectly align — discrepancies are logged, not silently dropped

---

## Getting started

```bash
git clone https://github.com/Revo69/uk-energy-pipeline
cd uk-energy-pipeline

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# fill in your Supabase credentials

python ingestion/elexon/fetch_boalf.py --date 2024-06-01
```

---

## Development roadmap

- [x] Phase 1 — Repo structure, first grabber, bronze layer
- [ ] Phase 2 — dbt staging models, seeds
- [ ] Phase 3 — Intermediate: mapping, reconciliation, cleaning
- [ ] Phase 4 — GitHub Actions automation
- [ ] Phase 5 — Streamlit dashboard, dbt docs
