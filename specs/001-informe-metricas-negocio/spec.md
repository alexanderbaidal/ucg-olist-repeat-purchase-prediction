# Feature Specification: Interpretación de negocio para ROC-AUC y Brier score

**Feature Branch**: `001-informe-metricas-negocio`

**Created**: 2026-09-18

**Status**: Draft

**Input**: User description: "Cerrar los gaps de traducción a negocio identificados en el Informe Técnico (docs/Informe_Tecnico_ShopMart_Olist.docx) y la Presentación de Defensa (docs/Defensa_ShopMart_Olist.pptx): (1) ROC-AUC (0.6308) se reporta en la tabla de métricas del Informe §3.2 pero nunca se traduce a una interpretación en lenguaje de negocio, a diferencia de PR-AUC que sí tiene su explicación (\"¿Qué significa un PR-AUC de 0.059?\" en la slide 11 y el Informe §3.4); (2) el Brier score (0.0317) aparece solo en la tabla de métricas del Informe §3.2 sin ninguna frase que explique qué mide o si es bueno/malo para un lector no técnico. El objetivo es agregar, tanto en el Informe como en la Presentación, una interpretación de negocio para ambas métricas siguiendo el mismo patrón ya usado para PR-AUC (métrica → qué significa → qué implica para ShopMart), sin alterar ninguna cifra ni reabrir el modelado."

## Clarifications

### Session 2026-09-18

- Q: ¿Qué formato deben usar las nuevas interpretaciones de negocio (ROC-AUC y Brier score) en cada documento? → A: cada documento replica su propio patrón ya existente — callout Q&A explícito ("¿Qué significa un ROC-AUC de 0.63?") en la Presentación, prosa narrativa integrada (sin subtítulo propio) en el Informe.
- Q: ¿El callout de Brier score en la Presentación debe citar la cifra exacta (0.0317), o alcanza con una frase cualitativa sin repetir el número? → A: cita la cifra exacta (0.0317), igual que PR-AUC y ROC-AUC en esa misma slide.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Interpretar el ROC-AUC reportado (Priority: P1)

Un lector no técnico del Informe Técnico o de la Presentación de Defensa (por ejemplo, un miembro del comité de defensa sin formación en ciencia de datos, o un perfil de negocio en ShopMart) llega a la cifra de ROC-AUC (0.6308 en test) y necesita entender, sin consultar a un experto, qué tan bueno es ese resultado y qué implica para la decisión de negocio.

**Why this priority**: el ROC-AUC aparece junto al PR-AUC en la tabla de resultados y en la slide de resumen de resultados (slide 11) — dejarlo sin interpretar al lado de una métrica que sí está explicada genera una asimetría notoria y da la impresión de que la métrica "no importa" o fue omitida por descuido.

**Independent Test**: se puede verificar leyendo únicamente el Informe §3.2/§3.4 (o la slide 11 de la Presentación), sin abrir los notebooks ni el `model_card.json`, y confirmando que el lector puede enunciar en una frase qué significa el ROC-AUC reportado y si es un resultado razonable.

**Acceptance Scenarios**:

1. **Given** un lector llega a la tabla de métricas del Informe §3.2, **When** lee la fila de ROC-AUC, **Then** encuentra, en esa sección o inmediatamente después (§3.4), una frase en lenguaje de negocio que explica qué mide el ROC-AUC y cómo calificar el valor reportado, con el mismo nivel de detalle que la interpretación ya existente de PR-AUC.
2. **Given** un lector revisa la slide de resultados de la Presentación (slide 11), **When** ve las métricas listadas, **Then** encuentra una interpretación de negocio del ROC-AUC comparable en tono y extensión a la que ya existe ahí para PR-AUC y el lift.

---

### User Story 2 - Interpretar el Brier score reportado (Priority: P2)

El mismo lector no técnico llega al Brier score (0.0317 en test), un término que no reconoce, y necesita entender qué mide y si es un buen resultado sin tener que buscar una definición externa.

**Why this priority**: el Brier score es el término más desconocido de la tabla de métricas para una audiencia no técnica; su ausencia de interpretación es el gap más agudo, pero es un requerimiento secundario frente al ROC-AUC porque tiene menor visibilidad (solo aparece en la tabla del Informe, no en el cuerpo de ninguna slide todavía).

**Independent Test**: se puede verificar leyendo el Informe/Presentación y confirmando que el lector puede explicar en una frase qué comunica el Brier score reportado, sin consultar fuentes externas.

**Acceptance Scenarios**:

1. **Given** un lector llega a la fila de "Otras métricas" (Brier score) en la tabla del Informe §3.2, **When** la lee, **Then** encuentra una frase que traduce ese número a una interpretación de negocio (qué tan bien calibradas están las probabilidades que produce el modelo) y si el valor reportado es favorable.
2. **Given** un lector revisa la slide de resultados de la Presentación (slide 11) o una adyacente, **When** llega al callout de Brier score, **Then** encuentra la misma interpretación de negocio que en el Informe, con la cifra exacta (0.0317), sin contradicciones entre ambos documentos.

