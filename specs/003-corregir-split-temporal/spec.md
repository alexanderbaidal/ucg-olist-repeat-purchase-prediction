# Feature Specification: Corregir split train/val/test no temporal en NB1 (a pesar de estar documentado como temporal)

**Feature Branch**: `003-corregir-split-temporal`

**Created**: 2026-09-18

**Status**: Draft

**Input**: User description: "Corregir el split train/val/test en NB1_Estrategia_Datos_EDA.ipynb: la celda de partición ordena `model_df` por `order_purchase_timestamp` con la intención documentada (celda markdown de NB1, informe técnico §3.2, y presentación de defensa slide 12) de que el conjunto de test contenga, en su mayoría, a los clientes más recientes (\"validez temporal\"). Sin embargo, las dos llamadas a `train_test_split` no pasan `shuffle=False`, por lo que sklearn usa su comportamiento por defecto (`shuffle=True`) y hace un StratifiedShuffleSplit que baraja aleatoriamente las filas, destruyendo el orden temporal previo. Verificado empíricamente contra raw_train/val/test.parquet: los tres splits cubren el mismo rango de fechas completo (2016-09/10 a 2018-07-19) con medias casi idénticas — es decir, hoy es un split aleatorio estratificado por clase, no temporal, contradiciendo lo que dicen NB1, el informe y la presentación. La corrección debe hacer que el código cumpla la intención ya documentada (test con los clientes más recientes), en vez de reescribir la documentación para bajar la exigencia, preservando en la medida de lo posible el desbalance de clases documentado (~3.3%) en cada partición. Es un cambio de datos aguas arriba: cambian los artifacts de NB1 y, en cascada, todo lo que depende de ellos en NB2 (comparación de modelos, hiperparámetros óptimos, métricas, umbral, lift, importancia de variables, model_card.json, test_predictions.parquet). La segmentación RFM no depende de este split y no debería cambiar. Por la política de sincronización de entregables del repo, el Informe Técnico y la Presentación de Defensa deben actualizarse con los números que resulten de la re-ejecución."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El conjunto de test refleja realmente a los clientes más recientes (Priority: P1)

Un miembro del comité de defensa o un lector técnico del Informe que audita la sección de diseño experimental (§3.2) necesita que la partición train/val/test se comporte exactamente como se describe: el conjunto de test debe estar compuesto, en su mayoría, por los clientes cuyo primer pedido ocurrió más recientemente, para que la evaluación se parezca a un despliegue real (entrenar con el pasado, evaluar con el futuro más próximo).

**Why this priority**: es la discrepancia raíz — si esto no se corrige, todo lo demás (métricas, umbral, lift, importancia de variables) sigue evaluándose sobre una partición aleatoria que contradice la narrativa de validez temporal ya publicada en NB1, el Informe y la Presentación.

**Independent Test**: ejecutar NB1 completo (Run All) y verificar, inspeccionando `order_purchase_timestamp` en `raw_train/val/test.parquet`, que el rango temporal de train antecede al de val, y el de val antecede al de test (sin el solapamiento total que existe hoy, donde los tres splits cubren el mismo rango 2016-09/10 a 2018-07-19 con medias casi idénticas).

**Acceptance Scenarios**:

1. **Given** NB1 se ejecuta completo con `RANDOM_STATE=42`, **When** se particiona `model_df` en train/val/test, **Then** el percentil 90 de `order_purchase_timestamp` en train es anterior (o igual) al percentil 10 de test, evidenciando que test concentra a los clientes cronológicamente más recientes y train a los más antiguos.
2. **Given** la partición corregida, **When** se compara la tasa de recompra (`repeat_purchase.mean()`) de train, val y test, **Then** las tres tasas permanecen razonablemente cercanas al ~3.3% documentado en `eda_summary.json` (sin un desbalance extremo entre splits causado por estacionalidad), de forma consistente con el desbalance ya reportado y usado para justificar la métrica primaria (PR-AUC).
3. **Given** el código corregido, **When** un lector inspecciona la celda de partición de NB1, **Then** el mecanismo usado para partir los datos es explícito y verificable (no depende de un `sort_values` cuyo efecto se descarta silenciosamente por un `shuffle` implícito).

---

### User Story 2 - Los artifacts y las métricas aguas abajo reflejan la partición corregida (Priority: P2)

