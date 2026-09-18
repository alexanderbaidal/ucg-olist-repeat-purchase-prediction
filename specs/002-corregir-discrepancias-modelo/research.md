# Research: Corregir discrepancias Informe/Presentación vs. código real

## 1. Dónde y cómo agregar el análisis de subgrupo por `customer_state` en NB2

**Decision**: agregar una celda de código inmediatamente después de la celda existente de
`subgroup_perf` (NB2 §5.2, celda que hoy agrupa solo por `main_category_grouped`), reutilizando el
mismo `raw_test` (que ya tiene las columnas `score` y `y_pred` calculadas ahí mismo) y el mismo patrón
exacto:

```python
subgroup_perf_state = (
    raw_test.groupby("customer_state")
    .apply(lambda g: pd.Series({
        "n": len(g),
        "tasa_real": g[TARGET].mean(),
        "tasa_predicha": g["y_pred"].mean(),
        "avg_precision": average_precision_score(g[TARGET], g["score"]) if g[TARGET].nunique() > 1 else np.nan,
    }))
    .sort_values("n", ascending=False)
)
subgroup_perf_state.round(3)
```

**Rationale**: es el mismo patrón, mismas columnas y misma función (`average_precision_score` con
guarda `nunique() > 1` para evitar el mismo error que ya se previene en el análisis por categoría) que
el código existente — máxima consistencia con lo ya validado, cero riesgo de introducir un criterio
distinto de medición entre categoría y estado. `customer_state` ya está presente en `raw_test.parquet`
(persistido por NB1 como columna de `raw_*` — ver el contrato de artifacts en `CLAUDE.md`), así que no
requiere ningún dato adicional.

**Alternatives considered**:
- **Reemplazar la celda existente por un groupby combinado (categoría + estado)**: descartada —
  mezclar ambos ejes en una sola tabla dificulta la lectura y no es lo que pide el spec (SC-001 pide
  un hallazgo trazable *por estado*, específicamente).
- **Usar una librería de fairness dedicada (p. ej. `fairlearn`)**: descartada — sería una dependencia
  nueva no justificada por el alcance (Principio I: no agregar dependencias sin necesidad clara), y el
  patrón `groupby` ya usado en el proyecto es suficiente para responder la pregunta concreta del spec.

## 2. Dónde y cómo persistir las rondas de boosting reales en `model_card.json`

**Decision**: en la celda de NB2 §7 que construye el diccionario `model_card` (antes de
`json.dump(...)`), agregar un campo nuevo dentro de `hyperparameters` o como campo hermano —
`"n_iter_actual": int(final_model.n_iter_)` — leyendo el atributo `n_iter_` que scikit-learn ya expone
en todo `HistGradientBoostingClassifier` entrenado (documenta cuántas rondas de boosting se
entrenaron realmente, sea que haya habido early stopping o no).

**Rationale**: `n_iter_` es un atributo público y estándar de scikit-learn — no requiere ningún
cálculo adicional ni volver a entrenar nada, solo leerlo del objeto `final_model` ya ajustado. Nombrarlo
`n_iter_actual` (en vez de sobrescribir `hyperparameters.max_iter`, que debe seguir representando el
techo de búsqueda tal como está documentado hoy) preserva la distinción semántica entre "límite
buscado" y "rondas realmente entrenadas" — la raíz misma de la discrepancia que este feature corrige.

**Alternatives considered**:
- **Sobrescribir `hyperparameters.max_iter` con el valor real**: descartada — `max_iter` es, por
  definición, el hiperparámetro de búsqueda (el resultado de `RandomizedSearchCV`); cambiar su
  significado rompería la trazabilidad de qué se buscó vs. qué se entrenó, y confundiría a cualquier
  lector futuro de `model_card.json`.
- **Calcular `n_iter_` fuera del notebook (script aparte)**: descartada — innecesario, el valor ya está
  disponible en el mismo objeto que NB2 serializa; agregar un script aparte sería scope creep frente al
  Principio I.

## 3. Ubicación real de las slides a editar en la Presentación (lección de `specs/001-informe-metricas-negocio`)

**Decision**: antes de editar cualquier slide, volver a localizar cada una por **búsqueda de
contenido** (`shape.text_frame.text`) sobre `prs.slides` en orden de presentación real, no asumir la
posición por el nombre de archivo `slideN.xml` ni por el número de slide citado en conversaciones
previas.

**Rationale**: durante la implementación de `specs/001-informe-metricas-negocio` se confirmó que el
orden de archivo (`ppt/slides/slideN.xml`) no coincide con el orden de presentación real
(`sldIdLst` en `presentation.xml`) para varias slides de este mismo `.pptx` — la slide de resultados
citada como "slide 11" en la conversación resultó estar en la posición de presentación `idx=12`
(0-indexed). Asumir la posición por nombre de archivo o por un número citado de memoria arriesga editar
la slide equivocada.

**Alternatives considered**:
- **Confiar en los números de slide ya usados en el spec** (p. ej. "slide 18" para la cifra de
  iteraciones): aceptable como punto de partida (fue verificado por contenido durante el análisis de
  discrepancias, no solo asumido), pero se re-verifica por contenido en el momento de editar, como red
  de seguridad barata contra que el archivo haya cambiado de orden entre entonces y ahora.

## 4. Umbral para matizar estados con muestra pequeña

**Decision**: no se define un umbral numérico nuevo y fijo. Se aplica el mismo criterio informal ya
usado implícitamente por el proyecto para categorías de producto poco frecuentes: si un estado tiene
`avg_precision` no disponible (`NaN`, por tener una sola clase) o un tamaño de muestra visiblemente
atípico frente al resto (visible por inspección de la tabla ordenada por `n`), se menciona
explícitamente como caso a tratar con cautela en el texto actualizado, sin inventar un punto de corte
numérico que el resto del proyecto no usa en ningún otro lado.

**Rationale**: introducir un umbral numérico nuevo y arbitrario (p. ej. "n < 30") sería una decisión de
metodología no solicitada por el spec ni respaldada por el resto del proyecto — el patrón existente
(categorías) tampoco define un umbral duro, solo señala cualitativamente "el volumen por subgrupo es
limitado". Mantener el mismo nivel de rigor (ni más laxo ni más estricto) es lo consistente con
FR-005.

**Alternatives considered**:
- **Fijar `n >= 30` como regla dura** (heurística común para aproximaciones asintóticas): descartada —
  sería una decisión estadística nueva no pedida por el spec ni usada en el resto del proyecto; se
  documenta como posible mejora futura si el usuario lo pide, no se decide unilateralmente acá.

## 5. Qué citar en Informe/Presentación cuando el hallazgo real todavía no existe (tiempo de planeación)

**Decision**: `research.md`, `data-model.md` y `quickstart.md` de este feature describen la
**estructura** del hallazgo a citar (rango de `avg_precision` por estado, cuál es el mejor/peor
estado, si algún estado requiere matiz) — no el valor final, que solo existe después de re-ejecutar
NB2. Las tareas de edición de documentación en `tasks.md` deben ejecutarse **después** de la
re-ejecución de NB2 en `tasks.md`, nunca antes, y deben citar el resultado real observado, no un
placeholder.

**Rationale**: consistente con el Principio IV (no re-derivar/asumir resultados) y con FR-004/FR-007 —
los documentos deben reportar lo que el código realmente produce, no una redacción anticipada de lo
que "probablemente" va a decir la tabla.
