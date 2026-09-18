# Feature Specification: Corregir discrepancias Informe/Presentación vs. código real

**Feature Branch**: `002-corregir-discrepancias-modelo`

**Created**: 2026-09-18

**Status**: Draft

**Input**: User description: "Corregir las 2 discrepancias confirmadas entre el Informe Técnico (docs/Informe_Tecnico_ShopMart_Olist.docx), la Presentación de Defensa (docs/Defensa_ShopMart_Olist.pptx) y el código real de los notebooks, encontradas en una revisión de discrepancias previa: (1) \"150 iteraciones\" del modelo final citadas en Informe §3.5 y slide 18 corresponden al techo del espacio de búsqueda de hiperparámetros (max_iter), no al modelo realmente entrenado — el modelo serializado tiene early_stopping=\"auto\" activo y su n_iter_ real es 62. (2) La afirmación de \"sin disparidades por customer_state/estado\" en Informe §3.4/§4.1 y slides 16/19 no está respaldada por ningún cálculo — el único análisis de subgrupo real en el código agrupa por categoría de producto, nunca por customer_state. Decisión aceptada: cerrar el gap (2) con evidencia real (agregar el análisis en NB2 §5.2, re-ejecutar NB2, actualizar los documentos con el hallazgo real), y corregir (1) citando el valor real de rondas de boosting entrenadas. Restricciones: no alterar RANDOM_STATE=42 ni ninguna decisión de modelado ya validada (target, features, splits, hiperparámetros de búsqueda, umbral de decisión); el cambio de NB2 es aditivo; los artifacts deben regenerarse y el Informe/Presentación deben sincronizarse según la constitución del proyecto (v1.1.0)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Auditoría real de desempeño por customer_state (Priority: P1)

Un lector del Informe Técnico o un miembro del comité de defensa que revisa la sección de consideraciones éticas (§4.1) necesita que la afirmación "no se detectaron disparidades sistemáticas por estado" esté respaldada por un cálculo real y trazable, no por una suposición.

**Why this priority**: es la discrepancia de mayor severidad de las dos — vive dentro de la sección de ética del Informe, donde se usa como evidencia de que el modelo no discrimina geográficamente, sin que exista ningún cómputo que la sustente.

**Independent Test**: ejecutar NB2 completo y verificar que §5.2 produce una tabla de desempeño por `customer_state` (análoga a la que ya existe por categoría de producto), y que el texto del Informe §3.4/§4.1 y de las slides 16/19 de la Presentación cita exactamente el hallazgo de esa tabla, no una afirmación genérica.

**Acceptance Scenarios**:

1. **Given** NB2 se ejecuta completo (Run All), **When** se llega a §5.2, **Then** se muestra una tabla de desempeño por `customer_state` (n, tasa_real, tasa_predicha, average_precision por estado), calculada sobre el mismo `raw_test` que ya tiene `score`/`y_pred`, con el mismo patrón que la tabla existente por `main_category_grouped`.
2. **Given** el Informe §3.4 y §4.1, **When** el lector busca la afirmación sobre disparidades por estado, **Then** encuentra un hallazgo trazable a esa tabla real (p. ej. rango de `average_precision` observado, o el estado con peor/mejor desempeño), no una afirmación sin cifra de respaldo.
3. **Given** algún estado con muestra pequeña en test (p. ej. RR, AP, AC, AM — con menos de ~20 filas cada uno) muestra un `average_precision` atípico o indefinido, **When** se reporta en el Informe/Presentación, **Then** el texto lo matiza explícitamente como orientativo/no concluyente, con el mismo criterio ya aplicado a categorías de producto poco frecuentes — nunca se presenta una conclusión "sin disparidades" limpia si la evidencia no la respalda.

---

### User Story 2 - Corrección de la cifra de iteraciones del modelo (Priority: P2)

Un lector técnico que revisa el Informe §3.5 o la slide 18 de la Presentación necesita que la cifra de "iteraciones del modelo" corresponda al modelo realmente entrenado, no al techo del espacio de búsqueda de hiperparámetros.

**Why this priority**: es una inexactitud factual verificable, pero de menor severidad que US1 porque no es una afirmación de fairness ni compromete la sección de ética — es una cifra descriptiva sobre el tamaño del modelo.

**Independent Test**: leer Informe §3.5 y la slide 18 de la Presentación y confirmar que la cifra citada coincide exactamente con el valor de rondas de boosting reales persistido en `model_card.json`, no con el hiperparámetro `max_iter` del espacio de búsqueda.

**Acceptance Scenarios**:

1. **Given** NB2 se ejecuta completo y empaqueta el modelo final, **When** se genera `model_card.json`, **Then** el archivo incluye el número real de rondas de boosting entrenadas (no solo el hiperparámetro `max_iter` de búsqueda).
2. **Given** el Informe §3.5, **When** el lector busca la cantidad de iteraciones del modelo, **Then** el valor citado coincide exactamente con el campo nuevo de `model_card.json`.
3. **Given** la slide 18 de la Presentación, **When** se lee la referencia a "iteraciones", **Then** coincide con el mismo valor citado en el Informe (mismo mensaje, sin contradicción entre documentos).

---

### Edge Cases