Un lector del Informe o la Presentación que cita PR-AUC, ROC-AUC, el umbral de decisión, el lift del decil superior o la importancia de variables necesita que esas cifras correspondan al modelo entrenado sobre la partición temporal real, no sobre la partición aleatoria actual.

**Why this priority**: es la consecuencia obligatoria de US1 — cambiar qué filas caen en train/val/test invalida cualquier artifact de NB2 calculado sobre la partición anterior; sin re-ejecutar y re-sincronizar, el Informe y la Presentación quedarían citando números de un modelo que ya no existe.

**Independent Test**: tras aplicar la corrección de US1, ejecutar NB1 → NB2 de punta a punta (Run All en ambos, en ese orden) y confirmar que todos los artifacts en `notebooks/artifacts/` quedan regenerados con timestamp posterior al cambio, y que `model_card.json` refleja las métricas resultantes de la nueva partición.

**Acceptance Scenarios**:

1. **Given** la corrección de US1 aplicada en NB1, **When** se ejecuta NB1 completo, **Then** se regeneran `X_train/val/test.parquet`, `y_train/val/test.parquet`, `raw_train/val/test.parquet` y `preprocessor.joblib` con los nuevos límites de partición.
2. **Given** los artifacts de NB1 regenerados, **When** se ejecuta NB2 completo, **Then** se regeneran `repeat_purchase_model.joblib`, `model_card.json` y `test_predictions.parquet`, con la comparación de candidatos, la búsqueda de hiperparámetros, el umbral calibrado, el lift del decil superior y la importancia de variables recalculados sobre la nueva partición (sin necesidad de fijar manualmente ningún valor anterior).
3. **Given** `customer_segments.parquet` depende de `orders_history.parquet` (historial completo, no de la partición train/val/test), **When** se re-ejecuta NB1, **Then** la segmentación RFM (conteos y perfiles por segmento) no cambia respecto a la versión actual.
4. **Given** los nuevos valores en `model_card.json`, **When** se actualizan `docs/Informe_Tecnico_ShopMart_Olist.docx` (+ su PDF) y `docs/Defensa_ShopMart_Olist.pptx`, **Then** toda cifra de desempeño citada en ambos documentos (PR-AUC de val/test, ROC-AUC, Brier score, lift del decil superior, hiperparámetros ganadores, importancia de variables, umbral) coincide exactamente con `model_card.json` regenerado, sin dejar ningún valor de la ejecución anterior.

---

### User Story 3 - La narrativa de "validez temporal" describe con precisión el mecanismo real (Priority: P3)

Un lector de NB1, el Informe (§3.2) o la Presentación (slide 12 y su nota de orador) necesita que el texto que ya afirma "se ordena por `order_purchase_timestamp`... para que el test contenga a los clientes más recientes" describa con precisión cómo se concilia esa validez temporal con la preservación del desbalance de clases documentado, sin dejar la impresión de que sigue siendo un `train_test_split` aleatorio estratificado.

**Why this priority**: es una consecuencia de redacción, no de cálculo — una vez corregido el mecanismo (US1) y re-sincronizados los números (US2), el texto que ya existe hoy sobre validez temporal deja de ser falso, pero puede requerir un ajuste menor de precisión (p. ej. aclarar que el balance de clases se preserva por partición dentro de cada clase, no por barajado aleatorio global).

**Independent Test**: leer la celda markdown de partición en NB1, el Informe §3.2 y la Presentación (slide 12 + nota de orador) tras la corrección, y confirmar que ninguno de los tres describe el mecanismo de forma inconsistente con el código real de NB1.

**Acceptance Scenarios**:

1. **Given** el código corregido de NB1, **When** se relee la celda markdown de la sección "6. Particiones y persistencia de artifacts", **Then** el texto describe con precisión cómo se logra simultáneamente el orden temporal y la preservación del desbalance de clases, sin afirmar un mecanismo que el código no ejecuta.
2. **Given** el Informe §3.2 y la Presentación slide 12 actualizados, **When** se comparan contra la celda markdown de NB1, **Then** los tres describen el mismo mecanismo sin contradicción entre sí.

---

### Edge Cases

