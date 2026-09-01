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
- Tested on AMD, NVIDIA, TSLA, COIN, and CRWD acquired-data runs using timestamped transcripts and high-frequency or minute-bar market rows
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

Earnings calls release information over time, not all at once. A prepared remark, a margin comment, a guidance detail, or a Q&A answer can become observable at a precise second during the call.

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
| Verification | 30 local tests passing with a project-local pytest temp directory |

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

The project currently has five modeled acquired-data examples, plus additional transcript-only acquisitions waiting on suitable high-frequency market data.

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
| TSLA Q2 2026 | 2m | 480s | 480s | 0.2000 | 3.47s | -0.466% | 0.1207 |
| COIN Q2 2026 | 2m | 240s | 240s | 0.2000 | 3.47s | -0.070% | 0.0133 |
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

I extended the acquired-data workflow to six additional public-company calls: TSLA, COIN, SNOW, CRWD, MU, and AVGO. All six were converted into timestamped transcript segment files. TSLA, COIN, and CRWD had enough publicly available sub-hour extended-hours market data to run the model.

[Download the expansion summary CSV](/assets/information-diffusion-2026-expansion-summary.csv)

| Ticker | Call | Status | Market data | T50 | T90 | Total impact | R^2 | Placebo R^2 percentile |
|---|---|---|---|---:|---:|---:|---:|---:|
| TSLA | Q2 2026 | Modeled | Yahoo 2-minute extended-hours bars | 480s | 480s | -0.466% | 0.1207 | 95% |
| COIN | Q2 2026 | Modeled, low event count | Yahoo 2-minute extended-hours bars | 240s | 240s | -0.070% | 0.0133 | 10% |
| CRWD | Q2 FY2027 | Modeled | Yahoo 1-minute extended-hours bars | 300s | 600s | -0.470% | 0.0315 | 60% |
| SNOW | Q1 FY2027 | Transcript only | No sub-hour public Yahoo history available for the call date | n/a | n/a | n/a | n/a | n/a |
| MU | Q3 2026 | Transcript only | No sub-hour public Yahoo history available for the call date | n/a | n/a | n/a | n/a | n/a |
| AVGO | Q2 2026 | Transcript only | No sub-hour public Yahoo history available for the call date | n/a | n/a | n/a | n/a | n/a |

TSLA price and event alignment:

![TSLA price and event alignment](/assets/information-diffusion-tsla-price-events.png)

TSLA cumulative absorption:

![TSLA cumulative absorption](/assets/information-diffusion-tsla-cumulative-absorption.png)

COIN price and event alignment:

![COIN price and event alignment](/assets/information-diffusion-coin-price-events.png)

COIN cumulative absorption:

![COIN cumulative absorption](/assets/information-diffusion-coin-cumulative-absorption.png)

CRWD price and event alignment:

![CRWD price and event alignment](/assets/information-diffusion-crwd-price-events.png)

CRWD cumulative absorption:

![CRWD cumulative absorption](/assets/information-diffusion-crwd-cumulative-absorption.png)

This batch is useful as a coverage expansion, but it is not directly comparable to the Databento-based AMD and NVIDIA runs. The newer runs use MarketBeat / Quartr speaker-turn timestamps rather than official VTT cue timing, and Yahoo close bars rather than true BBO quotes. COIN is especially low-signal after filtering because only two modeled events remained.

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

The NVIDIA R^2 improved meaningfully versus the unfiltered baseline, but T50, T90, and lambda varied across resolutions. That is a useful caution flag: the signal looks real enough to investigate, but not yet stable enough to treat as a production trading model.

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

The real NVIDIA timing ranked near the top of the 20 placebo runs, which is encouraging. The run count is still small, so the percentile should be treated as a diagnostic clue rather than statistical proof.

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

Latest local verification:

```text
30 passed, 4 expected warnings
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

The next serious research steps are to add a stricter provenance validator, redesign the distributed-lag matrix around clock time for dense event windows, and repeat the acquired-data run across more companies and quarters to test out-of-sample stability.
