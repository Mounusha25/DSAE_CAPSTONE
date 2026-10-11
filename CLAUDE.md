# CLAUDE.md

Guidance for working in this repository. Read this first; details live in `README.md` and `docs/`.

## What this project is
FSE 570 data science capstone (ASU, Fall 2026): **"Carbon, Water, and Reliability: A Decision Framework for
Arizona's AI Data Center Boom."** One framework for three linked decisions, using public data only:

| Question | Method | Entry point |
|---|---|---|
| RQ1 When to run flexible workloads | LightGBM + temporal CNN forecasts of grid carbon intensity; simple scheduler | `forecast_v2.py`, `forecast_nn.py`, `schedule_eval.py` |
| RQ2 Where to site | Transparent multi-criteria score over 0.25-degree cells; logistic / LightGBM + SHAP | `build_sites.py`, `rq2_siting.py` |
| RQ3 Who pays (reliability) | Fixed-effects panel regression; PyMC hierarchical model | `rq3_model.py`, `bayes_rq3.py` |
| Water (extension) | Wet-bulb cooling water + generation-side water; carbon-vs-water frontier | `water_frontier.py`, `water_scarcity.py` |

The user is working **alone**. The timeline, phases and role assignments in the original proposal are
"just for namesake": ignore them unless asked. Aim for the most that is achievable and defensible.

Status: Midpoint 1 is complete and packaged. Remaining work is the full scope (see "Deferred" below).

## Commands
```bash
.venv/bin/python src/<script>.py        # always use the project venv; scripts take no required args
bash src/download_static.sh             # eGRID, LBNL, EIA-861 files (static downloads)
```
Full run order is in `README.md`. Secrets live in `.env` (`EIA_API_KEY`, git-ignored): never print, commit
or paste it. `data/` is git-ignored. No GPU is needed (CPU only, Apple M4 was enough).

## Layout
```
src/            pipeline, models, audits (one script per step; each docstring states inputs and outputs)
data/raw/       downloaded sources;  data/processed/  cleaned tables the models read
results/        metrics CSVs, results/figures/ for plots, results/_audit_before/ for pre-fix snapshots
docs/           MIDPOINT1_REPORT.md, data_dictionary.md, EDA.md, AUDIT.md, regulatory_context_check.md
```
Unified tables: `master_ba_hour.parquet` (BA x hour), `master_utility_year.parquet`, `sites_scored.parquet`.

## Rules for working here
1. **Do not assume. Verify or label it.** Every number in a report must trace to a file in `results/` or a
   script. If something comes from memory or a web search, say so. This project already had an unsourced hub
   list and an undercounted queue column; see `docs/AUDIT.md`.
2. **Report nulls and limits plainly.** RQ3 is a null result with bounds; the siting label is constructed, so
   its AUC is a reconstruction check, never validation. Do not write "validated" for either.
3. **Flag, don't impute.** Gaps in EIA-930 are flagged and excluded, not interpolated.
4. **Chronological splits only** for time series (train 2021-2023, early-stop 2024, test 2025). Compare
   models on identical rows. Design choices made after seeing test results must be re-tested on another year
   (`audit_rolling_origin.py`).
5. **Name the primary model before looking at the test set.** LightGBM (residual over persistence) is
   primary; the neural net is secondary and single-seed.
6. **Scenario estimates are not measurements.** Water factors are Macknick et al. (2012) medians; PUE 1.2 and
   the on-site cooling curve are assumptions. Always show the sensitivity runs.
7. After changing a data-building script, **re-run downstream scripts and update the docs and figures** that
   quote its numbers (report, data dictionary, deck). Keep a pre-change snapshot when a fix might move results.
8. Cite figures from the source files, not from earlier chat messages. Numbers have been corrected before.

## Gotchas learned the hard way
- **EIA-930 timestamps**: treated as UTC hours; Arizona has no DST, so local = UTC-7. EIA documents
  hour-ending labels; results are robust to a +-1 hour offset but the convention is unresolved.
- The local-time year includes 7 hours of Dec-2020 at the series start: filter `year >= 2021` for annual stats.
- **LBNL Queued Up has no load or data-center requests** (generation/storage only). Capacity is
  `mw_1 + mw_2 + mw_3`; the county field has a "Mojave" typo. RQ3 exposure is EIA-861 commercial +
  industrial MWh **per total customer**, a diluted proxy.
- **Counties**: use the Census TIGERweb point-in-polygon lookup (`build_sites.py`), not nearest-well labels.
  La Paz is FIPS 012 (an even code). Queue strain is county-level, so every cell in a county shares it.
- **GWSI**: `WL_DTW == 0` with no date is a placeholder (about 24% of sites). Only sites measured since 2015
  are used. Depth is a cross-section, not a time series.
- **EIA-861**: SAIDI without major events is missing for about 12% of rows, and 15% of utilities switch
  between IEEE and other standards. Values far from a utility's own median are probable unit errors.
- **linearmodels / pandas**: `row.pct_change` on a Series collides with a method; use `row["pct_change"]`.
- **PyMC**: the posterior variable `nu` is only the Gamma part (the model adds 2). `az.summary` rounds, so
  read the saved trace for exact posterior numbers.
- **External APIs**: Open-Meteo rate-limits the free tier (use NASA POWER). Overpass is often busy; retry the
  `z.overpass-api.de` mirror. The old USGS `waterservices` endpoint is decommissioned (use the OGC API).
- Carbon intensity is **production-based**: SRP looks clean mainly because Palo Verde is reported there.
  Do not use the multi-site shifting numbers; they are artifacts of that accounting and the hydro assumption.

## Deferred to the full scope (approved by the user, rough priority)
1. EIA-860 plant cooling types; separate CSP (Solana) from PV in generation-side water.
2. Marginal vs average emissions from EIA-930; test whether scheduling conclusions change.
3. Revealed-preference siting from real data-center locations (OSM) with infrastructure features and
   spatial cross-validation, replacing the constructed label.
4. Stage 1 siting optimizer + sensitivity frontier (needs service-territory polygons).
5. Data extensions: EIA-930 back to 2018 and more BAs, interchange for consumption-based carbon, LBNL queue
   wait times from queue dates, more EIA-861 tables, ADWR Wells55, ACC docket text.
6. A data-center-specific RQ3 exposure (Virginia / ERCOT event study) or an explicit power analysis.
7. Conformal forecast intervals; multi-seed neural net.
8. Final report, a small public tool, a brief for the ACC docket; correct the proposal's citations and the
   Colorado River figure (about 27% for 2027-28, not 16-20%).

## Style
Match the surrounding code: scripts are single-purpose with a module docstring stating inputs, outputs and
the assumptions made; pandas for tables, parquet for processed data, no notebooks. Write documentation in
plain, specific language and state limits next to the claim they limit.
