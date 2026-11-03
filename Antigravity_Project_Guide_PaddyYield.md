# Full Build Guide — Climate-Smart Agriculture: Bi-Seasonal Paddy Yield Forecasting
### For use inside Google Antigravity IDE (Agent Manager)

This guide turns your Proposal + Workflow doc into a sequence of **copy-paste prompts** for the Antigravity agent, in the order you should run them. Each phase = one agent task. Wait for one to fully finish (and verify the Artifact it produces) before pasting the next — don't run them all in parallel on a single laptop, or the agent will collide on files/dependencies.

**Mode recommendation:** Use **Agent-driven ("Autopilot")** for Phases 0–2 and 6–8 (low risk, easy to inspect). Switch to **Plan mode** for Phases 3–5 (modeling) so the agent produces a Plan Artifact you can review/edit *before* it writes code — this matters most where wrong choices (leakage, bad splits) are expensive to undo.

---

## Phase 0 — Repo Scaffold & Environment

**Paste this first, in a new empty folder opened in Antigravity:**

```
Set up a new Python data science project called "paddy-yield-forecast" with this exact structure:

paddy-yield-forecast/
├── data/
│   ├── raw/            # untouched original datasets go here
│   ├── interim/        # partially cleaned data
│   └── processed/      # final model-ready datasets
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_preprocessing.ipynb
│   ├── 03_feature_engineering.ipynb
│   ├── 04_modeling.ipynb
│   └── 05_evaluation.ipynb
├── src/
│   ├── __init__.py
│   ├── data_loader.py
│   ├── preprocessing.py
│   ├── features.py
│   ├── models.py
│   └── visualization.py
├── dashboard/
│   └── app.py           # Streamlit dashboard
├── reports/
│   └── figures/
├── tests/
├── requirements.txt
├── .gitignore
└── README.md

Initialize a git repo. Create a Python virtual environment. Add a requirements.txt including: pandas, numpy, scikit-learn, lightgbm, matplotlib, seaborn, plotly, streamlit, scipy, statsmodels, jupyter, pytest, category_encoders. Install everything and confirm the environment works by printing library versions. Add a .gitignore suited for Python + data science (ignore data/raw, data/interim, .venv, __pycache__, .ipynb_checkpoints, *.pkl).

Do NOT push to GitHub yet — this stays local for now.
```

**Verify:** check the Artifact/plan shows all folders created and `pip list` confirms installs. If LightGBM fails to install (common on some Windows setups), tell the agent to try `pip install lightgbm --prefer-binary` or conda as fallback.

---

## Phase 1 — Add Your Datasets (three steps — don't add everything at once)

From your folder screenshot, you actually have 9 raw dataset folders, each likely containing multiple files (by year/district/table). They group into three categories, and they don't all get added at the same point in the pipeline:

| Your folder | Category | When to add |
|---|---|---|
| Kaggle Sri Lanka Rice Production & Climate | Primary (target + climate) | Now |
| Paddy average yield and production by district | Agriculture | Now |
| Paddy extent sown and harvested by district | Agriculture | Now |
| Paddy extent sown and harvested, average (national) | Agriculture | Now |
| Colombo Consumer Price Index (CCPI) | Macro — inflation | Later, at Phase 3 |
| National Consumer Price Index (NCPI) | Macro — inflation | Later, at Phase 3 |
| Expenditure Approach — Aggregate Demand | Macro — GDP | Later, at Phase 3 |
| Production Approach — GDP | Macro — GDP | Later, at Phase 3 |
| Sri Lanka Weather | Optional climate supplement | Only if EDA shows gaps |

You have **two GDP measures** (Expenditure vs Production approach) and **two inflation measures** (CCPI vs NCPI) — that's normal, CBSL publishes both as different ways of computing the same underlying thing. With a day left, don't try to use all four as separate model features; pick one GDP series and one CPI series as your primary macro variables and keep the other pair only as a footnote/cross-check if time allows. I've written the prompts below to have the agent pick and justify one of each rather than you deciding blind right now.

**Step 1 — copy folders in, now (before Phase 2 EDA):**

Copy these folders as-is into `data/raw/`:
- `data/raw/kaggle_rice_climate/`
- `data/raw/dcs_paddy_yield_district/`
- `data/raw/dcs_paddy_extent_district/`
- `data/raw/dcs_paddy_extent_national/`

Then paste:

