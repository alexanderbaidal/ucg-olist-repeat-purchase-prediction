# Content/Data Model: Corregir discrepancias Informe/Presentación vs. código real

Este feature combina un cambio de código (NB2) con actualizaciones de contenido (Informe/Presentación).
Describe dos entidades de datos nuevas (producidas por NB2) y las 6 unidades de contenido que consumen
sus resultados en los documentos finales.

## Entidad: Fila de desempeño por `customer_state`

Producida por la nueva celda de NB2 §5.2 (`subgroup_perf_state`), una fila por estado observado en
`raw_test` (hasta 27 posibles, según los estados presentes en el conjunto de test).

| Campo | Tipo | Descripción |
|---|---|---|
| `customer_state` (índice) | string (2 letras) | Código de estado brasileño, p. ej. `SP`, `RJ` |
| `n` | int | Tamaño de muestra del estado en el conjunto de test |
| `tasa_real` | float [0,1] | Tasa de recompra observada (`repeat_purchase.mean()`) dentro del estado |
| `tasa_predicha` | float [0,1] | Tasa de recompra predicha por el modelo (`y_pred.mean()`) dentro del estado |
| `avg_precision` | float [0,1] o `NaN` | PR-AUC del subgrupo; `NaN` si el estado tiene una sola clase (`nunique()==1`) |

**Regla de validación**: la suma de `n` sobre todas las filas DEBE ser igual a `len(raw_test)`
(12,643 en la última ejecución confirmada) — ningún cliente de test queda fuera de algún estado.

## Entidad: Campo `n_iter_actual` en `model_card.json`

Campo nuevo, hermano de `hyperparameters` (o anidado dentro de él, según se decida en implementación —
ver `research.md` §2), que registra las rondas de boosting realmente entrenadas por `final_model`.

| Campo | Tipo | Descripción |
|---|---|---|
| `n_iter_actual` | int | `final_model.n_iter_` — rondas de boosting reales, distinto de `hyperparameters.max_iter` (el techo buscado) |

**Regla de validación**: `n_iter_actual <= hyperparameters.max_iter` siempre (early stopping solo
puede detener el entrenamiento antes del techo, nunca superarlo).

## Unidades de contenido a actualizar en `docs/`

Igual que en `specs/001-informe-metricas-negocio/data-model.md`, cada inserción/edición comparte esta
estructura:

| Campo | Descripción |
|---|---|
| `hallazgo` | `subgrupo_customer_state` o `rondas_boosting_reales` |
| `documento` | Informe Técnico (.docx) o Presentación de Defensa (.pptx) |
| `ubicacion` | Sección/slide exacta (re-verificada por contenido, ver `research.md` §3) |
| `fuente_de_verdad` | `notebooks/artifacts/model_card.json` (siempre, tras la re-ejecución de NB2) |

### Instancias

| # | Hallazgo | Documento | Ubicación |
|---|---|---|---|
| 1 | `subgrupo_customer_state` | Informe | §3.4 (resultados) y/o §4.1 (sesgos y equidad) |
| 2 | `subgrupo_customer_state` | Presentación | Slide de desempeño por subgrupo / errores y casos atípicos |
| 3 | `subgrupo_customer_state` | Presentación | Slide de ética — sesgos y equidad |
| 4 | `rondas_boosting_reales` | Informe | §3.5 (prototipado y despliegue) |
| 5 | `rondas_boosting_reales` | Presentación | Slide de prototipo y despliegue |

## Reglas de consistencia

- Las instancias 1, 2 y 3 (`subgrupo_customer_state`) DEBEN citar el mismo hallazgo real (mismo mensaje
  sustantivo, adaptado en extensión a cada documento/slide) — no se permite que el Informe diga una
  cosa y la Presentación otra sobre el mismo resultado.
- Las instancias 4 y 5 (`rondas_boosting_reales`) DEBEN citar exactamente `n_iter_actual` de
  `model_card.json`, no `hyperparameters.max_iter`.
- Ninguna instancia se inserta fuera de una sección/slide ya existente (SC-004 del spec).
- Si la fila de `subgroup_perf_state` de algún estado tiene `avg_precision = NaN` o un `n` muy pequeño
  frente al resto, las instancias 1–3 DEBEN mencionar esa salvedad explícitamente (FR-005).
