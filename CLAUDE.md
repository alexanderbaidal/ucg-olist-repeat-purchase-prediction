# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The binding rules here are formalized in `.specify/memory/constitution.md` (v1.0.0) — this file is
the day-to-day operational detail; that one is the governance source of truth. Keep both in sync: a
change to a non-negotiable rule (scope discipline, the fixed seed, the artifact contract, the settled
modeling decisions, or the surgical-edit convention) must be reflected in both files together.

## What this repo is

Trabajo de Titulación (Maestría en Data Science e IA, UCG). Caso de negocio ficticio "ShopMart"
(caso-de-uso/data-driven-e-commerce.md) resuelto con el dataset público Olist Brazilian E-Commerce.
El problema elegido: predecir si un cliente hará una **segunda compra** a partir de las señales de su
**primer pedido**, complementado con una segmentación RFM. No es una app de producción: es un proyecto
académico de dos notebooks + un informe/presentación finales en `docs/`.

There is no application code, no test suite, and no linter configured — all logic lives in the two
notebooks under `notebooks/`. Don't introduce a `src/` package, test framework, or CI unless asked;
that would be scope creep for what this repo actually is.

## Commands

```bash
pip install -r requirements.txt   # única fuente de verdad de dependencias (usada también por Colab/Databricks)
```

There is no CLI, build step, lint, or test command — the notebooks *are* the deliverable. To "run" the
project: open Jupyter with cwd set to `notebooks/` (VS Code does this automatically via
`.vscode/settings.json`) and execute, **in this order**:

1. `NB1_Estrategia_Datos_EDA.ipynb` — Run All, start to finish.
2. `NB2_Modelado_Validacion.ipynb` — Run All, start to finish. It hard-depends on the artifacts NB1
   writes to `notebooks/artifacts/` and fails fast with an explicit message if they're missing; it
   cannot run standalone or before NB1.

`RANDOM_STATE = 42` is fixed in both notebooks — don't change it if the goal is to reproduce the
numbers already reported in `docs/`.

Both notebooks use paths relative to `notebooks/` (`dataset/`, `artifacts/`). If asked to run/debug a
cell via a script instead of the Jupyter UI, `cwd` must still be `notebooks/`.

Environment-specific setup (Colab bootstrap cell, Databricks Repos caveat, local venv) is documented
per-environment in README.local.md / README.colab.md / README.databricks.md — check those before
troubleshooting an environment-detection issue instead of guessing.

## Architecture: the NB1 → NB2 artifact contract

The two notebooks communicate **only** through files persisted in `notebooks/artifacts/` — there's no
shared module. Changing a column name, feature list, or file format in NB1 silently breaks NB2 unless
both are updated together. The contract:

| Artifact | Produced by | Consumed by | Purpose |
|---|---|---|---|
| `X_train/val/test.parquet`, `y_train/val/test.parquet` | NB1 §6 | NB2 §1 | Preprocessed model matrices (transformed features + target) |
| `raw_train/val/test.parquet` | NB1 §6 | NB2 §5, §7 | Pre-transformation splits with IDs — used for subgroup error analysis and to attach scores back to readable rows |
| `preprocessor.joblib` | NB1 §5 | NB2 §7 (packaging) | Fitted `ColumnTransformer`, fit on train only |
| `eda_summary.json` | NB1 §6 | NB2 (sanity checks / model card) | EDA decisions, target definition, class balance, feature lists |
| `orders_history.parquet` | NB1 §6 | NB2 §6 | Full per-customer order history (not just first order) — the only input to the RFM segmentation |
| `repeat_purchase_model.joblib`, `model_card.json`, `test_predictions.parquet`, `customer_segments.parquet` | NB2 §7 | `docs/` (manually) | Final deliverables; `docs/` embeds their numbers/figures as static text/images, so regenerating the notebooks desyncs `docs/` until it's manually updated |

## Key modeling decisions already validated (don't re-derive from scratch)

These came out of EDA/experimentation in NB1/NB2 against the real numbers — treat them as settled
unless new evidence contradicts them:

- **Target (`repeat_purchase`)**: 1 if a `customer_unique_id` has ≥2 orders across the *entire*
  dataset; every feature is derived only from that customer's **first** order, to avoid temporal
  leakage. Eligibility filters: first order must be `delivered` and occur before
  `max(order_purchase_timestamp) - 90 days` (censoring cutoff, since customers near the end of the
  window haven't had time to repeat-purchase yet).
- **Severe class imbalance** (~3.3% positive). Primary metric is **PR-AUC (`average_precision`)**, not
  accuracy or plain ROC-AUC. Model comparison and hyperparameter search both optimize
  `average_precision` under 3-fold `StratifiedKFold`.
- **No individual feature is a strong driver** of repeat purchase — all Spearman correlations with the
  target are weak (|ρ| < 0.03), `review_score` included. Don't expect a "smoking gun" feature; frame
  findings accordingly.
- Long-tailed monetary features (`total_price`, `total_freight`) — `total_price` and
  `total_payment_value` are collinear, only `total_price` is kept.
- Product category (71 raw categories) is grouped to top-15 + `other` (`main_category_grouped`) before
  modeling.
- Candidate models compared: `DummyClassifier` (baseline), `LogisticRegression(class_weight="balanced")`,
  `HistGradientBoostingClassifier`. The gradient boosting model wins on PR-AUC and is the one tuned
  (`RandomizedSearchCV`, 6 iterations) and shipped as `repeat_purchase_model.joblib`.
- Business framing of model quality uses **top-decile lift** (repeat-purchase rate among the top-scored
  10% of test customers vs. random contact), not just PR-AUC/ROC-AUC in isolation.
- **RFM segmentation** (NB2 §6) is a *separate, complementary* unsupervised analysis, not derived from
  the classifier — it clusters `recency_days` / `frequency` / `monetary` (log-transformed, scaled) via
  `KMeans(n_clusters=4)` over `orders_history.parquet` (full history, not just first orders). Segment
  labels are assigned from each cluster's actual recency/frequency/monetary profile (not just a
  monetary ranking) and deliberately kept business-neutral (no value-judgment labels like "bad
  customers") per the case's ethical considerations.
- `model_card.json` is the single source of truth for final hyperparameters, decision threshold, and
  test metrics — check it before quoting numbers rather than re-reading the notebook cells.

## Working with the notebooks

- Prefer minimal, surgical edits to a single cell over restructuring — both notebooks follow the
  numbered section structure visible in their markdown headers (§1 environment setup ... §6/§7
  persistence/packaging), and NB2 in particular assumes NB1's artifact schema exactly.
- If you change `FEATURES`, `NUMERIC_FEATURES`, `CATEGORICAL_FEATURES`, or the preprocessing pipeline in
  NB1, both notebooks must be re-run end-to-end (NB1 then NB2) and `docs/` deliverables checked for
  drift, since nothing else auto-syncs them.
- `plantillas/plantilla-notebook.md` and `caso-de-uso/plantilla-notebook.md` define the expected
  academic structure/rubric for each notebook — check them if asked to restructure sections rather than
  inventing a new structure.
- The 9 raw Olist CSVs are already committed under `notebooks/dataset/`; don't assume they need
  downloading. The `kagglehub` fallback in NB1 §1 only triggers if that folder is empty/incomplete.