- Si el desbalance de clases varía por estacionalidad (ya documentado en `eda_summary.json` como "estacionalidad moderada en volumen y tasa de recompra por mes"), un split puramente cronológico y contiguo podría producir una tasa de recompra notablemente distinta en test respecto a train/val. La corrección DEBE evitar ese riesgo sin sacrificar el orden temporal global (ver FR-002).
- Si, tras la partición corregida, algún estado de `customer_state` o categoría de `main_category_grouped` queda con muestra insuficiente en alguno de los splits (por ejemplo, una categoría que solo aparece en pedidos muy recientes y por tanto solo cae en test), el análisis de subgrupo existente en NB2 debe seguir manejándolo igual que hoy (marcar como no disponible/orientativo), sin romper la ejecución.
- Si al re-ejecutar NB1 → NB2 con `RANDOM_STATE=42` el modelo ganador de la comparación de candidatos cambia (por ejemplo, si la regresión logística deja de empatar técnicamente con el ensemble bajo la nueva partición), la decisión de negocio de continuar con `HistGradientBoostingClassifier` DEBE reevaluarse con la misma justificación metodológica ya usada (mayor margen de mejora vía tuning), documentando explícitamente si el resultado cambia.
- Si el porcentaje de filas en cada split ya no es exactamente 70/15/15 por el efecto de partir de forma temporal dentro de cada clase (redondeos), la desviación aceptable es la misma tolerancia ya usada hoy entre splits (diferencias de fracción de punto porcentual, como las ya observadas: train 3.30%, val 3.30%, test 3.31%).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: NB1 DEBE particionar `model_df` en train/val/test (70/15/15) de forma que el conjunto de test esté compuesto, en su mayoría, por los clientes cronológicamente más recientes según `order_purchase_timestamp`, y train por los más antiguos — cumpliendo la intención ya documentada en la celda markdown de la sección 6, en el Informe §3.2 y en la Presentación slide 12.
- **FR-002**: La partición corregida DEBE preservar el desbalance de clases documentado (~3.3% de recompra) en cada uno de los tres splits, dentro de una tolerancia comparable a la ya observada hoy entre splits (décimas de punto porcentual), sin depender de un barajado aleatorio global que rompa el orden temporal de FR-001.
- **FR-003**: El mecanismo de partición NO DEBE depender de un efecto secundario silencioso (como un `sort_values` cuyo orden se descarta por un parámetro de barajado no explícito) — debe ser verificable por inspección directa del código que el resultado cumple FR-001 y FR-002.
- **FR-004**: `RANDOM_STATE=42` DEBE seguir aplicándose a cualquier paso de la partición que conserve un componente aleatorio (p. ej., desempate dentro de un mismo instante de tiempo, si aplica).
- **FR-005**: Ningún otro elemento ya validado del pipeline (definición del target, filtros de elegibilidad y censura de 90 días, lista de `FEATURES`/`NUMERIC_FEATURES`/`CATEGORICAL_FEATURES`, pipeline de preprocesamiento) DEBE modificarse como parte de esta corrección — el cambio se limita a cómo se asignan las filas ya elegibles a train/val/test.
- **FR-006**: Tras aplicar la corrección, NB1 DEBE ejecutarse completo (Run All) para regenerar `X_train/val/test.parquet`, `y_train/val/test.parquet`, `raw_train/val/test.parquet` y `preprocessor.joblib`, y NB2 DEBE ejecutarse completo (Run All) a continuación para regenerar `repeat_purchase_model.joblib`, `model_card.json` y `test_predictions.parquet` sobre la nueva partición.
- **FR-007**: `customer_segments.parquet` (segmentación RFM) NO DEBE cambiar como resultado de esta corrección, dado que se calcula sobre `orders_history.parquet` (historial completo), independiente de la partición train/val/test.
- **FR-008**: `docs/Informe_Tecnico_ShopMart_Olist.docx` (y su PDF exportado) y `docs/Defensa_ShopMart_Olist.pptx` DEBEN actualizarse para citar exactamente las métricas, hiperparámetros, umbral, lift e importancia de variables del `model_card.json` regenerado, reemplazando cualquier cifra de la ejecución anterior — según la política de sincronización de entregables ya vigente en el repositorio.
- **FR-009**: La celda markdown de NB1 (sección 6), el Informe §3.2 y la Presentación slide 12 (incluida su nota de orador) DEBEN describir el mecanismo de partición real de forma consistente entre sí, sin afirmar un comportamiento que el código no ejecuta.
- **FR-010**: Si el resultado de la comparación de candidatos (DummyClassifier / LogisticRegression / HistGradientBoostingClassifier) cambia bajo la nueva partición respecto al empate técnico ya reportado, el Informe §3.1/3.3 y la Presentación (slide 11 y su nota de orador) DEBEN actualizarse para reflejar el resultado real de la re-ejecución, manteniendo la misma justificación metodológica para la elección del modelo final si sigue aplicando, o documentando el cambio de criterio si ya no aplica.

