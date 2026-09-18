<!--
Sync Impact Report
==================
Version change: [TEMPLATE] → 1.0.0 (initial ratification — no prior filled constitution existed)
Bump rationale: MAJOR because this is the first concrete version establishing binding
  principles (semver starts counting from a defined baseline, not from the unfilled template).

Modified principles: none (first fill; all [PRINCIPLE_N_*] placeholders replaced)

Added sections:
  - I. Academic Scope Discipline (NON-NEGOTIABLE)
  - II. Reproducibility & Fixed Seed
  - III. NB1→NB2 Artifact Contract Integrity
  - IV. Validated Modeling Decisions Are Settled
  - V. Surgical, Traceable Notebook Edits
  - Dataset, Dependencies & Environment Handling (Section 2)
  - Documentation & Deliverables Sync (Section 3)
  - Governance

Removed sections: none

Templates requiring updates:
  - .specify/templates/plan-template.md — ✅ no change needed (Constitution Check gate already
    reads dynamically from this file; generic src/tests options are marked [REMOVE IF UNUSED])
  - .specify/templates/spec-template.md — ✅ no change needed (technology-agnostic, no conflict)
  - .specify/templates/tasks-template.md — ✅ no change needed (tests explicitly marked OPTIONAL,
    consistent with Principle I; src/tests paths are generic defaults adjusted per plan.md)
  - .specify/templates/commands/*.md — ⚠ directory not present in this repo; nothing to update
  - CLAUDE.md — ✅ no change needed (this constitution formalizes rules already stated there)
  - README.md / README.local.md / README.colab.md / README.databricks.md — ✅ no change needed
    (environment-specific instructions already match Section 2 below)

Follow-up TODOs: none — no placeholders were deferred.
-->

# ucg-olist-repeat-purchase-prediction Constitution

## Core Principles

### I. Academic Scope Discipline (NON-NEGOTIABLE)

This repository is a Trabajo de Titulación deliverable (Maestría en Data Science e IA, UCG), not a
production system. The two notebooks under `notebooks/` plus the report/presentation in `docs/` ARE
the deliverable. The project MUST NOT introduce a `src/` package, a test framework, a linter, or a
CI/CD pipeline unless the user explicitly asks for it in that conversation. Any proposal to
restructure the project MUST be justified against the case brief
(`caso-de-uso/data-driven-e-commerce.md`) and the notebook rubric
(`plantillas/plantilla-notebook.md`, `caso-de-uso/plantilla-notebook.md`) before being adopted.

**Rationale**: scope creep dilutes an academic deliverable, adds maintenance surface with no grading
value, and contradicts how this repo has been built and reviewed to date.

### II. Reproducibility & Fixed Seed

`RANDOM_STATE = 42` MUST remain fixed in both notebooks whenever the goal is to reproduce numbers
already reported in `docs/`. The notebooks MUST be executed in strict order — `NB1_Estrategia_Datos_EDA.ipynb`
(Run All, start to finish) before `NB2_Modelado_Validacion.ipynb` (Run All, start to finish) — because
NB2 hard-depends on the artifacts NB1 writes to `notebooks/artifacts/` and fails fast with an explicit
error if they are missing. NB2 MUST NOT be treated as runnable standalone or before NB1.

**Rationale**: the reported metrics, figures, and business conclusions in `docs/` are tied to this
exact seed and execution order; breaking either silently invalidates the deliverable's numbers.

### III. NB1→NB2 Artifact Contract Integrity

The two notebooks communicate ONLY through files persisted in `notebooks/artifacts/` — there is no
shared module between them. Any change to a column name, feature list, or file format on either side
of the artifact contract (as documented in `CLAUDE.md`'s artifact table: `X_*`/`y_*` matrices,
`raw_*` splits, `preprocessor.joblib`, `eda_summary.json`, `orders_history.parquet`,
`repeat_purchase_model.joblib`, `model_card.json`, `test_predictions.parquet`,
`customer_segments.parquet`) MUST be mirrored on the consuming side in the same change, and both
notebooks MUST be re-run end-to-end afterward to confirm the contract still holds.

**Rationale**: because there is no shared module or schema validation, an unmirrored change fails
silently in NB2 (or worse, runs with stale semantics) rather than raising an explicit error.

### IV. Validated Modeling Decisions Are Settled

Modeling decisions already validated against real EDA/experimentation results MUST NOT be re-derived
from scratch or silently changed absent new evidence. This includes: the `repeat_purchase` target
definition (≥2 orders across the full dataset, features derived only from the first order, with the
90-day censoring cutoff to avoid leakage); PR-AUC (`average_precision`) as the primary metric under
~3.3% class imbalance; the finding that no individual feature is a strong driver (all |Spearman ρ| <
0.03); the `total_price`-over-`total_payment_value` collinearity resolution; the top-15+`other`
category grouping; the `DummyClassifier` / `LogisticRegression` / `HistGradientBoostingClassifier`
comparison and the selection of the tuned gradient boosting model; the top-decile lift business
framing; and RFM segmentation as a separate, complementary, business-neutral unsupervised analysis
over full order history (not derived from the classifier). `model_card.json` is the single source of
truth for final hyperparameters, decision threshold, and test metrics — it MUST be checked before
quoting numbers, rather than re-reading notebook cells or re-running experiments to re-confirm
settled results.

**Rationale**: these conclusions came from real experimentation against the actual dataset; treating
them as open questions each session wastes effort and risks introducing numeric drift against the
already-written `docs/` deliverables.

### V. Surgical, Traceable Notebook Edits

Edits to either notebook MUST be minimal and surgical — targeted at a single cell or numbered section
— rather than restructuring the notebook. Edits MUST follow the existing numbered section structure
(§1 environment setup … §6/§7 persistence/packaging) and the academic structure/rubric defined in
`plantillas/plantilla-notebook.md` / `caso-de-uso/plantilla-notebook.md`, rather than inventing a new
structure. Whenever `FEATURES`, `NUMERIC_FEATURES`, `CATEGORICAL_FEATURES`, or the preprocessing
pipeline changes in NB1, both notebooks MUST be re-run end-to-end (NB1 then NB2) and every `docs/`
deliverable MUST be checked for drift before the change is considered complete.

**Rationale**: NB2 assumes NB1's exact artifact schema; large restructures multiply the surface area
that can silently desynchronize the two notebooks or the `docs/` write-up.

## Dataset, Dependencies & Environment Handling

The 9 raw Olist CSVs are already committed under `notebooks/dataset/`; downloading them MUST NOT be
assumed necessary. The `kagglehub` fallback in NB1 §1 only triggers when that folder is empty or
incomplete. `requirements.txt` is the single source of truth for dependencies and is used identically
across local, Colab, and Databricks environments — dependency changes MUST be made there, not
duplicated elsewhere. Both notebooks use paths relative to `notebooks/` (`dataset/`, `artifacts/`);
any script-based execution of a cell (as opposed to the Jupyter UI) MUST still set `cwd` to
`notebooks/`. Environment-specific setup (Colab bootstrap cell, Databricks Repos caveats, local venv)
is documented per environment in `README.local.md` / `README.colab.md` / `README.databricks.md` —
these MUST be consulted before treating an environment-detection issue as a bug to guess at.

## Documentation & Deliverables Sync

`docs/` embeds the notebooks' final numbers and figures as static text/images (report, presentation,
model card narrative). Regenerating the notebooks — even without changing their logic, if the
underlying artifacts change — desyncs `docs/` until it is manually updated; nothing in this repo
auto-syncs them. Any change that alters NB1's or NB2's outputs (metrics, thresholds, figures, segment
profiles) MUST be flagged to the user as requiring a manual `docs/` update, and MUST NOT be presented
as complete until that sync is either done or explicitly deferred by the user.

## Governance

This constitution codifies rules already operationalized day-to-day in `CLAUDE.md`; the two MUST be
kept consistent — an amendment here that changes a binding rule MUST be reflected in `CLAUDE.md` in
the same change, and vice versa when `CLAUDE.md` gains a new non-negotiable rule.

**Amendment procedure**: propose the change, identify affected principles/sections, update this file
via `/speckit-constitution`, and propagate the change to `CLAUDE.md` and any dependent
`.specify/templates/*` files in the same session.

**Versioning policy** (semantic versioning for governance):
- MAJOR: backward-incompatible principle removal or redefinition (e.g., reversing a settled modeling
  decision in Principle IV, or lifting the no-`src/`/no-CI rule in Principle I).
- MINOR: a new principle or section added, or materially expanded guidance on an existing one.
- PATCH: wording clarifications, typo fixes, non-semantic refinements.

**Compliance review**: because this project has no CI or automated linting by design (Principle I),
compliance is reviewed manually — by whoever is editing the notebooks or `docs/` in a given session —
against this file and `CLAUDE.md` before a change is considered done, not enforced by a pipeline gate.

**Version**: 1.0.0 | **Ratified**: 2026-09-18 | **Last Amended**: 2026-09-18
