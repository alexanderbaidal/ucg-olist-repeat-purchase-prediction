# Research: Corregir split train/val/test no temporal en NB1

## 1. Por qué no basta con agregar `shuffle=False`

**Decision**: no usar `train_test_split(..., shuffle=False, stratify=y)`.

**Rationale**: `sklearn.model_selection.train_test_split` no permite combinar `shuffle=False` con
`stratify` distinto de `None` — lanza `ValueError`. Como la estratificación (preservar ~3.3% de
recompra en cada split) es un requisito ya validado (Principio IV, FR-002 del spec), y el orden
temporal es el otro requisito (FR-001), ambos no pueden lograrse con una sola llamada a
`train_test_split`. Se necesita un mecanismo propio.

**Alternatives considered**:
- **Quitar `stratify` y usar `shuffle=False` puro** (corte cronológico global contiguo: primer 70% →
  train, siguiente 15% → val, último 15% → test): descartada tras verificarlo empíricamente contra
  `raw_train/val/test.parquet` recombinados (84,282 filas) — produce una tasa de recompra de **3.97%
  en train, 2.02% en val y 1.46% en test**, muy lejos del ~3.3% documentado en `eda_summary.json` y
  usado como baseline en todo el Informe/Presentación. La causa: incluso tras excluir los últimos 90
  días (censura), la tasa de recompra sigue decreciendo cuanto más cerca del final de la ventana está
  el primer pedido — hay menos tiempo transcurrido para que ocurra una segunda compra. Un corte
  cronológico global rompería la premisa "~3.3% severamente desbalanceado" ya validada y usada para
  justificar PR-AUC como métrica primaria, y haría que el modelo final se comparara contra un baseline
  de test completamente distinto al reportado hasta ahora.

## 2. Mecanismo elegido: partición temporal dentro de cada clase

**Decision**: reemplazar las dos llamadas a `train_test_split` por una función que, para cada valor de
`repeat_purchase` (0 y 1) por separado, ordena las filas de esa clase por `order_purchase_timestamp`
(con un tiebreaker determinístico, ver §3) y toma el primer 70% → train, el siguiente 15% → val, el
último 15% → test; luego concatena los tres fragmentos de ambas clases y los reordena por timestamp
para persistencia.

```python
def temporal_split_by_class(df, target=TARGET, ts_col="order_purchase_timestamp",
                             train_frac=0.70, val_frac=0.15):
    train_parts, val_parts, test_parts = [], [], []
    for _, sub in df.groupby(target):
        sub = sub.sort_values([ts_col, "order_id"], kind="mergesort")
        n = len(sub)
        n_train = int(round(n * train_frac))
        n_val = int(round(n * val_frac))
        train_parts.append(sub.iloc[:n_train])
        val_parts.append(sub.iloc[n_train:n_train + n_val])
        test_parts.append(sub.iloc[n_train + n_val:])
    train_df = pd.concat(train_parts).sort_values(ts_col).reset_index(drop=True)
    val_df = pd.concat(val_parts).sort_values(ts_col).reset_index(drop=True)
    test_df = pd.concat(test_parts).sort_values(ts_col).reset_index(drop=True)
    return train_df, val_df, test_df
```

**Rationale**: se verificó empíricamente sobre las 84,282 filas elegibles reales (recombinando
`raw_train/val/test.parquet` actuales, que contienen exactamente el conjunto elegible completo aunque
particionado al azar):

| Split | n | tasa de recompra | rango de fechas |
|---|---|---|---|
| train | 58,997 | 3.30% | 2016-09-15 → 2018-03-18 |
| val | 12,642 | 3.30% | 2018-01-23 → 2018-05-12 |
| test | 12,643 | 3.31% | 2018-03-21 → 2018-07-19 |

