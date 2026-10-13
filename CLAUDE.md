# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A course homework assignment ("Netflix Hit Predictor", deadline 2026/11/11 23:59), not a software project. There is no build, lint, or test suite. The work is a single Jupyter notebook plus a Word report.

- `netflix.ipynb` — the starter notebook. Code is filled into the empty cells (Part 1 (3), Part 3, Part 4). Final deliverable must be named `studentID_Netflix.ipynb`, keep its outputs, and **run top to bottom** showing how the final dataset, HitScore and model were produced.
- `studentID_netflix_report.docx` — the written answers. It contains questions the notebook does not (e.g. "select three important lines of code and explain input/output/what breaks if removed" for Parts 1 and 2, the class-distribution table, CV mean/std table, FP/FN error table, before/after modification table). Read it before deciding what the notebook must produce.
- `netflix_briefing_EN.pptx` — the assignment brief (data dictionary, grading, deliverables).

Grading weights: Part 1 20%, Part 2 5%, Part 3 50% (analysis/feature engineering 35%, HitScore design 15%), Part 4 25%, +2% bonus for uploading the AI-tool chat. Effort should follow these weights — Part 3 (open-ended HitScore definition) matters most.

## Environment

Use the existing venv in the parent folder: `../.venv`, a uv project defined by `../pyproject.toml` + `../uv.lock` (Python 3.12, pandas 3.x, scikit-learn 1.9, matplotlib, seaborn, xgboost; dev group: ipykernel, nbconvert). Do not create another environment or use `uv run --with ...` ad-hoc installs; system `python3` has no pandas. Add packages with `uv add` (or `uv add --dev`) from the parent folder so `pyproject.toml` and the lock file stay in sync — never `pip install` into the venv.

Run ad-hoc checks from this directory, since the notebook reads `./data` relative to CWD:

```bash
../.venv/bin/python script.py
```

Execute the whole notebook headlessly (the "runs top to bottom" requirement):

```bash
../.venv/bin/jupyter nbconvert --to notebook --execute --inplace netflix.ipynb
```

`--inplace` overwrites saved outputs; use `--output <path>` to test without touching the notebook.

The notebook metadata says Python 3.13, but the kernel actually used is the venv's 3.12.

## Data and pipeline

Three CSVs in `data/`:

- `netflix.titles.composite.csv` (691 rows, small) — the core table: one row per Netflix Top 10 title with viewing hours/weeks/year (2021–2022) and **pre-matched** TMDb/IMDb title + rating fields from a fuzzy-matching step.
- `imdb.titles.composite.csv` (~299k rows, 80 MB) and `tmdb.titles.v3.csv` (~217k rows, 79 MB) — TV-series-only metadata. Load with `low_memory=False`; loading takes noticeable time, so avoid re-reading them repeatedly. IMDb uses `\N` for missing values.

Notebook flow: keep rows matched in both sources (691 → 391) → left-merge IMDb extras on `imdb_title == primaryTitle` and TMDb extras on `tmdb_title == name` → feature engineering (`binge_velocity`, `bayesian_rating`) → user-defined HitScore and binary Hit label → classifier (Logistic Regression suggested) with 80/20 split, 3-fold CV on the training set only, test-set evaluation, FP/FN analysis, one model modification compared before/after.

Non-obvious facts that drive the design:

- **The title-based merge is many-to-many.** IMDb has ~23.6k duplicated `primaryTitle`s and TMDb ~13k duplicated `name`s (remakes, same-name shows across countries), so the merge inflates 391 → 801 rows. Part 1 (3) is about fixing this; the analytical unit should be one row per Netflix title (`key` is unique in the core table). Disambiguation signals available: `netflix_year_hint` vs IMDb `startYear` / TMDb `first_air_date`, and the core table's already-matched `imdb_numVotes` / `tmdb_vote_count` vs the candidates'.
- Duplicated rows from the merge would leak the same title into both train and test sets — dedupe (or group-split by `key`) before splitting.
- Movies cannot match the TV-only IMDb/TMDb files, so dropping unmatched rows introduces selection bias; the report expects this to be discussed.
- Some fuzzy matches in the core table are wrong (e.g. "The 100" has IMDb rating 9.4 from 12 votes; "1000 Miles from Christmas" matched TMDb "100 Miles From Nowhere"). Low vote counts are a red flag — this is the motivation for `bayesian_rating`.
- Target leakage: if HitScore is built from viewing hours/weeks/popularity, those same columns (and features derived from them such as `binge_velocity`) should not be used as model inputs, or the classifier trivially recovers the label. Pick model features that would be known about a new series concept (genre, language, seasons/episodes, network, year, etc.).
