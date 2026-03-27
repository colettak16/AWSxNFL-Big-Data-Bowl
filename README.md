# AWS x NFL Big Data Bowl

**Authors:** Kyle Coletta & Connor Feldman

## Project Overview

This project is a data science analysis of the 2023 NFL season submitted to the AWS x NFL Big Data Bowl competition. It uses official NFL player tracking data — frame-by-frame GPS coordinates for every player on every passing play — to investigate six research questions spanning offensive strategy, individual player performance, and environmental factors. The analysis is published as an interactive Quarto book deployed to GitHub Pages.

## Tech Stack

- **R** — primary analysis language
- **Quarto** — literate programming framework for reproducible, book-style output
- **dplyr / tidyverse** — data wrangling (filtering, joining, aggregating across 18 weeks of CSVs)
- **ggplot2** — all visualizations (boxplots, density plots, faceted grids, choropleth maps)
- **maps** — US state-level choropleth for ADOT geographic analysis
- **lubridate** — date parsing for BYE week and game schedule handling
- **reticulate** — Python interop bridge
- **GitHub Pages** — static hosting of the rendered `/docs` output

## Architecture & Design Decisions

The project is structured as a **Quarto book** — each analysis chapter is a self-contained `.qmd` file that loads raw data, performs feature engineering, and renders visualizations inline. There is no shared R package or module layer; data loading and preprocessing are repeated per-chapter, which keeps each chapter independently reproducible at the cost of some redundancy.

**Data layout:** The tracking data is split into 36 CSVs (18 `input_2023_wNN.csv` + 18 `output_2023_wNN.csv`). Each analysis loads only the weeks it needs, then `bind_rows()` them into a single data frame. The `supplementary_data.csv` (18K rows, 41 columns of play-level metadata) is the join key that links tracking frames to play context like formation, coverage type, and EPA.

**Rendering:** `_quarto.yml` configures the book to output to `/docs` with `code-fold: true`, so published pages show narrative and figures by default with code hidden behind a toggle — appropriate for a competition submission read by non-technical judges.

**Statistical rigor:** Each analysis enforces minimum sample size cutoffs (e.g., 50+ routes per receiver, 100+ plays per formation-coverage combination) before drawing conclusions, and the one formal regression (timezone analysis) reports 95% confidence intervals explicitly.

## Key Features

- **Formation vs. Coverage Effectiveness (EPA):** Assigns offensive/defensive WPA correctly based on `possession_team` vs. `home_team_abbr`, then compares median EPA across all formation × coverage combinations with 100+ plays. Finds Pistol vs. Cover 3 Zone produces the highest median EPA; Cover 3 Zone appears in the top 2 worst defenses against every formation.

- **Receiver Separation at Target:** At the ball-arrival frame (identified as `max(frame_id)` per play), computes Euclidean distance between the targeted receiver and every defender on the field, then takes the minimum — the "nearest defender distance." Averaged across 50+ targets per receiver, this surfaces elite route runners (Wan'Dale Robinson, Rashee Rice, Tyler Lockett, Jayden Reed) without relying on catch or yards-after-contact outcomes.

- **Play Action Effect on Separation:** Joins tracking data with `supplementary_data` on `play_action` flag, then overlays `geom_density` distributions of separation distance for Play Action vs. standard passes, faceted by route type (Hitch, Slant, Post, Corner, etc.). Finds Play Action consistently shifts the separation distribution rightward across all route types.

- **ADOT Geographic Analysis (Choropleth):** Tests the cold-weather-shortens-passes hypothesis by grouping weeks 1–4 as "Early" and weeks 15–18 as "Late," filtering to outdoor stadium teams only (excludes 10 dome teams by hardcoded list), mapping each team to its home state, and computing `Late_ADOT − Early_ADOT`. Renders as a US choropleth with a diverging blue-to-red color scale. Finds no consistent weather signal — several cold-climate teams (Seattle, Green Bay, Pittsburgh) show *increased* ADOT late season.

- **BYE Week Movement Analysis:** For six teams with known BYE week timing, calculates total distance traveled per player per play by summing frame-to-frame Euclidean displacement (`sqrt(dx² + dy²)`) across all tracking frames. Compares density distributions for the week before vs. the week after each team's BYE. Finds no meaningful change in movement intensity post-rest.

- **Timezone Travel Regression:** Maps all 32 teams to one of four time zones (ET, CT, MT, PT), computes the absolute timezone difference for each away game, and fits `point_diff ~ tz_diff` (where `point_diff = away_score − home_score`). The timezone slope is +0.79 points per zone crossed but non-significant (95% CI: −0.90, 2.49); the intercept of −3.58 (CI: −6.15, −1.00) confirms a significant home-field advantage independent of travel.