- **n y tasa de recompra por split son prácticamente idénticos a los ya reportados hoy** (58,997 /
  12,642 / 12,643 y 3.30% / 3.30% / 3.31%) — porque tomar una fracción fija (70/15/15) del conteo de
  cada clase produce el mismo tamaño de partición sin importar si las filas se eligen al azar o por
  orden cronológico. Esto significa que **SC-002 se cumple con margen** (muy por debajo de la
  tolerancia de ±0.5 puntos porcentuales) y que el criterio de éxito de negocio ya validado (desbalance
  ~3.3%, baseline PR-AUC≈0.033) no se ve alterado por el cambio de mecanismo.
- **Orden temporal global**: el percentil 90 de `order_purchase_timestamp` en train (2018-02-20) es
  anterior al percentil 10 en test (2018-05-16) — cumple FR-001/SC-001. El 0% de las filas de train
  tiene timestamp posterior al mínimo de test en la ejecución verificada.
- Hay solapamiento parcial y esperado entre val y test en sus rangos de fecha (val llega hasta
  2018-05-12, test empieza en 2018-03-21) porque la clase positiva (solo 2,782 filas) tiene su propio
  corte temporal ~2 meses antes que la clase negativa (su percentil 70 cae en 2018-01-23 vs. 2018-03-18
  de la clase negativa) — es decir, los clientes que sí recompran tienden a tener su primer pedido más
  temprano en la ventana del dataset que los que no recompran, algo coherente con que necesitan tiempo
  para generar esa segunda compra incluso más allá de la censura de 90 días. Como la clase negativa
  domina >96% de las filas de cada split, el orden temporal "en su mayoría" (tal como ya lo describe
  NB1/Informe/Presentación) se cumple ampliamente pese a este solapamiento minoritario — se documenta
  explícitamente en vez de ocultarse (Edge Cases del spec).

**Alternatives considered**:
- **`sklearn.model_selection.TimeSeriesSplit`**: descartada — está diseñada para validación cruzada de
  series temporales con ventanas expansivas/deslizantes sobre un único flujo ordenado, no para producir
  tres particiones hold-out con una restricción de balance de clases; hubiera requerido la misma lógica
  de partición-por-clase por encima, sin aportar nada sobre la solución directa.
- **Interpolar entre corte global y estratificación** (p. ej., cuantiles de tiempo ponderados por
  clase): descartada por complejidad injustificada — la partición por clase ya cumple ambos requisitos
  con una implementación simple y auditable, consistente con el Principio V (ediciones quirúrgicas).

## 3. Tiebreaker determinístico para timestamps duplicados

**Decision**: usar `sort_values([ts_col, "order_id"], kind="mergesort")` (orden secundario por
`order_id`, y `mergesort` por ser un algoritmo de ordenamiento estable) en vez de un `sort_values`
solo por timestamp.

**Rationale**: se detectaron 209 timestamps duplicados sobre las 84,282 filas elegibles (dos o más
clientes con el mismo `order_purchase_timestamp` a nivel de segundo). Sin un criterio de desempate
explícito, el orden relativo de esas filas dependería del algoritmo de ordenamiento interno de pandas
(no garantizado estable por defecto), lo que podría producir una partición ligeramente distinta entre
ejecuciones o entornos (local/Colab/Databricks) — justo el tipo de no-determinismo que Principio II
busca evitar. Como `order_id` es único por fila, ordenar por `(timestamp, order_id)` con `mergesort`
elimina cualquier ambigüedad sin introducir ningún componente aleatorio ni depender de `RANDOM_STATE`
para este paso puntual.

**Alternatives considered**:
- **Dejar el desempate implícito** (orden de aparición en el DataFrame): descartada — el orden de
  aparición en `model_df` ya depende de merges previos de NB1, no es una propiedad estable garantizada
  entre ejecuciones.
- **Usar `customer_unique_id` como desempate**: descartada — `order_id` es la clave que ya identifica
  de forma única cada fila de `model_df` (un registro por primer pedido), es más directa y no requiere
  verificar unicidad adicional.

## 4. Qué contenido de NB1/NB2/`docs/` depende de esta partición (y qué no)

