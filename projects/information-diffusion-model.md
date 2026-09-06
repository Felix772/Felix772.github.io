---
layout: page
permalink: /projects/information-diffusion-model/
title: Information Diffusion Model
---

## Information Diffusion Model

- Built a Python research toolkit for measuring how quickly earnings-call information appears in stock prices
- Converts timestamped call transcripts into event-level information shocks
- Aligns those shocks with high-frequency stock, market benchmark, and sector benchmark quote data
- Estimates abnormal returns, distributed-lag response curves, T50, T90, exponential lambda, and half-life
- Adds acquired-data filtering, event clustering, multi-resolution checks, and deterministic placebo diagnostics
- Expanded to eight stocks, 207 cataloged calls, and 137 newly modeled quarters with explicit timing and price-coverage audits
- Maintained as a Git-tracked Python package with source, tests, scripts, and docs separated from local data and generated outputs
- [GitHub](https://github.com/Felix772/Information_Diffusion_Model)
- [Related Agent](https://github.com/Felix772/earnings-call-intelligence-agent)

> Research support only: this project is not investment advice and does not generate buy, sell, or hold recommendations.

## Table of contents
- [Problem](#problem)
- [Research Pipeline](#research-pipeline)
- [Repository Status](#repository-status)
- [Information Shock](#information-shock)
- [Acquired Data Runs](#acquired-data-runs)
- [2026 Expansion Batch](#2026-expansion-batch)
- [Diagnostics](#diagnostics)
- [Project Structure](#project-structure)
- [Testing](#testing)
- [Limitations](#limitations)

---

## Problem

Earnings calls release information over time, not all at once. Prepared remarks, margin comments, guidance details, and Q&A answers arrive throughout the call. The precision with which they can be located depends on the transcript source.

This project asks a narrow research question:

> Once a piece of call information becomes observable, how quickly does the market incorporate it into price?

The goal is not to predict the full earnings reaction from scratch. The goal is to build an auditable event-time research pipeline that connects:

- timestamped transcript evidence
- event-level financial information shocks
- high-frequency price movement
- diffusion speed diagnostics

---

## Research Pipeline

```text
timestamped earnings-call transcript
  -> financial information event extraction
  -> novelty, materiality, direction, and shock scoring
  -> optional event filtering and clustering
  -> stock / QQQ / SOXX quote alignment
  -> abnormal return calculation
  -> distributed-lag model
  -> T50, T90, lambda, half-life, placebo diagnostics
```

Core design choices:

- transcript and market timestamps are normalized to UTC
- event timing uses the moment a transcript segment becomes observable
- novelty is calculated only against earlier information, avoiding future transcript leakage
- market response is benchmark-adjusted with pre-call beta estimation where data is available
- synthetic mode is separate from acquired-data mode, so real runs do not silently fall back to fake data

---

## Repository Status

The current repository is organized as a reusable Python research package rather than a one-off notebook.

| Area | Current state |
|---|---|
| Repository | [Felix772/Information_Diffusion_Model](https://github.com/Felix772/Information_Diffusion_Model) |
| Package | `earnings-diffusion`, Python 3.12+ |
| Tracked source | `README.md`, `pyproject.toml`, `scripts/`, `src/`, `tests/`, `.gitignore`, `.gitattributes` |
| Ignored local artifacts | `data/`, `outputs/`, Python bytecode, test scratch folders, Matplotlib cache, local environment files |
| Verification | 36 local tests passing; timestamp-offset, export, and model-rank checks passed |

The repository README now documents both local-file workflows and provider-backed workflows. External sources such as Q4 captions, MarketBeat / Quartr transcripts, Yahoo chart bars, Alpha Vantage transcripts, and Databento market data are treated as optional inputs with explicit provenance notes.

Generated reports and acquired datasets are kept out of version control by default. This avoids accidentally publishing large local files, provider caches, or licensed transcript / market data while keeping the code path reproducible.

---

## Information Shock

Each extracted event is represented as a signed information shock:

```text
shock = direction x surprise x novelty x materiality
```

The components are kept separate so the model can distinguish:

- whether the information is positive or negative
- whether it is new relative to earlier call content
- whether it is economically material
- whether it carries enough signal to model

For acquired-data experiments, the pipeline can filter low-signal events and cluster nearby events into aggregate shocks. This keeps the analysis focused on measurable information bursts instead of every transcript sentence.

---

## Acquired Data Runs

The initial five acquired-data examples below are preserved as earlier experiments. The expanded study now catalogs 207 calls; the September 2026 expansion and its separately audited results follow in the next section.

| Call | Transcript source | Market source | Segments | Extracted events | Retained events | Modeled events |
|---|---|---|---:|---:|---:|---:|
| AMD Q2 2026 | ASR from acquired webcast audio | Databento cache | 845 | 229 | 57 | 44 |
| NVIDIA Q2 FY2027 | Official Q4 post-event captions | Databento cache | 726 | 189 | 31 | 24 |
| TSLA Q2 2026 | MarketBeat / Quartr speaker-turn timestamps | Yahoo extended-hours 2-minute bars | 109 | 83 | 11 | 11 |
| COIN Q2 2026 | MarketBeat / Quartr speaker-turn timestamps | Yahoo extended-hours 2-minute bars | 72 | 40 | 2 | 2 |
| CRWD Q2 FY2027 | MarketBeat / Quartr speaker-turn timestamps | Yahoo extended-hours 1-minute bars | 103 | 209 | 51 | 35 |

Main results after filtering and clustering:

| Call | Resolution | T50 | T90 | Lambda | Half-life | Total impact | R^2 |
|---|---:|---:|---:|---:|---:|---:|---:|
| AMD Q2 2026 | 1s | 61s | 94s | 0.0101 | 68.92s | 0.118% | 0.0160 |
| NVIDIA Q2 FY2027 | 1s | 50s | 58s | 0.0179 | 38.65s | 0.135% | 0.0417 |
| TSLA Q2 2026 | 2m | 480s | 480s | 0.0027 | 253.71s | -0.466% | 0.1207 |
| COIN Q2 2026 | 2m | 240s | 240s | 0.0054 | 129.23s | -0.070% | 0.0133 |
| CRWD Q2 FY2027 | 1m | 300s | 600s | 0.0027 | 253.19s | -0.470% | 0.0315 |

Baseline comparison:

| Call | Baseline R^2 | Filtered / clustered R^2 | Change |
|---|---:|---:|---:|
| AMD Q2 2026 | 0.0040 | 0.0160 | +0.0120 |
| NVIDIA Q2 FY2027 | 0.0060 | 0.0417 | +0.0357 |

NVIDIA event alignment:

![NVIDIA price and event alignment](/assets/information-diffusion-nvda-price-events.png)

NVIDIA cumulative absorption:

![NVIDIA cumulative absorption](/assets/information-diffusion-nvda-cumulative-absorption.png)

---

## 2026 Expansion Batch

Updated September 6, 2026. I expanded the study across **AMD, AVGO, COIN, CRWD, MU, NVDA, SNOW, and TSLA**, adding **196 transcripts and 21,880 timestamped speaker-turn segments**. The local catalog now contains **207 unique calls**.

The expansion completed **138 model runs**, including **137 quarters not previously modeled** and one repeat of an existing quarter. The new quarters contain **3,913 modeled information events**; all 138 runs contain 3,966 events. The counts below refer to the 137 newly modeled quarters.

| Stock | Added transcripts | Added segments | New modeled quarters |
|---|---:|---:|---:|
| AMD | 27 | 3,152 | 20 |
| AVGO | 26 | 2,276 | 10 |
| COIN | 21 | 1,913 | 13 |
| CRWD | 26 | 2,805 | 13 |
| MU | 25 | 2,576 | 24 |
| NVDA | 22 | 1,996 | 20 |
| SNOW | 23 | 2,983 | 17 |
| TSLA | 26 | 4,179 | 20 |

### Results and downloads

The table shows the latest successfully modeled call for each stock, selected by call date. Quarter labels follow each company's fiscal calendar. T50 and T90 are raw first threshold crossings; half-life comes from a separate exponential fit, so it need not match T50.

| Call | Modeled events | T50 (s) | T90 (s) | Fitted half-life (s) | R² |
|---|---:|---:|---:|---:|---:|
| AMD Q4 2025 | 39 | 120 | 120 | 1,039.0 | 0.012 |
| AVGO Q3 2026 | 31 | 360 | 480 | 309.3 | 0.028 |
| COIN Q4 2025 | 9 | 660 | 660 | 1,679.2 | 0.047 |
| CRWD Q4 2026 | 45 | 420 | 660 | 428.1 | 0.015 |
| MU Q2 2026 | 49 | 420 | 420 | 363.6 | 0.010 |
| NVDA Q4 2026 | 24 | 120 | 120 | 304.5 | 0.056 |
| SNOW Q2 2027 | 23 | 480 | 660 | 406.9 | 0.039 |
| TSLA Q4 2025 | 11 | 120 | 360 | 399.6 | 0.049 |

**Every one of the 137 new cumulative responses reverses direction; 123 also overshoot the normalized response range.** These values therefore do not establish stable absorption times or a ranking of which stock processes information fastest. Sixteen fitted half-lives fall below the price-data resolution and are unresolved. Calls with fewer than ten modeled events carry an additional low-sample caution.

- [Download all 137 newly modeled quarters](/assets/information-diffusion-new-model-results-20260906.csv): timing, event counts, raw T50/T90, lambda, half-life, R², placebo diagnostics, source links, and measurement flags.
- [Download the 207-call coverage catalog](/assets/information-diffusion-call-catalog-20260906.csv): completed runs, preserved earlier models, and excluded calls with reasons.

### Timestamp audit

I checked **69 scheduled call starts against company announcements and corrected 68**. All 65 initially flagged anchors were resolved, and a wider check identified four additional errors. Corrections include wrong calendar dates, time-zone offsets, and unusual but valid morning or quarter-hour starts. Original source values and superseded models remain archived locally; only models matching the current call anchor enter the combined results.

Speaker-turn offsets come from MarketBeat / Quartr. Segment ends use the next turn's start, with a bounded word-count estimate for final or same-offset turns. Absolute timestamps add these offsets to the scheduled start, with daylight-saving time handled explicitly. Actual webcast delays and word-level timing are not independently measured. Other call anchors remain labeled source-reported rather than independently verified.

### Market data and model controls

I acquired **75 monthly minute-price subsets spanning January 2020 through March 2026** from public [ggaddam](https://huggingface.co/datasets/ggaddam/OHLCV-1m) and [mito0o852](https://huggingface.co/datasets/mito0o852/OHLCV-1m) archives attributed to Finnhub. Recent calls use Yahoo extended-hours bars. The historical archives are community redistributions with unverified exchange provenance. Bar closes stand in for bid, ask, and last; they do not provide actual quote spreads.

Bar timestamps move to interval ends to avoid treating a closing price as observable early. Historical coverage requires at least 70% of expected call minutes, no gap above five minutes, at least 20 pre-estimation observations, and prices continuing ten minutes beyond the transcript. Sparse QQQ coverage triggers a SPY check. Conflicting duplicate prices or insufficient coverage exclude a call. Missing bars remain missing in storage; the existing return pipeline forward-fills them during alignment.

The new runs use a 600-second maximum lag, 15-second event clustering, and a post-event window extended by one price interval to observe the final lag. Beta estimation excludes the hour before the call. Batch fits include deterministic timestamp-shift placebos, with iteration counts retained in the download. All 138 completed model designs passed the rank check.

An independent FirstRate comparison of the Tesla July 19, 2023 call window found 301 matching TSLA minutes and 276 matching QQQ minutes, with median absolute close differences of 0.215 and 0.000 basis points respectively. This single-window check does not validate the whole archive.

The catalog records **57 calls excluded by coverage or model checks, eight transcript-only calls without a matching monthly price file, and four preserved earlier models** outside the expansion results. These minute-bar experiments use different data and settings from the initial one-second AMD and NVIDIA examples above.

---

## Diagnostics

The project includes checks that are meant to make a higher R^2 harder to overinterpret.

### Multi-resolution stability

NVIDIA was run at 1-second, 5-second, 10-second, and 30-second resolution from the same acquired dataset:

| Resolution | T50 | T90 | Lambda | R^2 | Total impact |
|---|---:|---:|---:|---:|---:|
| 1s | 50s | 58s | 0.0179 | 0.0417 | 0.0013 |
| 5s | 45s | 50s | 0.0195 | 0.0328 | 0.0012 |
| 10s | 10s | 50s | 0.0309 | 0.0629 | 0.0016 |
| 30s | 90s | 120s | 0.0165 | 0.0253 | 0.0023 |

The NVIDIA R^2 improved meaningfully versus the unfiltered baseline, but T50, T90, and lambda varied across resolutions. This variation limits the interpretation of any single speed estimate; the experiment remains exploratory.

### Placebo tests

Placebo diagnostics shift event timestamps within the same call window and refit the model against the same market rows. This checks whether the real transcript timing performs better than plausible pseudo-event timings.

For NVIDIA Q2 FY2027:

| Metric | Result |
|---|---:|
| Placebo iterations | 20 |
| Real R^2 percentile | 95% |
| Real absolute total-impact percentile | 70% |
| Placebo R^2 median | 0.0322 |
| Placebo R^2 p95 | 0.0415 |
| Real R^2 | 0.0417 |

In this earlier example, the real NVIDIA timing ranked near the top of the 20 placebo runs. The run count is still small, so the percentile should be treated as a diagnostic clue rather than statistical proof.

---

## Project Structure

```text
earnings-diffusion/
  src/earnings_diffusion/
    models/       typed transcript, event, market, and call schemas
    ingestion/    transcript, ASR, PDF, Alpha Vantage, and acquired-data loaders
    nlp/          financial event extraction, novelty, and shock scoring
    market/       quote loading, mid-price returns, alignment, abnormal returns
    diffusion/    distributed-lag model, exponential fit, T50/T90 metrics, plots
    simulation/   synthetic data generator for controlled experiments
    pipeline.py   synthetic, real-call, and acquired-data orchestration
  scripts/
    run_demo.py
    run_real_call.py
    run_acquired_dataset.py
    run_alpha_vantage_databento.py
    run_pdf_databento.py
    acquire_q4_caption_transcript.py
    acquire_marketbeat_yahoo_batch.py
  tests/
  data/            ignored local/acquired data
  outputs/         ignored generated reports and charts
```

The acquired-data path is intentionally separate from the synthetic demo. If a required real transcript or market file is missing, the loader raises an error instead of fabricating data.

---

## Testing

The current test suite covers both synthetic and acquired-data behavior:

- event filtering
- event clustering
- multi-resolution output files
- placebo reproducibility
- acquired results JSON fields
- real-call timestamp and market-data handling
- core diffusion metrics and summary outputs
- primary-source anchor overrides and disputed-anchor exclusion
- interval-end bar timing and minimum market coverage

Latest local verification:

```text
36 passed, 3 expected fixture warnings
```

---

## Limitations

This is a research prototype, not a deployable trading system.

Current caveats:

- the information extractor is still rule-based rather than a validated analyst-grade event classifier
- surprise is limited unless external consensus expectations are supplied
- R^2 is in-sample and should be replaced or supplemented with out-of-sample validation before quantitative model development
- placebo percentiles are useful diagnostics, but they are not a full causal identification strategy
- different transcript sources can affect timing granularity, since AMD used ASR segments while NVIDIA used official caption segments
- the current event-window regression concatenates per-event return windows, so a future clock-time lag design should handle dense or overlapping events more cleanly

Next steps are to verify the remaining source-reported anchors, redesign the distributed-lag matrix around clock time for overlapping events, add release-text novelty controls, and evaluate stability on held-out quarters. The expanded sample provides more observations, but overlapping return windows, in-sample R², and HC1 errors do not establish causality.