---

### Edge Cases

- Si la interpretación de negocio de una métrica sugiere una conclusión más optimista que la que ya dan PR-AUC y el lift (p. ej. un ROC-AUC de 0.63 podría leerse como "aceptable" en aislamiento), la redacción nueva debe mantenerse consistente con el framing general ya establecido en el documento ("señal predictiva modesta pero accionable"), sin sobrevender el desempeño del modelo.
- Si agregar la interpretación del Brier score obliga a introducir jerga adicional (p. ej. "calibración de probabilidad"), esa jerga debe acompañarse de su propia traducción en la misma frase, siguiendo el patrón ya usado para PR-AUC.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El Informe Técnico DEBE incluir, en la tabla de métricas de §3.2 o inmediatamente después en §3.4, una interpretación en lenguaje de negocio del ROC-AUC de test reportado (0.6308), integrada como prosa narrativa (sin subtítulo propio), igual que el patrón ya usado para PR-AUC en esa sección.
- **FR-002**: El Informe Técnico DEBE incluir, con el mismo formato de prosa narrativa, una interpretación en lenguaje de negocio del Brier score de test reportado (0.0317), explicando qué mide (calidad/calibración de las probabilidades predichas) y si el valor es favorable.
- **FR-003**: La Presentación de Defensa DEBE incluir, en la slide de resultados (slide 11) o una slide adyacente, un callout Q&A explícito para ROC-AUC (p. ej. "¿Qué significa un ROC-AUC de 0.63?") y para Brier score (p. ej. "¿Qué significa un Brier score de 0.0317?"), cada uno citando la cifra exacta del test y replicando el mismo patrón visual ya usado ahí para PR-AUC ("¿Qué significa un PR-AUC de 0.059?").
- **FR-004**: Ninguna cifra ya reportada (PR-AUC, ROC-AUC, Brier score, lift, tamaños de muestra, hiperparámetros) DEBE modificarse como parte de este trabajo — el cambio es exclusivamente narrativo/interpretativo.
- **FR-005**: Ningún artifact bajo `notebooks/artifacts/` (incluido `model_card.json`) ni los notebooks DEBEN regenerarse, reentrenarse o editarse como parte de este trabajo — es una actualización de documentación final, no de modelado.
- **FR-006**: El lenguaje agregado DEBE mantener el mismo nivel de honestidad/matiz ya presente en el resto de ambos documentos (evitar sobrevender el desempeño del modelo; conservar el framing de "señal modesta pero accionable" ya usado en las conclusiones).
- **FR-007**: La interpretación agregada para cada métrica DEBE ser comprensible por un lector no técnico sin necesitar consultar los notebooks, el `model_card.json` o fuentes externas.
- **FR-008**: La interpretación de ROC-AUC y Brier score DEBE ser idéntica en sustancia entre el Informe y la Presentación (mismo mensaje, adaptado solo en extensión al formato de cada documento).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las apariciones del valor de ROC-AUC en el Informe y en la Presentación están acompañadas de una frase interpretativa en lenguaje de negocio, verificable por inspección directa de ambos documentos.
- **SC-002**: El 100% de las apariciones del valor de Brier score en el Informe y en la Presentación están acompañadas de una frase interpretativa en lenguaje de negocio, verificable por inspección directa de ambos documentos.
- **SC-003**: La Presentación y el Informe transmiten la misma interpretación de negocio para ambas métricas, sin contradicciones detectables entre los dos documentos.
- **SC-004**: El 100% de las cifras numéricas relacionadas con métricas del modelo que aparecen en el Informe y la Presentación después del cambio siguen coincidiendo exactamente con `model_card.json`.
- **SC-005**: El cambio se inserta dentro de la estructura de secciones/slides ya existente (no se crean secciones o slides nuevas fuera de las previstas por la rúbrica del caso).

## Assumptions

- El "lector no técnico" objetivo es equivalente al perfil de un miembro no especializado del comité de defensa, el mismo público ya asumido para la interpretación existente de PR-AUC.
- La redacción se realiza en español, consistente con el resto de ambos documentos.
- No se requiere generar ni modificar figuras o tablas — el trabajo es agregar texto interpretativo junto a datos ya existentes.
- El término técnico ("ROC-AUC", "Brier score") se conserva en el texto (no se elimina ni se reemplaza), acompañado de su traducción a lenguaje de negocio, igual que se hizo con PR-AUC.
- Este trabajo no altera ninguna decisión de modelado ya validada (Principio IV de la constitución del proyecto) ni requiere re-ejecutar NB1/NB2.