**Decision**: antes de editar cualquier documento, clasificar cada mención numérica existente según si
depende de la partición train/val/test o no, usando el resultado real de la re-ejecución (no una
suposición) para decidir qué texto cambia.

**Rationale — inventario verificado por inspección de las celdas markdown de ambos notebooks**:

| Ubicación | Depende del split? | Nota |
|---|---|---|
| NB1 celda markdown de nulos estructurales (0.16%–2.98%) | No | Se calcula sobre las 84,282 filas elegibles completas, antes de particionar. |
| NB1 "Hallazgo clave #3" — tasa de recompra por `review_score` (2.96%–3.46%) | No | Ídem — EDA sobre la tabla completa. |
| NB1 celda markdown de partición (§6) | **Sí** | Es la descripción del mecanismo en sí — se reescribe como parte de este feature (US3, FR-009), no como consecuencia de la re-ejecución. |
| NB2 celda markdown §3 ("0.0468 vs. 0.0455", comparación de candidatos) | **Sí** | Cifras de CV calculadas sobre `X_train`/`y_train`, que cambian con la partición. Se revisa tras el Run All. |
| NB2 resto de celdas markdown (§2, §4, §5.1, §5.2, §5.3, §6, §7, conclusión) | No cita cifras hardcodeadas | Describen método o remiten a "el model card"/"arriba" sin citar números en prosa — no requieren edición salvo que el hallazgo cualitativo cambie (p. ej. si la lista de top-features de §5.3 cambia de composición, no solo de cifra). |
| Informe §2.1/§2.3 (n_customers, tasas por review_score) | No | Mismas cifras de EDA sobre la tabla completa. |
| Informe §3.1/§3.2/§3.3/§3.4, §5.1 (comparación CV, métricas finales, lift, importancia) | **Sí** | Todas dependen de `model_card.json` regenerado. |
| Presentación slides 6–9 (datos, EDA, review score) | No | Idem EDA sobre tabla completa. |
| Presentación slides 5, 10–16 (baseline, candidatos, diseño experimental, métricas, negocio, importancia, límites) | **Sí** | Dependen de `model_card.json` regenerado. |
| Presentación slides 17–20 (RFM, ética/privacidad) | No | RFM usa `orders_history.parquet` completo, independiente del split (FR-007). |

**Alternatives considered**:
- **Actualizar todos los documentos "por si acaso"**: descartada — editar secciones/slides que no
  dependen del split sin necesidad viola el alcance quirúrgico (Principio V) y arriesga introducir
  inconsistencias nuevas donde no había ninguna.

## 5. Ubicación real de las slides a editar (lección heredada)

**Decision**: igual que en `specs/001-.../research.md` y `specs/002-.../research.md`, re-localizar cada
slide por **búsqueda de contenido** sobre el orden real de presentación (`sldIdLst` en
`presentation.xml`), no por el nombre de archivo `slideN.xml` ni por el número citado de memoria en
esta conversación.

**Rationale**: ya se confirmó en features anteriores de este mismo repo que ambos órdenes no coinciden
para varias slides de este `.pptx`. Los números de slide usados en este `plan.md` (5, 10–16) fueron
verificados por contenido durante el análisis de discrepancias previo, pero se re-verifican en el
momento de editar como red de seguridad barata.

## 6. Qué citar en `docs/` mientras el resultado real todavía no existe (tiempo de planeación)

**Decision**: este `research.md`, `data-model.md` y `quickstart.md` describen la **estructura** de lo
que hay que verificar y sincronizar (qué tabla/campo de `model_card.json` citar en cada sección/slide),
no los valores finales — esos solo existen después de correr NB1 → NB2. `tasks.md` debe secuenciar la
edición de `docs/` estrictamente después del Run All de ambos notebooks.

**Rationale**: consistente con el Principio IV (no asumir/re-derivar resultados) y con FR-006/FR-008 —
los documentos deben reportar lo que el código realmente produce tras la corrección, no una redacción
anticipada de "probablemente el PR-AUC sube/baja".
