# Implementation Plan: Corregir split train/val/test no temporal en NB1

**Branch**: `003-corregir-split-temporal` | **Date**: 2026-09-18 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/003-corregir-split-temporal/spec.md`

## Summary

En NB1 §6, `model_df` se ordena por `order_purchase_timestamp` pero las dos llamadas a
`train_test_split` no pasan `shuffle=False`, así que sklearn usa su `shuffle=True` por defecto y hace
un `StratifiedShuffleSplit` que baraja las filas — el orden temporal se pierde por completo (confirmado
empíricamente: train/val/test cubren hoy el mismo rango de fechas 2016-09/10↔2018-07-19). Como
`stratify` y `shuffle=False` no pueden combinarse en `train_test_split`, la corrección reemplaza esas
dos llamadas por una función propia de partición **temporal por clase**: para cada valor de
`repeat_purchase` por separado, ordena por `(order_purchase_timestamp, order_id)` (tiebreaker
determinístico — hay 209 timestamps duplicados) y toma el primer 70%/siguiente 15%/último 15% de esa
clase. Esto preserva simultáneamente el orden temporal global (validado empíricamente: p90 de
train ≤ p10 de test) y el desbalance de clases documentado (~3.3%, prácticamente idéntico al actual:
3.30%/3.30%/3.31%), sin ningún componente aleatorio — más fuerte en reproducibilidad que el mecanismo
anterior. NB1 se re-ejecuta completo (Run All), lo que regenera todos sus artifacts; NB2 se re-ejecuta
completo a continuación porque consume esos artifacts (Principio II). Cualquier métrica, hiperparámetro
ganador, umbral o hallazgo de NB2 que cambie de valor se sincroniza después en
`docs/Informe_Tecnico_ShopMart_Olist.docx` (+PDF) y `docs/Defensa_ShopMart_Olist.pptx`.

## Technical Context

**Language/Version**: Python (mismo entorno de `requirements.txt` que ya usan los notebooks — pandas,
scikit-learn). No se agrega ninguna dependencia nueva.

**Primary Dependencies**: `pandas` (`groupby`/`sort_values`/`iloc` para la partición temporal por
clase — reemplaza `sklearn.model_selection.train_test_split` en esa celda puntual; el resto de NB1/NB2
sigue usando scikit-learn sin cambios). Para editar `docs/*.docx`/`.pptx` se reutiliza
`python-docx`/`python-pptx` como herramienta local efímera (ya usada en `specs/001-...` y
`specs/002-...`, no se agrega a `requirements.txt`).

**Storage**: Archivos existentes: `notebooks/NB1_Estrategia_Datos_EDA.ipynb` (única celda de código +
su celda markdown, §6), `notebooks/NB2_Modelado_Validacion.ipynb` (solo re-ejecución; posible ajuste de
una celda markdown si el resultado de la comparación de candidatos cambia, ver Constraints), todos los
artifacts de `notebooks/artifacts/` (regenerados en cascada), `docs/Informe_Tecnico_ShopMart_Olist.docx`,
`docs/Defensa_ShopMart_Olist.pptx`. Sin almacenamiento nuevo.

**Testing**: Sin framework de tests (Principio I). Validación manual/por inspección vía
`quickstart.md`: comparar percentiles de timestamp entre splits, tasa de recompra por split, y diff de
`model_card.json` antes/después para confirmar qué cambió y por qué.

**Target Platform**: Ejecución de notebook (local/Colab/Databricks); documentos Word/PowerPoint
consumidos por el tutor/comité de defensa.

**Project Type**: Corrección puntual de código de notebook + re-ejecución en cascada + actualización de
documentación — no aplica ninguna estructura de proyecto de software (`src/`, `backend/`, etc.),
consistente con el Principio I.

**Performance Goals**: N/A — particionar 84,282 filas en dos grupos (`groupby` por clase, cada uno con
un `sort_values` y tres `iloc` slices) es trivial frente al resto de NB1/NB2 (perfilado de 9 tablas,
`RandomizedSearchCV`, permutation importance).

**Constraints**:
- `RANDOM_STATE=42` se mantiene fijo en todo el resto del pipeline (Principio II); la partición en sí
  ya no depende de aleatoriedad (ver Summary) — no es una relajación del principio, es una garantía más
  fuerte de reproducibilidad para ese paso específico.
- Ningún otro elemento ya validado cambia: target, censura de 90 días, `FEATURES`/
  `NUMERIC_FEATURES`/`CATEGORICAL_FEATURES`, pipeline de preprocesamiento (FR-005; Principio IV).
- El cambio de código se limita a la celda de partición de NB1 §6 y su celda markdown adyacente
  (Principio V — edición quirúrgica, sin restructurar).
- NB1 DEBE ejecutarse completo (Run All) y luego NB2 completo (Run All), en ese orden (Principio II) —
  a diferencia de `specs/002-...`, aquí SÍ se re-ejecuta NB1 porque el cambio está en su artifact
  contract (qué filas caen en cada split), no solo en NB2.
- `customer_segments.parquet` (RFM) NO debe cambiar — se calcula sobre `orders_history.parquet`
  (historial completo), independiente de esta partición (FR-007).
- Todo hallazgo de NB2 con un número hardcodeado en prosa (markdown) que dependa de la partición DEBE
  revisarse tras el Run All — se identificó un caso concreto: NB2 celda markdown §3 ("Resultado de la
  comparación... 0.0468 vs. 0.0455") cita las cifras de CV textualmente; si cambian, esa celda markdown
  también debe editarse, no solo `docs/`. Los hallazgos de NB1 que mencionan cifras (nulos estructurales,
  rango de tasa de recompra por `review_score`) NO dependen del split — ocurren antes de la partición
  sobre las 84,282 filas elegibles completas — y no requieren revisión.
- Edición de `docs/`: Informe §3.1/§3.2/§3.3/§3.4 (y §5.1 si el lift cambia lo suficiente para alterar
  la redacción de las conclusiones) y Presentación slides 5, 10, 11, 12, 13, 14, 15 (comparación de
  candidatos, diseño experimental, métricas finales, impacto de negocio, importancia de variables) —
  la ubicación exacta de cada slide se re-verifica por contenido antes de editar, no por el número de
  slide citado en esta conversación (lección heredada de `specs/001-.../research.md` y
  `specs/002-.../research.md`). Slides 6–9 (descripción de datos, EDA, mito del review score) y 17–20
  (RFM, ética, privacidad) NO dependen de la partición y no deberían tocarse salvo que la re-ejecución
  demuestre lo contrario.

**Scale/Scope**: 1 celda de código + 1 celda markdown reemplazadas en NB1 §6; posible ajuste de 1 celda
markdown en NB2 §3 (cifras de CV) según el resultado real de la re-ejecución; Run All completo de NB1 y
NB2; ediciones de texto acotadas en hasta 4 secciones del Informe y hasta 7 slides de la Presentación,
solo donde el número/hallazgo realmente cambió.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio / Sección | Evaluación |
|---|---|
| I. Academic Scope Discipline | **PASS** — sin `src/`, tests, linter ni CI nuevos; sin dependencias nuevas (pandas ya está en uso). |
| II. Reproducibility & Fixed Seed | **PASS** — `RANDOM_STATE=42` se mantiene en el resto del pipeline; la partición corregida es determinística (sin componente aleatorio) en vez de depender del seed, lo cual es una garantía de reproducibilidad estrictamente igual o más fuerte. NB1 se ejecuta completo antes que NB2 (Run All en ambos), exactamente el orden que exige el principio. |
| III. NB1→NB2 Artifact Contract Integrity | **PASS** — el cambio no altera ningún nombre de columna, feature ni formato de archivo del contrato documentado en `CLAUDE.md`; solo cambia qué filas caen en cada partición. Ambos notebooks se re-ejecutan end-to-end tras el cambio, como exige el principio ante cualquier edición que afecte los artifacts. |
| IV. Validated Modeling Decisions Are Settled | **PASS, con nota** — el principio protege decisiones ya validadas "absent new evidence"; aquí SÍ hay evidencia nueva (un bug de partición confirmado empíricamente, no una preferencia estética), lo cual es precisamente la excepción que el principio contempla. Si el resultado de la comparación de candidatos u otra cifra cambia bajo la partición corregida, el cambio se documenta explícitamente (FR-010) en vez de ocultarse — no es una re-derivación silenciosa, es la corrección de un defecto que invalidaba la evidencia anterior. |
| V. Surgical, Traceable Notebook Edits | **PASS** — la edición se limita a la celda de partición de NB1 §6 (código + markdown) y, si aplica, a una celda markdown puntual de NB2 §3; sin restructurar ninguna sección numerada. |
| Dataset, Dependencies & Environment Handling | **PASS** — sin cambios a `requirements.txt`; sin datos nuevos; rutas relativas a `notebooks/` sin cambios. |
| Documentation & Deliverables Sync | **PASS** — este feature es la aplicación directa de esa sección (v1.1.0): como el cambio altera resultados de ejecución aguas abajo, el Informe y la Presentación se sincronizan con los valores reales tras el Run All, en el mismo trabajo, no como una tarea diferida. |
| Governance | **PASS** — no se amenda la constitución ni `CLAUDE.md`; esta corrección no introduce ninguna regla nueva, solo corrige una implementación que contradecía una narrativa ya documentada. |

No hay violaciones. **Complexity Tracking: no aplica**.

## Project Structure

### Documentation (this feature)

```text
specs/003-corregir-split-temporal/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan) — entidades de la partición y del contenido a sincronizar
├── quickstart.md         # Fase 1 (/speckit-plan) — guía de validación manual
└── tasks.md              # Fase 2 (/speckit-tasks) — no generado por este comando
```

No se genera `contracts/`: el feature no expone ninguna interfaz externa — es una corrección de código
de notebook + re-ejecución en cascada + documentación interna del proyecto.

### Source Code (repository root)

No aplica ninguna estructura de código fuente del template (`src/`, `backend/`, etc.), por el
Principio I. Los archivos que este feature toca son:

```text
notebooks/
├── NB1_Estrategia_Datos_EDA.ipynb       # §6 (celda de partición + su markdown, reemplazadas)
├── NB2_Modelado_Validacion.ipynb        # Run All completo; posible ajuste puntual en §3 (markdown
│                                          # con cifras de CV) si el resultado real cambia
└── artifacts/                            # TODOS regenerados: X/y_train/val/test.parquet,
                                           # raw_train/val/test.parquet, preprocessor.joblib (por NB1);
                                           # repeat_purchase_model.joblib, model_card.json,
                                           # test_predictions.parquet (por NB2).
                                           # customer_segments.parquet NO debería cambiar (FR-007).

docs/
├── Informe_Tecnico_ShopMart_Olist.docx   # §3.1/§3.2/§3.3/§3.4 (y §5.1 si aplica) editadas
└── Defensa_ShopMart_Olist.pptx           # subconjunto de slides 5/10-15 editadas (ver Constraints)
```

**Structure Decision**: sin estructura de código nueva. El trabajo se realiza sobre
`NB1_Estrategia_Datos_EDA.ipynb` (la celda de partición), se propaga por Run All a
`NB2_Modelado_Validacion.ipynb`, y de ahí a los dos archivos de `docs/` listados arriba — mismo enfoque
de edición ya usado en `specs/001-informe-metricas-negocio` y `specs/002-corregir-discrepancias-modelo`.