### Key Entities

- **Partición train/val/test corregida**: la asignación de cada cliente elegible (fila de `model_df`) a train, val o test, determinada por su posición cronológica (`order_purchase_timestamp`) dentro de su propia clase de `repeat_purchase`, de forma que se cumplen simultáneamente el orden temporal global (FR-001) y la preservación del desbalance de clases (FR-002).
- **Artifacts derivados de NB1** (`X_train/val/test.parquet`, `y_train/val/test.parquet`, `raw_train/val/test.parquet`, `preprocessor.joblib`): dependen directamente de la nueva partición y deben regenerarse.
- **Artifacts derivados de NB2** (`repeat_purchase_model.joblib`, `model_card.json`, `test_predictions.parquet`): dependen transitivamente de la nueva partición (a través de los artifacts de NB1) y deben regenerarse.
- **`customer_segments.parquet`**: no depende de la partición corregida; se declara explícitamente fuera del alcance del cambio.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Tras la corrección, el percentil 90 de `order_purchase_timestamp` en train es anterior (o igual) al percentil 10 de test, verificable por inspección directa de `raw_train.parquet`/`raw_test.parquet` — a diferencia del estado actual, donde ambos splits cubren el mismo rango completo de fechas.
- **SC-002**: La tasa de recompra de cada split (train, val, test) tras la corrección se mantiene dentro de ±0.5 puntos porcentuales del ~3.3% documentado en `eda_summary.json`, evitando que la estacionalidad introduzca un desbalance extremo entre splits.
- **SC-003**: El 100% de las cifras de desempeño (PR-AUC, ROC-AUC, Brier score, lift del decil superior, hiperparámetros ganadores, umbral de decisión, importancia de variables) citadas en el Informe y la Presentación coinciden exactamente con el `model_card.json` regenerado tras este cambio.
- **SC-004**: `customer_segments.parquet` permanece byte-idéntico (o numéricamente idéntico en sus columnas agregadas) antes y después de este cambio, confirmando que la segmentación RFM no se vio afectada.
- **SC-005**: La celda markdown de NB1, el Informe §3.2 y la Presentación slide 12 describen el mismo mecanismo de partición sin contradicción entre sí, verificable por lectura comparativa directa.
- **SC-006**: Ningún elemento fuera del alcance declarado (target, filtros de elegibilidad, censura de 90 días, features, pipeline de preprocesamiento, RANDOM_STATE) cambia de valor como resultado de este trabajo.

## Assumptions

- Se asume que "la mayoría de los clientes más recientes en test" (tal como ya lo describe la documentación existente) no exige una partición estrictamente monótona en el tiempo (es decir, un único punto de corte cronológico global), sino que permite preservar el balance de clases particionando cronológicamente dentro de cada clase por separado — esto es necesario porque un corte estrictamente cronológico y global no puede combinarse con la preservación exacta del desbalance de clases si la tasa de recompra varía por estacionalidad (ya documentada como "moderada").
- Se asume que la proporción 70/15/15 ya validada se mantiene como el esquema de partición; este trabajo corrige *cómo* se asignan las filas a cada partición, no las proporciones.
- Se asume que el comité de defensa y los lectores del Informe consideran aceptable una tolerancia de ±0.5 puntos porcentuales en la tasa de recompra entre splits como preservación razonable del "desbalance severo (~3.3%)" ya reportado, en línea con la tolerancia que ya existe hoy entre splits (3.30%/3.30%/3.31%).
- Se asume que ningún otro documento o artifact fuera de `docs/Informe_Tecnico_ShopMart_Olist.docx`, su PDF, `docs/Defensa_ShopMart_Olist.pptx` y los artifacts de `notebooks/artifacts/` cita cifras de desempeño del modelo que requieran actualización.
