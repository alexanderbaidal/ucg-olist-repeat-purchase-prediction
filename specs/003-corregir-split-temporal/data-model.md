# Content/Data Model: Corregir split train/val/test no temporal en NB1

Este feature combina un cambio de código (NB1, con re-ejecución en cascada de NB2) con actualizaciones
de contenido (Informe/Presentación). Describe la entidad de partición corregida, los artifacts que
dependen de ella, y las unidades de contenido en `docs/` que deben sincronizarse con el resultado real.

## Entidad: Partición temporal por clase (train/val/test)

Producida por la función `temporal_split_by_class` (NB1 §6, ver `research.md` §2), reemplaza las dos
llamadas actuales a `train_test_split`. Una fila de `model_df` (un cliente elegible, un registro por
primer pedido) pertenece a exactamente una de las tres particiones.

| Campo | Tipo | Descripción |
|---|---|---|
| `order_purchase_timestamp` | datetime | Clave de ordenamiento primaria dentro de cada clase |
| `order_id` | string | Clave de ordenamiento secundaria (tiebreaker determinístico ante timestamps duplicados) |
| `repeat_purchase` | 0/1 | Clase dentro de la cual se aplica el corte 70/15/15 por separado |

**Reglas de validación**:
- La unión de train/val/test DEBE cubrir exactamente las 84,282 filas elegibles de `model_df`, sin
  duplicados ni omisiones (misma invariante que el mecanismo anterior).
- Para cada clase (`repeat_purchase` ∈ {0, 1}), el conteo de filas en train + val + test DEBE ser igual
  al conteo total de esa clase en `model_df`.
- El percentil 90 de `order_purchase_timestamp` en train DEBE ser ≤ al percentil 10 en test (FR-001,
  SC-001).
- La tasa de recompra (`repeat_purchase.mean()`) de cada split DEBE estar dentro de ±0.5 puntos
  porcentuales del ~3.3% documentado en `eda_summary.json` (FR-002, SC-002).

## Artifacts que dependen de la partición corregida

| Artifact | Producido por | Cambia con este feature? |
|---|---|---|
| `X_train/val/test.parquet`, `y_train/val/test.parquet` | NB1 §6 | Sí — mismas columnas/schema, filas distintas |
| `raw_train/val/test.parquet` | NB1 §6 | Sí — mismas columnas/schema, filas distintas |
| `preprocessor.joblib` | NB1 §5 (fit sobre `X_train`) | Sí — mismos pasos de transformación, estadísticos (medianas, medias/desvíos del `StandardScaler`) recalculados sobre el nuevo `X_train` |
| `eda_summary.json` | NB1 §6 (`splits`) | Sí — el bloque `splits` (n y rate por partición) se regenera; el resto del archivo (hallazgos de EDA sobre la tabla completa) no cambia |
| `orders_history.parquet` | NB1 §6 | **No** — se construye sobre el historial completo, no sobre `model_df` particionado |
| `repeat_purchase_model.joblib`, `model_card.json`, `test_predictions.parquet` | NB2 §7 | Sí — modelo reentrenado, métricas y umbral recalculados sobre la nueva partición |
| `customer_segments.parquet` | NB2 §6 (RFM) | **No** — depende de `orders_history.parquet`, no de `X/y_train/val/test` (FR-007) |

## Unidades de contenido a revisar/actualizar en NB2 y en `docs/`

Cada unidad de contenido se clasifica según `research.md` §4 (depende o no del split), y se actualiza
solo si el valor real tras la re-ejecución difiere del actual.

| Campo | Descripción |
|---|---|
| `ubicacion` | Celda markdown de notebook, o sección/slide de `docs/` |
| `depende_del_split` | Sí/No, según el inventario de `research.md` §4 |
| `fuente_de_verdad` | `model_card.json` regenerado (para todo lo que depende del split) |
| `accion` | `revisar_y_actualizar_si_cambio` o `no_tocar` |

### Instancias con `depende_del_split = Sí`