- Si al re-ejecutar NB2 con `RANDOM_STATE=42` el valor real de rondas de boosting entrenadas difiere del 62 observado en el análisis previo (p. ej. por una versión distinta de scikit-learn en el entorno de ejecución), el valor citado en los documentos DEBE ser el que efectivamente produzca esa re-ejecución — `model_card.json` manda, no el 62 de referencia usado para detectar el gap.
- Si el análisis de subgrupo por `customer_state` produce un `average_precision` indefinido para algún estado (todas las observaciones de ese estado tienen la misma clase, como ya puede ocurrir con categorías de producto poco frecuentes), debe manejarse de la misma forma que el código existente (marcar como no disponible, sin romper la celda ni el resto de la ejecución).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: NB2 §5.2 DEBE incluir un análisis de subgrupo por `customer_state` (columnas: n, tasa_real, tasa_predicha, average_precision), calculado sobre el mismo conjunto de test y las mismas columnas (`score`, `y_pred`) ya usadas para el análisis existente por `main_category_grouped`.
- **FR-002**: El nuevo análisis DEBE usar `RANDOM_STATE=42` y NO DEBE alterar ningún hiperparámetro, split, feature o umbral de decisión ya validado.
- **FR-003**: NB2 DEBE re-ejecutarse completo (Run All) tras el cambio para regenerar los artifacts derivados en `notebooks/artifacts/`. NB1 NO requiere re-ejecutarse, porque el cambio no toca `FEATURES`/`NUMERIC_FEATURES`/`CATEGORICAL_FEATURES` ni el pipeline de preprocesamiento.
- **FR-004**: El Informe Técnico (§3.4 y §4.1) y la Presentación (slides 16 y 19) DEBEN actualizarse para reportar el hallazgo real de la tabla de desempeño por `customer_state`, reemplazando la afirmación actual sin respaldo.
- **FR-005**: Si el resultado real muestra un valor atípico en algún estado con muestra pequeña, el texto actualizado DEBE matizar esa lectura como orientativa/no concluyente, con el mismo criterio ya aplicado a categorías de producto poco frecuentes.
- **FR-006**: NB2 DEBE calcular y persistir en `model_card.json` el número real de rondas de boosting entrenadas por el modelo final, en vez de requerir inspección manual del objeto serializado para obtenerlo.
- **FR-007**: El Informe Técnico (§3.5) y la Presentación (slide 18) DEBEN citar el valor real de rondas de boosting entrenadas (persistido en `model_card.json`) en lugar del hiperparámetro `max_iter=150` del espacio de búsqueda.
- **FR-008**: Ninguna métrica de desempeño ya reportada y validada (PR-AUC, ROC-AUC, Brier score, lift del decil superior, hiperparámetros de búsqueda, umbral de decisión) DEBE cambiar de valor como resultado de este trabajo — el cambio es aditivo (una celda de análisis nueva + un campo nuevo en `model_card.json`), no una re-optimización del modelo.
- **FR-009**: `model_card.json` DEBE seguir siendo la única fuente de verdad citada al reportar tanto las rondas de boosting reales como el desempeño por `customer_state` en los documentos finales, consistente con la práctica ya establecida para PR-AUC/ROC-AUC/Brier/lift.

### Key Entities

- **Tabla de desempeño por `customer_state`**: una fila por estado (27 posibles), con columnas `n` (tamaño de muestra en test), `tasa_real` (tasa de recompra observada), `tasa_predicha` (tasa de recompra predicha por el modelo), `average_precision` (PR-AUC del subgrupo, o no disponible si el subgrupo tiene una sola clase). Vive en NB2 §5.2, junto a la tabla ya existente por `main_category_grouped`.
- **Campo de rondas de boosting reales en `model_card.json`**: un nuevo campo numérico (junto a `hyperparameters.max_iter` ya existente) que registra cuántas rondas de boosting entrenó realmente el modelo final, distinto del techo `max_iter` del espacio de búsqueda.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las menciones a "disparidades por customer_state/estado" en el Informe y la Presentación están respaldadas por un hallazgo trazable a la tabla de subgrupo real de NB2 §5.2, verificable por inspección directa.
- **SC-002**: El 100% de las menciones a la cantidad de iteraciones/rondas de boosting del modelo en el Informe y la Presentación coinciden exactamente con el valor persistido en `model_card.json`.
- **SC-003**: Ninguna métrica de desempeño global ya reportada (PR-AUC, ROC-AUC, Brier score, lift) cambia de valor entre el `model_card.json` anterior y el regenerado tras este trabajo.
- **SC-004**: El cambio se inserta dentro de las secciones/slides ya existentes (§3.4, §3.5, §4.1 del Informe; slides 16, 18, 19 de la Presentación), sin crear secciones o slides nuevas.
- **SC-005**: Si algún estado con muestra pequeña muestra desempeño atípico, esa salvedad queda documentada explícitamente en el texto actualizado, verificable por inspección directa.

## Assumptions

- El `groupby` por `customer_state` se agrega en la misma celda o inmediatamente después de la celda existente de `subgroup_perf` en NB2 §5.2, reutilizando `raw_test` (que ya tiene `score` y `y_pred` calculados ahí mismo).
- No se requiere ninguna librería ni cambio de entorno adicional — pandas/scikit-learn ya se usan para el análisis de subgrupo existente por categoría.
- La re-ejecución de NB2 se hace con Run All completo (Principio II de la constitución del proyecto) para mantener reproducibilidad y consistencia entre todos los artifacts derivados.
- No existe en el proyecto un umbral numérico fijo para "muestra pequeña" en el análisis de subgrupo; se aplica el mismo criterio informal ya usado implícitamente para categorías de producto poco frecuentes.
- Este trabajo no reabre el Principio IV de la constitución (decisiones de modelado ya validadas): target, features, hiperparámetros buscados y umbral de decisión permanecen sin cambios; solo se agrega una celda de análisis descriptivo adicional y se persiste un valor ya calculable (rondas de boosting reales) que hoy no se guarda.
