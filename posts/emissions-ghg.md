---
title: "Detecting Methane Anomalies in Alberta with Satellite Data"
date: 2026-05-14
authors:
  - name: Paul Brunet
    url: https://github.com/p-brunet
description: >
  A geospatial data pipeline that cross-references Sentinel-5P satellite methane
  observations with Alberta Energy Regulator facility reports, built on DuckDB,
  Apache Iceberg, Airflow and dbt.
thumbnail: "https://raw.githubusercontent.com/p-brunet/emissions-ghg/main/outputs/visualizations/ch4_facilities_overlay.png"
tags:
  - data-engineering
  - energy
  - satellite
  - visualization
keywords:
  - methane
  - GHG
  - Sentinel-5P
  - TROPOMI
  - DuckDB
  - Apache Iceberg
  - Airflow
  - dbt
  - H3
  - Alberta
  - AER
---

<span class="badge badge-python">Python</span>
<span class="badge badge-duckdb">DuckDB</span>
<span class="badge badge-airflow">Airflow</span>
<span class="badge badge-dbt">dbt</span>
<span class="badge badge-docker">Docker</span>

# Detecting Methane Anomalies in Alberta with Satellite Data

**GitHub**: [p-brunet/emissions-ghg](https://github.com/p-brunet/emissions-ghg)

## Overview

This project had two parallel goals: **build a production-grade data engineering pipeline from scratch**, and apply it to a concrete environmental monitoring problem. The architecture is the primary deliverable. The methane analysis is what makes it real.

The applied question: oil and gas operators in Alberta are required to report their monthly emissions to the **Alberta Energy Regulator (AER)**. Satellite instruments independently measure atmospheric methane concentrations over the same territory. This pipeline ingests both sources, aligns them spatially using H3 hexagonal cells, and flags statistical inconsistencies between what was declared and what the atmosphere shows.

The intent is not to model correlation or prove fraud. It is to demonstrate that a geospatial data pipeline built entirely on public sources can produce actionable signals for regulatory cross-referencing.

:::{image} https://raw.githubusercontent.com/p-brunet/emissions-ghg/main/outputs/visualizations/ch4_facilities_overlay.png
:alt: CH₄ satellite overlay on Alberta facilities
:width: 60%
:align: center
:::

## The Problem

Satellite instruments measure atmospheric methane concentrations (in parts per billion) integrated over the full air column above a point on the ground. Regulators collect ground-level volumetric declarations from individual operators. These two datasets speak different languages; bridging them is non-trivial.

The main constraint is resolution: at H3 level 6, a single cell covers ~36 km² and may contain dozens of independent facilities, wetlands, and agricultural sources. Battery extraction sites are also not the dominant methane emitter in Alberta; livestock and agriculture contribute significantly to the regional signal.

## Data Sources

::::{grid} 1 1 2 2

:::{card} Sentinel-5P / TROPOMI (ESA)
Satellite observations of atmospheric CH₄ concentrations, downloaded automatically via the Copernicus Data Space API. Coverage: 17 months (Jan 2025 → May 2026), yielding **1,394,442 raw pixels** over Alberta.
:::

:::{card} Alberta Energy Regulator (AER)
Monthly battery reports from ~9,900 oil and gas facilities across Alberta. Downloaded manually (no API available) and loaded as CSV. Coverage: 14 reporting months, **9,945 facilities** tracked.
:::

::::

## Architecture

The pipeline follows a **medallion architecture**: three layers of increasing refinement, each serving a different purpose.

| Layer | What it contains |
|-------|-----------------|
| **Bronze** | Raw ingested data. Satellite pixels stored in Apache Iceberg on MinIO (partitioned by month). AER reports in DuckDB. |
| **Silver** | Cleaned and filtered data. Satellite pixels passing a quality score ≥ 0.5, with H3 hexagonal cell IDs assigned. AER facilities deduplicated and geocoded. |
| **Gold** | Analysis-ready tables. Satellite observations spatially joined with AER facility data per H3 cell and month. Anomaly flags computed from z-scores. |

The entire pipeline runs in **Docker**: Airflow orchestrates ingestion and scheduling, dbt handles all transformations from silver to gold, and DuckDB is the query engine throughout.

:::{image} https://raw.githubusercontent.com/p-brunet/emissions-ghg/main/outputs/visualizations/ch4_heatmap.png
:alt: CH₄ heatmap over Alberta
:width: 60%
:align: center
:::

## Spatial Join with H3

The key to linking satellite pixels and ground facilities is **H3**, Uber's open-source hexagonal grid system. Each satellite pixel and each facility is assigned to an H3 cell at resolution 6 (~36 km² per cell). This common grid makes spatial joins straightforward and deterministic, with no approximate geometry matching needed.

The tradeoff: at resolution 6, a single cell can contain up to **47 independent facilities**. The satellite signal is a cell-level average; attributing it to a specific operator is not possible without additional modelling (e.g., wind dispersion).

## Results

After filtering, the pipeline matched **73,908 facility-month pairs** where both satellite coverage and AER declarations were available. From these, **340 anomaly flags** were generated across three categories:

| Flag | Count | What it means |
|------|-------|---------------|
| `HIGH_CH4_LOW_REPORTED` | 146 | Satellite elevated, declared volumes below facility's historical 25th percentile |
| `HIGH_CH4_HIGH_REPORTED` | 155 | Both signals elevated, consistent with high-activity periods |
| `LOW_CH4_HIGH_REPORTED` | 39 | High declared volumes, satellite unusually quiet |

::::{grid} 1 1 2 2

:::{card} Anomaly flag distribution
```{image} https://raw.githubusercontent.com/p-brunet/emissions-ghg/main/outputs/visualizations/anomaly_flags.png
:alt: Anomaly flags
:class: chart-card-img
```
:::

:::{card} Monthly CH₄ vs. AER volumes
```{image} https://raw.githubusercontent.com/p-brunet/emissions-ghg/main/outputs/visualizations/monthly_ch4_vs_aer.png
:alt: Monthly CH₄ vs AER
:class: chart-card-img
```
:::

::::

No strong linear correlation was found between reported AER volumes and satellite CH₄ at H3 resolution 6, an expected result given that cells aggregate multiple independent facilities and background atmosphere. The correlation would likely improve at H3 resolution 7 or 8 (~5 km² and ~0.7 km² respectively).

## What's Next

The most interesting architectural evolution would be a **native geospatial pipeline**: rather than converting NetCDF satellite swaths to tabular rows early in the process, the bronze layer would retain the data in its original array format and the silver or gold layer would store it as **Zarr**, a cloud-native N-dimensional array format analogous to what Iceberg is for tables. Only the final analytical join would flatten the data to rows. This would preserve the full spatial resolution of TROPOMI retrievals and open the door to pixel-level corrections impossible once the data is tabularised.

On the pipeline side, finer H3 resolution (7 or 8) and a proper REST Iceberg catalog (replacing the SQLite-backed local catalog) are the two most direct improvements to the current architecture.

## Stack

| Tool | Role |
|------|------|
| `DuckDB 1.5.0` | Query engine across all layers |
| `Apache Iceberg (PyIceberg 0.9.0)` | Table format for satellite bronze layer |
| `MinIO` | S3-compatible object storage |
| `Apache Airflow 2.9.3` | Pipeline orchestration |
| `dbt-duckdb 1.8.2` | SQL transformations (silver → gold) |
| `H3 (DuckDB extension)` | Hexagonal spatial indexing, resolution 6 |
| `Docker Compose` | Full local stack (Airflow, MinIO, Postgres) |
| `Python 3.13` | Ingestion scripts and visualizations |

## Source Code

[github.com/p-brunet/emissions-ghg](https://github.com/p-brunet/emissions-ghg)
