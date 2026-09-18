# Implementation Plan: Interpretación de negocio para ROC-AUC y Brier score

**Branch**: `001-informe-metricas-negocio` | **Date**: 2026-09-18 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-informe-metricas-negocio/spec.md`

## Summary

Agregar, en `docs/Informe_Tecnico_ShopMart_Olist.docx` y `docs/Defensa_ShopMart_Olist.pptx`, una
interpretación en lenguaje de negocio para dos métricas del modelo (ROC-AUC de test = 0.6308, Brier
score de test = 0.0317) que hoy solo aparecen como cifras en la tabla de métricas del Informe §3.2, sin
el tratamiento de traducción a negocio que ya recibe PR-AUC. El Informe usa prosa narrativa integrada
(igual que PR-AUC en §3.4); la Presentación usa un callout Q&A explícito por métrica (igual que el
callout ya existente de PR-AUC en la slide 11), citando la cifra exacta en ambos casos. No se modifica
ninguna cifra existente, no se reentrena ni regenera ningún artifact, y no se toca ningún notebook.

## Technical Context

**Language/Version**: N/A — no hay código de aplicación involucrado; el trabajo es edición de texto
dentro de dos documentos binarios ya existentes (`.docx`, `.pptx`).

**Primary Dependencies**: `python-docx` y `python-pptx` como herramienta de edición local de un solo
uso (ver `research.md`) — no se agregan a `requirements.txt`, porque ese archivo es la fuente de
verdad de dependencias del entorno de notebooks/modelado (Colab/Databricks/local), y esta tarea no
toca ese entorno.

**Storage**: Archivos existentes en `docs/`: `Informe_Tecnico_ShopMart_Olist.docx`,
`Defensa_ShopMart_Olist.pptx`. No hay base de datos ni almacenamiento nuevo.

**Testing**: Sin framework de tests (Principio I de la constitución — no aplica a este repo). La
validación es manual/por inspección, siguiendo `quickstart.md` y los criterios de éxito medibles del
spec (presencia de la frase interpretativa, fidelidad numérica contra `model_card.json`).

**Target Platform**: Documentos Word/PowerPoint, consumidos por el tutor/comité de defensa en
Microsoft Office o software compatible (LibreOffice, Google Docs/Slides).

**Project Type**: Actualización de contenido de documentación (no es software; no aplica ninguna de
las estructuras de proyecto del template — single project / web / mobile).

**Performance Goals**: N/A.

**Constraints**:
- No modificar ninguna cifra ya reportada (FR-004).
- No regenerar ni reentrenar ningún artifact bajo `notebooks/artifacts/`, ni tocar los notebooks
  (FR-005; Principio IV de la constitución).
- No crear secciones o slides nuevas — el texto se inserta dentro de la estructura existente (SC-005).
- Preservar el formato/estilo original de ambos documentos (tipografía, tablas, plantilla de la
  maestría) — cualquier edición debe ser indistinguible en estilo del resto del documento.

**Scale/Scope**: 4 inserciones de texto acotadas (ROC-AUC × 2 documentos, Brier score × 2 documentos),
sobre 2 archivos ya existentes. Sin cambios de alcance más allá de eso.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio / Sección | Evaluación |
|---|---|
| I. Academic Scope Discipline | **PASS** — no se introduce `src/`, framework de tests, linter ni CI. `python-docx`/`python-pptx` se usan como herramienta de edición local, sin agregarse a `requirements.txt` ni al entorno reproducible del proyecto. |
| II. Reproducibility & Fixed Seed | **N/A** — no se toca ningún notebook ni `RANDOM_STATE`; no aplica re-ejecución de NB1/NB2. |
| III. NB1→NB2 Artifact Contract Integrity | **N/A** — no se modifica ningún artifact ni el contrato entre notebooks. |
| IV. Validated Modeling Decisions Are Settled | **PASS** — el feature no reabre ninguna decisión de modelado; solo añade narrativa alrededor de cifras ya asentadas en `model_card.json` (FR-004, FR-005). |
| V. Surgical, Traceable Notebook Edits | **N/A** (no hay edición de notebooks) — el mismo espíritu quirúrgico se aplica por analogía a los documentos: inserciones puntuales dentro de la estructura existente, sin restructurar (SC-005). |
| Dataset, Dependencies & Environment Handling | **PASS** — sin cambios a `requirements.txt` ni al dataset. |
| Documentation & Deliverables Sync | **PASS** — este feature es, en sí mismo, la corrección de un gap de sincronización narrativa señalado en una revisión previa; no hay resultados de ejecución nuevos que sincronizar. |
| Governance (consistencia constitution ↔ CLAUDE.md) | **PASS** — no se amenda la constitución ni `CLAUDE.md` como parte de este feature. |

No hay violaciones. **Complexity Tracking: no aplica** (sin justificaciones pendientes).

## Project Structure

### Documentation (this feature)

```text
specs/001-informe-metricas-negocio/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan) — modelo de contenido, no de datos de software
├── quickstart.md         # Fase 1 (/speckit-plan) — guía de validación manual
└── tasks.md              # Fase 2 (/speckit-tasks) — no generado por este comando
```

No se genera `contracts/`: el feature no expone ninguna interfaz externa (API, CLI, esquema) — es
edición de contenido en dos documentos internos del proyecto.

### Source Code (repository root)

No aplica ninguna de las estructuras de código fuente del template (`src/`, `backend/`, `frontend/`,
etc.) — este repo, por el Principio I de la constitución, no tiene código de aplicación. Los únicos
archivos que este feature toca son:

```text
docs/
├── Informe_Tecnico_ShopMart_Olist.docx   # §3.2 (tabla de métricas) y §3.4 (narrativa) editados
└── Defensa_ShopMart_Olist.pptx           # slide 11 (o adyacente) editada
```

**Structure Decision**: sin estructura de código nueva. El trabajo se realiza directamente sobre los
dos archivos de `docs/` listados arriba, usando el enfoque de edición descrito en `research.md`.
