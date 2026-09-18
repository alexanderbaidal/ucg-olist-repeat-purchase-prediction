# Implementation Plan: Corregir discrepancias Informe/Presentación vs. código real

**Branch**: `002-corregir-discrepancias-modelo` | **Date**: 2026-09-18 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-corregir-discrepancias-modelo/spec.md`

## Summary

Cerrar dos discrepancias detectadas entre `docs/Informe_Tecnico_ShopMart_Olist.docx` /
`docs/Defensa_ShopMart_Olist.pptx` y el código real de `notebooks/NB2_Modelado_Validacion.ipynb`:
(1) agregar en NB2 §5.2 un análisis de subgrupo real por `customer_state` (hoy inexistente, aunque el
Informe y la Presentación afirman que "no hay disparidades por estado"), y (2) persistir en
`model_card.json` el número real de rondas de boosting entrenadas por el modelo final (hoy los
documentos citan `max_iter=150`, el techo del espacio de búsqueda, cuando el modelo real —con
`early_stopping="auto"` activo— entrenó 62 rondas). NB2 se re-ejecuta completo (Run All) para
regenerar los artifacts; NB1 no se toca. Ninguna métrica de desempeño ya validada cambia — el cambio
es aditivo. Los dos documentos se actualizan después para citar los valores reales.

## Technical Context

**Language/Version**: Python (el mismo entorno de `requirements.txt` que ya usan los notebooks —
pandas, scikit-learn). No se agrega ninguna dependencia nueva al entorno de modelado.

**Primary Dependencies**: `pandas`/`scikit-learn` (ya en uso en NB2 para el análisis de subgrupo
existente y el modelo). Para editar `docs/*.docx`/`.pptx` se reutiliza `python-docx`/`python-pptx`
como herramienta local efímera (ya instalada en la sesión durante `specs/001-informe-metricas-negocio`
— ver su `research.md` §1 — no se agrega a `requirements.txt`).

**Storage**: Archivos existentes: `notebooks/NB2_Modelado_Validacion.ipynb`,
`notebooks/artifacts/model_card.json` (y el resto de artifacts que NB2 regenera al correr Run All),
`docs/Informe_Tecnico_ShopMart_Olist.docx`, `docs/Defensa_ShopMart_Olist.pptx`. Sin almacenamiento
nuevo.

**Testing**: Sin framework de tests (Principio I de la constitución). Validación manual/por
inspección, siguiendo `quickstart.md` y los criterios de éxito medibles del spec (comparación de
`model_card.json` antes/después, presencia de las cifras corregidas en ambos documentos).

**Target Platform**: Ejecución de notebook (local/Colab/Databricks, según README.*.md del repo);
documentos Word/PowerPoint consumidos por el tutor/comité de defensa.

**Project Type**: Corrección puntual de código de notebook + actualización de documentación — no
aplica ninguna estructura de proyecto de software (`src/`, `backend/`, etc.), consistente con el
Principio I.

**Performance Goals**: N/A — el `groupby` adicional sobre 27 estados y ~12,643 filas de test es
computacionalmente trivial frente al resto de NB2 (búsqueda de hiperparámetros, permutation
importance).

**Constraints**:
- `RANDOM_STATE=42` NO se modifica (FR-002).
- Ninguna decisión de modelado ya validada cambia: target, features, splits, espacio de búsqueda de
  hiperparámetros, umbral de decisión (FR-002, FR-008; Principio IV de la constitución).
- El cambio en NB2 es aditivo: una celda de análisis nueva en §5.2 + un campo nuevo en el diccionario
  `model_card` de §7 (FR-001, FR-006, FR-008).
- NB2 DEBE re-ejecutarse completo (Run All) tras el cambio; NB1 NO requiere re-ejecutarse (FR-003).
- Las ediciones en `docs/` se insertan dentro de las secciones/slides ya existentes (§3.4, §3.5, §4.1
  del Informe; slides de "Resultados: Qué Aprendió el Modelo"/"Errores y Casos Atípicos" (idx=15,
  desempeño por subgrupo), "Prototipo y Despliegue" (idx=17, cifra de iteraciones) y "Ética: Sesgos y
  Equidad" (idx=18) de la Presentación — sin crear slides nuevas (SC-004). **Nota heredada de
  `specs/001-informe-metricas-negocio`**: el orden de archivo (`slideN.xml`) no coincide con el orden
  de presentación real (`sldIdLst`) para varias slides — la ubicación exacta de cada slide DEBE
  verificarse por contenido (búsqueda de texto), no asumirse por nombre de archivo, igual que se hizo
  en la feature anterior.

**Scale/Scope**: 1 celda nueva (o extensión de la existente) en NB2 §5.2; 1 campo nuevo en el
diccionario `model_card` de NB2 §7; ediciones de texto acotadas en 3 secciones del Informe y 3 slides
de la Presentación.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio / Sección | Evaluación |
|---|---|
| I. Academic Scope Discipline | **PASS** — sin `src/`, tests, linter ni CI nuevos. `python-docx`/`python-pptx` se reutilizan como herramienta local ya instalada, sin tocar `requirements.txt`. |
| II. Reproducibility & Fixed Seed | **PASS** — `RANDOM_STATE=42` sin cambios; NB2 se re-ejecuta completo (Run All) tras el cambio, como exige el principio ante cualquier edición de notebook. |
| III. NB1→NB2 Artifact Contract Integrity | **PASS** — el cambio es aditivo sobre el lado de salida de NB2 (`model_card.json` gana un campo nuevo, no se renombra ni elimina ninguno existente); el lado consumidor real de ese campo (`docs/`) se actualiza en el mismo cambio (FR-007), cumpliendo la letra del principio. |
| IV. Validated Modeling Decisions Are Settled | **PASS** — el feature explícitamente NO reabre ninguna decisión de modelado; FR-002/FR-008 lo hacen una restricción dura. Es exactamente el tipo de auditoría/diagnóstico que el principio permite sin violar el "no re-derivar". |
| V. Surgical, Traceable Notebook Edits | **PASS** — una celda de análisis nueva en §5.2 y un campo nuevo en §7, siguiendo la estructura numerada existente, sin restructurar NB2. |
| Dataset, Dependencies & Environment Handling | **PASS** — sin cambios a `requirements.txt`; sin datos nuevos. |
| Documentation & Deliverables Sync | **PASS** — este feature es la aplicación directa de esa sección (v1.1.0): un cambio en los resultados/artifacts de NB2 (nuevo campo, nuevo hallazgo de subgrupo) se sincroniza con `docs/` en el mismo cambio, no se deja pendiente. |
| Governance | **PASS** — no se amenda la constitución ni `CLAUDE.md`. |

No hay violaciones. **Complexity Tracking: no aplica**.

## Project Structure

### Documentation (this feature)

```text
specs/002-corregir-discrepancias-modelo/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan) — entidades del análisis de subgrupo y del model card
├── quickstart.md         # Fase 1 (/speckit-plan) — guía de validación manual
└── tasks.md              # Fase 2 (/speckit-tasks) — no generado por este comando
```

No se genera `contracts/`: el feature no expone ninguna interfaz externa — es una corrección de código
de notebook + documentación interna del proyecto.

### Source Code (repository root)

No aplica ninguna estructura de código fuente del template (`src/`, `backend/`, etc.), por el
Principio I. Los archivos que este feature toca son:

```text
notebooks/
├── NB2_Modelado_Validacion.ipynb     # §5.2 (nueva celda de subgrupo por customer_state)
│                                      # §7 (nuevo campo en model_card)
└── artifacts/                        # regenerados por NB2 Run All (model_card.json, etc.)

docs/
├── Informe_Tecnico_ShopMart_Olist.docx   # §3.4, §3.5, §4.1 editadas
└── Defensa_ShopMart_Olist.pptx           # 3 slides editadas (ver Technical Context > Constraints)
```

**Structure Decision**: sin estructura de código nueva. El trabajo se realiza directamente sobre
`NB2_Modelado_Validacion.ipynb` (una celda + un campo) y luego sobre los dos archivos de `docs/`
listados arriba, reutilizando el enfoque de edición de `specs/001-informe-metricas-negocio/research.md`.