```
I've placed four dataset folders in data/raw/: kaggle_rice_climate/, dcs_paddy_yield_district/, dcs_paddy_extent_district/, dcs_paddy_extent_national/. Each folder may contain multiple files (e.g. split by year or district).

In src/data_loader.py:
1. For each folder, list every file inside it and its format (csv/xlsx).
2. Write a function per folder that reads and concatenates all files in it into a single DataFrame (align columns, handle any header row differences between files).
3. Write a function `load_primary()` that returns all four as a dict of DataFrames.
4. For each resulting DataFrame print: shape, column names, dtypes, first 5 rows, unique values in the season column, and % missing values per column.

Run this and show me the output before we go further. Do NOT reference any macroeconomic data yet — that comes in a later step.
```

**Check manually:** confirm rainfall/temperature units, season label spelling ("Yala"/"Maha" — real datasets are inconsistent), and that the district-level DCS files and the national Kaggle file share a compatible year range. Fix now if wrong — decide here whether you'll actually use the district-level detail or aggregate it to national level to match the Kaggle set (aggregating is simpler and probably safer given the time you have).

**Step 2 — later, at the start of Phase 3 (preprocessing/merge), not now:**

Only once you're about to run the Phase 3 prompt below, copy in the four macro folders:
- `data/raw/cbsl_ccpi/`
- `data/raw/cbsl_ncpi/`
- `data/raw/cbsl_gdp_expenditure/`
- `data/raw/cbsl_gdp_production/`