| # | Ubicación | Contenido actual citado | Fuente de verdad tras Run All |
|---|---|---|---|
| 1 | NB1 §6, celda markdown de partición | "se ordena por `order_purchase_timestamp`... para que el test contenga a los clientes más recientes" (mecanismo, no cifra) | Reescritura directa como parte de este feature (US3), no depende de re-ejecutar |
| 2 | NB2 §3, celda markdown ("Resultado de la comparación") | PR-AUC CV: 0.0468 (logística) vs. 0.0455 (ensemble) | `comparison_df` recalculado tras Run All |
| 3 | Informe §3.1/§3.3 | Empate técnico CV (0.0468/0.0455/0.0330) | `comparison_df` recalculado |
| 4 | Informe §3.3 (tabla), §3.4 | PR-AUC val/test, ROC-AUC, Brier, lift | `model_card.json.metrics` |
| 5 | Informe §3.2 | Hiperparámetros ganadores, umbral calibrado | `model_card.json.hyperparameters`, `decision_threshold` |
| 6 | Informe §3.4 (Figura 6) | Top features por permutation importance | `model_card.json.top_features_by_permutation_importance` |
| 7 | Informe §5.1 | Cifra de lift citada en conclusiones (2.27x) | `model_card.json.metrics.test.top_decile_lift` |
| 8 | Presentación slide 5 | Baseline PR-AUC ≈ 0.033 | `comparison_df` (fila `baseline_dummy`) |
| 9 | Presentación slide 10 | Baseline PR-AUC = 0.033 | ídem |
| 10 | Presentación slide 11 (+ nota de orador) | CV logística 0.047 vs. HGB 0.046; PR-AUC test = 0.059 | `comparison_df` + `model_card.json` |
| 11 | Presentación slide 12 (+ nota de orador) | Mecanismo de split, hiperparámetros de búsqueda, umbral | Reescritura directa (mecanismo) + `model_card.json` (hiperparámetros) |
| 12 | Presentación slide 13 (+ nota de orador) | PR-AUC 0.059, ROC-AUC 0.63, lift 2.27x, Brier 0.0317 | `model_card.json.metrics` |
| 13 | Presentación slide 14 (+ nota de orador) | Lift 2.27x, "~7.5 de cada 100" | `model_card.json.metrics.test.top_decile_lift` |
| 14 | Presentación slide 15 (+ nota de orador) | Top features (cuotas, precio, flete, categorías) | `model_card.json.top_features_by_permutation_importance` |

### Instancias con `depende_del_split = No` (no tocar salvo evidencia en contra)

NB1 hallazgos de nulos estructurales y de tasa de recompra por `review_score`; Informe §2.1/§2.3;
Presentación slides 6–9 (datos/EDA/review score) y 17–20 (RFM, ética/privacidad, anexos) —
ver inventario completo en `research.md` §4.

## Reglas de consistencia

- Las instancias 2 y 3 (comparación de candidatos) DEBEN citar el mismo par de cifras — si NB2 cambia,
  el Informe se actualiza con el mismo valor, no una aproximación distinta.
- Las instancias 4, 8, 9, 10, 12, 13 (métricas/baseline) DEBEN citar exactamente `model_card.json`
  regenerado — ninguna cifra de la ejecución anterior a este feature debe sobrevivir en ningún
  documento.
- Si la comparación de candidatos cambia de resultado (p. ej. deja de haber empate técnico, o el
  ensemble ya no necesita tuning para superar a la regresión logística), las instancias 3, 10 y 11
  DEBEN reflejar la justificación metodológica real observada (FR-010), no la ya escrita si dejó de
  aplicar.
- Ninguna instancia se inserta fuera de una sección/slide ya existente — mismo criterio que
  `specs/002-.../data-model.md` (SC-004 equivalente implícito en este feature vía Principio V).
- Antes de dar el trabajo por terminado, `customer_segments.parquet` DEBE compararse (numéricamente)
  contra la versión pre-cambio para confirmar que es idéntico (SC-004 del spec).