This is annual data, not seasonal, so it has to be joined to the seasonal paddy/climate table by year (each year's GDP/inflation value applies to both its Yala and Maha seasons) — and since you have two GDP series and two CPI series, the Phase 3 prompt tells the agent to consolidate each folder, then pick one representative series for each and note why, rather than feeding in four overlapping macro columns.

**Step 3 — optional, only if needed:**

Only add the `Sri Lanka Weather` folder if Phase 2's EDA shows real gaps in the Kaggle dataset's rainfall/temperature coverage. It itself contains four sub-folders — you won't need all of them even if you do end up using this source:

| Sub-folder | Use if... |
|---|---|
| Sri Lanka Climate Data | This is your best bet if you need it — most likely already at monthly/seasonal aggregation, closest match to your target's Yala/Maha structure. |
| Sri Lanka Weather Data for All Districts | Only if you specifically want district-level climate detail (adds real merge complexity — skip unless you have time to spare). |
| Daily Summary of Sri Lanka's Weather | Skip unless the above two are missing something — daily data needs aggregating up to seasonal yourself before it's usable, which is extra work for one day left. |
| Sri Lanka Weather Dataset | Check what this actually contains before using — vague name, could overlap with one of the others above. |

If you do need this source, add only `data/raw/srilanka_climate/` (the "Sri Lanka Climate Data" sub-folder) rather than all four — otherwise skip the whole thing entirely if the primary dataset's climate columns already look complete.

---

## Phase 2 — Data Exploration (EDA)

```
In notebooks/01_eda.ipynb, perform exploratory data analysis on the four loaded agriculture/climate datasets (kaggle_rice_climate, dcs_paddy_yield_district, dcs_paddy_extent_district, dcs_paddy_extent_national):

1. Univariate analysis: histograms for all continuous variables (rainfall, temperature, paddy yield, extent sown/harvested), bar chart of record counts by season (Yala vs Maha).
2. Outlier detection: box plots for each continuous variable, flag values beyond 1.5*IQR, and report how many outliers per column.
3. Missing data: a heatmap or bar chart showing missingness per column across the ~70 years of records.
4. Duplicate detection: check for duplicate rows and duplicate (year, season) combinations, report counts.
5. Time trend: line plots of paddy yield over time, split by season (Yala vs Maha).
6. Consistency check: compare the Kaggle dataset's national yield figures against the DCS national/district datasets aggregated up — flag any year where they disagree noticeably, since we'll need to pick one as the authoritative target variable.
7. Correlation: a feature similarity/correlation matrix (heatmap) between rainfall, temperature, and yield (macroeconomic variables aren't loaded yet — that correlation comes after Phase 3's merge).

Save all figures to reports/figures/ as PNGs. Summarize the key data quality issues AND the Kaggle-vs-DCS consistency findings in a markdown cell at the end of the notebook — this will feed directly into the preprocessing plan, including which source becomes the authoritative Y_seasonal.
```

**This step matters most for your report's "Data Exploration" section — save the summary text, you'll reuse it almost verbatim in Methodology/Results.**

---

## Phase 3 — Data Preprocessing (Cleaning, Reduction, Transformation)

Switch to **Plan mode** here.

**Before pasting this prompt:** copy the four macro folders (`cbsl_ccpi/`, `cbsl_ncpi/`, `cbsl_gdp_expenditure/`, `cbsl_gdp_production/`) into `data/raw/` now, per Step 2 in Phase 1 above — this prompt consolidates and merges them in.

```
I've now added four macroeconomic dataset folders in data/raw/: cbsl_ccpi/, cbsl_ncpi/, cbsl_gdp_expenditure/, cbsl_gdp_production/ (each may contain multiple files by year).

In src/data_loader.py: consolidate each folder into one annual DataFrame (year, value). We have two inflation series (CCPI, NCPI) and two GDP series (Expenditure approach, Production approach) that measure overlapping things. Compare their coverage (year range, missing years) and pick ONE inflation series and ONE GDP series as our primary E_inf and E_gdp variables — print your reasoning (e.g. longer year coverage, fewer gaps, more standard for this kind of analysis). Keep the unused pair as a small comparison table in the notebook only, not as model features.

Merge the chosen annual GDP and inflation series into our seasonal paddy+climate table by year — since it's annual data, both the Yala and Maha rows for a given year get that year's values.

Then, based on the data quality issues found in notebooks/01_eda.ipynb, implement a preprocessing pipeline in src/preprocessing.py and demonstrate it in notebooks/02_preprocessing.ipynb. It must:

CLEANING:
- Remove exact duplicate rows and duplicate (year, season) merges, keeping the most complete record.
- Handle missing values: use median imputation for skewed continuous variables (rainfall, inflation) and mean imputation for roughly-normal ones (temperature, GDP growth) — justify the choice per column with a comment.
- Detect and treat noise/outliers using IQR-based capping (winsorization) rather than deletion, to preserve sample size given the small yearly record count.

REDUCTION:
- Perform attribute subset selection: drop or merge columns that are redundant or near-constant.
- If any regional-level granularity exists, aggregate to the seasonal-national level needed for the target variable.

TRANSFORMATION:
- Binarize the categorical season label (Yala/Maha) into a numeric S_binary variable (0/1), matching the proposal's mathematical formulation.
- Standardize all continuous variables (rainfall, temperature, GDP growth, inflation) using z-score scaling (StandardScaler), fit ONLY on a designated training split to avoid data leakage.
- Save the fitted scaler with joblib for reuse at inference time.

Merge the cleaned agriculture/climate data (using the authoritative yield source decided in Phase 2) with the chosen GDP and inflation series into a single model-ready table keyed by (year, season), save it to data/processed/model_ready.csv. Print before/after row counts and a summary table of the transformations applied.
```

**Sanity check before moving on:** open `model_ready.csv` yourself and eyeball 10 rows. Confirm the season binarization and scaling look right, and that you have enough rows left (70 years × 2 seasons should give you ~140 rows before any drops — flag it if preprocessing dropped a lot).

---

## Phase 4 — Feature Engineering

```
In src/features.py and notebooks/03_feature_engineering.ipynb, engineer additional features on top of data/processed/model_ready.csv:

- Lag features: previous season's and previous year's yield (yield_lag1, yield_lag2), since yield is likely autocorrelated.
- Rolling averages: 3-season rolling mean of rainfall and temperature.
- Interaction terms: rainfall × temperature, inflation × GDP growth.
- A drought/anomaly flag: seasons where rainfall is more than 1 std below the historical seasonal mean.

Re-save the enriched table to data/processed/model_ready_features.csv. Show a correlation matrix of the new features against the target yield to confirm they add signal and aren't redundant with existing columns.
```

---

## Phase 5 — Modeling & Evaluation (answers your Research Question 2)

Stay in **Plan mode** — this is the highest-stakes phase.

```
Using data/processed/model_ready_features.csv, implement in src/models.py and demonstrate in notebooks/04_modeling.ipynb and notebooks/05_evaluation.ipynb:

SPLIT STRATEGY:
- This is time-series data — use a chronological train/test split (e.g. last 15% of years as holdout), NOT a random split, to avoid leakage from future into past.
- Set up k-fold cross-validation using TimeSeriesSplit (not standard KFold) for hyperparameter tuning on the training portion.

MODELS TO COMPARE (this directly answers our research question: do ensemble methods outperform deep neural networks?):
1. Random Forest Regressor — tune n_estimators, max_depth, min_samples_leaf via grid/random search with TimeSeriesSplit CV.
2. LightGBM Regressor — tune num_leaves, learning_rate, n_estimators similarly.
3. A baseline: simple linear regression, to show whether the "advanced" methods are worth the complexity.
4. A small feed-forward neural network (e.g. Keras/PyTorch, 2-3 hidden layers) as the deep learning comparison point required by the research question.

METRICS:
- Report RMSE, MAE, and R² for every model on the holdout test set.
- Plot predicted vs actual yield for each model.
- Plot feature importance for Random Forest and LightGBM (this feeds the "which factors most strongly influence yield" discussion).
- Explicitly discuss overfitting/underfitting: show train vs test metric gap for each model.

Save the best model with joblib to a new models/ folder. Print a clean comparison table (model, RMSE, MAE, R²) I can paste directly into the report's Results section.
```

**Reality check on the neural net:** with ~70 years × 2 seasons of data you have very few rows for a DNN — expect it to underperform the tree-based models. That's a legitimate, reportable finding (small tabular data usually favors gradient boosting), not a failure — say so explicitly in your report rather than hiding it.

---

## Phase 6 — Analytical Dashboard

```
Build an interactive dashboard in dashboard/app.py using Streamlit that includes:
- Box plots of climate variables by decade (to spot outliers, per the proposal).
- Histograms and clustered bar charts of paddy production by season and (if available) region.
- The feature correlation/similarity matrix.
- A model comparison panel showing RMSE/MAE/R² for all trained models.
- A "predict yield" interactive form where a user inputs rainfall, temperature, GDP growth, inflation, and season, and gets a predicted yield from the best saved model.

Load data from data/processed/model_ready_features.csv and the saved model from models/. Run it locally and confirm it launches without errors on localhost.
```

---

## Phase 7 — Documentation & Report Drafting

```
Generate a draft final report in reports/final_report.md following this structure: Abstract, Introduction, Literature Review (leave placeholder citations for me to fill in — do not invent fake references), Methodology (summarize the preprocessing pipeline, feature engineering, and models exactly as implemented in src/), Results (pull in the actual metrics table and key figures generated earlier — reference the file paths), Discussion, Conclusion, References (IEEE style, empty template entries).

Also generate reports/individual_contribution_template.md with a table for all 5 team members and their stated roles: Pipeline Architecture Lead, Data Quality & Preprocessing Specialist, Categorical Engineering Specialist, Machine Learning Modeling Engineer, Descriptive Statistics & Visualization Lead — with blank cells for us to fill in specific contributions.
```

Fill in the Literature Review and References yourself — an agent should never fabricate citations for an academic submission; that's the one part of this you must write and verify manually.

---

## Phase 8 — GitHub (only after everything above runs cleanly end-to-end)

```
Everything runs correctly locally now. Prepare this repo for GitHub:

1. Review .gitignore to make sure data/raw and data/interim are excluded (keep data/processed and reports tracked, since they're small and needed for reproducibility) — confirm with me before committing anything that looks like a large binary file.
2. Write a clear README.md: project title, problem statement, folder structure, setup instructions (venv + requirements.txt), how to run each notebook in order, how to launch the dashboard.
3. Stage and commit everything with a clean initial commit message.
4. Do NOT push yet — show me `git status` and `git log` first so I can confirm before we push to the remote.
```

Once you review that output, create the GitHub repo yourself (or have the agent do `gh repo create` if the GitHub CLI is authenticated), add the remote, and push. Then have each teammate clone it and work in feature branches for their section rather than everyone editing the single laptop's copy directly.

---

## Order-of-operations checklist

- [ ] Phase 0 — scaffold + environment
- [ ] Phase 1 — datasets loaded & verified
- [ ] Phase 2 — EDA complete, issues documented
- [ ] Phase 3 — preprocessing pipeline, model_ready.csv verified by eye
- [ ] Phase 4 — feature engineering
- [ ] Phase 5 — RF vs LightGBM vs linear vs DNN comparison, metrics table
- [ ] Phase 6 — Streamlit dashboard runs locally
- [ ] Phase 7 — report draft + contribution template generated
- [ ] Phase 8 — GitHub push, README, team clones

## A few things worth doing yourself, not delegating to the agent
- Confirming units/column meanings after Phase 1 (agents guess from column names and can guess wrong).
- Literature review and references (academic integrity — must be real, must be yours).
- Sanity-checking the DNN vs tree-model result before you write conclusions about it.
- Deciding what's "good enough" R²/RMSE for your context — the agent can compute metrics but shouldn't be the one deciding your project is "done."
