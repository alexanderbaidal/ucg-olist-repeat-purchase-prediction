---

description: "Task list for Corregir split train/val/test no temporal en NB1"
---

# Tasks: Corregir split train/val/test no temporal en NB1

**Input**: Design documents from `specs/003-corregir-split-temporal/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, quickstart.md

**Tests**: no se generan tareas de test automatizado — este repo no tiene framework de tests
(Principio I de la constitución) y el spec no lo solicitó. La validación es manual, vía
`quickstart.md` y comparación directa de artifacts/documentos antes y después.

**Organization**: las tareas están agrupadas por historia de usuario (US1 = el split es realmente
temporal, US2 = artifacts y métricas aguas abajo sincronizados, US3 = la narrativa del mecanismo es
precisa), sobre una base Foundational compartida de respaldo/localización de la celda a editar.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: se puede ejecutar en paralelo (archivos distintos, sin dependencias pendientes)
- **[Story]**: a qué historia de usuario pertenece (US1, US2, US3)
- Cada tarea incluye la ruta de archivo exacta

## ⚠️ Notas sobre paralelismo y secuencia

1. **US1 debe completarse y verificarse antes de que arranque US2**: US2 depende de los artifacts que
   NB1 regenera bajo el mecanismo corregido (no tiene sentido re-ejecutar NB2 sobre una partición que
   todavía no pasó la verificación de US1).
2. **US3 (narrativa) puede ejecutarse en paralelo con US2 una vez que el código de US1 está fijado**:
   describir el mecanismo real (NB1 §6 markdown, Informe §3.2, Presentación slide 12) no depende de las
   métricas recalculadas de NB2 — solo depende de que la implementación de T006 ya sea definitiva.
3. **US2 y US3 editan los mismos dos archivos de `docs/`** (`.docx`/`.pptx`) en secciones/slides
   distintas — para evitar conflictos de edición sobre el mismo archivo binario, cada archivo se edita
   en una sola pasada por historia (no se abre y guarda el mismo `.docx` dos veces en paralelo); ver
   dependencias explícitas T020←T015 y T021←T016.

---

## Phase 1: Setup

**Purpose**: capturar el estado previo (para poder verificar SC-002/SC-003/SC-004 después) y confirmar
las herramientas de edición

- [X] T001 Registrar los valores actuales de `notebooks/artifacts/model_card.json` completo (copia en
      el scratchpad de la sesión) como línea base para comparar después de re-ejecutar NB1+NB2 (SC-003).
- [X] T002 [P] Registrar el resumen agregado actual de `notebooks/artifacts/customer_segments.parquet`
      (`groupby("segment_label")` con n/recencia/frecuencia/monto medios) como línea base para verificar
      que no cambia (SC-004, FR-007).
- [X] T003 [P] Registrar los campos actuales de `notebooks/artifacts/eda_summary.json` (todo excepto
      `splits`) como línea base para verificar SC-006 (target, features, censura de 90 días,
      `base_table_hash` sin cambios).
- [X] T004 [P] Confirmar que `python-docx`/`python-pptx` siguen disponibles en el entorno de trabajo
      (ya usados como herramienta efímera en `specs/001-...` y `specs/002-...` — no se reinstalan ni se
      agregan a `requirements.txt`).

**Checkpoint**: líneas base registradas, herramientas de edición confirmadas.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: preparar la edición quirúrgica de NB1 antes de tocar código

**⚠️ CRITICAL**: ninguna historia de usuario debe editar NB1 sin que esta fase esté completa

- [X] T005 Respaldar `notebooks/NB1_Estrategia_Datos_EDA.ipynb` (copia sin editar en el scratchpad de
      la sesión) antes de cualquier edición.
- [X] T006 Localizar por contenido, en `notebooks/NB1_Estrategia_Datos_EDA.ipynb`, la celda de código
      de partición (con las dos llamadas a `train_test_split`, sección "6. Particiones y persistencia
      de artifacts") y su celda markdown adyacente — confirmar los índices reales de celda, no asumir
      los citados en esta conversación (mismo criterio que `research.md` §5 para las slides).

**Checkpoint**: NB1 respaldado, celdas objetivo localizadas con certeza — la edición de US1 puede
empezar.

---

## Phase 3: User Story 1 - El conjunto de test refleja realmente a los clientes más recientes (Priority: P1) 🎯 MVP

**Goal**: el mecanismo de partición de NB1 cumple lo que ya afirma su propia documentación — test
compuesto, en su mayoría, por los clientes cronológicamente más recientes — sin depender de un efecto
secundario que el `shuffle` por defecto de `train_test_split` descarta.

**Independent Test**: ejecutar NB1 completo (Run All) e inspeccionar `order_purchase_timestamp` en
`raw_train/val/test.parquet` regenerados — el percentil 90 de train debe ser anterior o igual al
percentil 10 de test (hoy los tres splits cubren el mismo rango de fechas completo).

### Implementation for User Story 1

- [X] T007 [US1] En `notebooks/NB1_Estrategia_Datos_EDA.ipynb` (celda de código de la sección 6
      localizada en T006), reemplazar las dos llamadas a `train_test_split` por una función
      `temporal_split_by_class` (pandas `groupby(TARGET)` + `sort_values([ts_col, "order_id"],
      kind="mergesort")` + `iloc` para tomar 70%/15%/15% de cada clase por separado, luego concatenar y
      reordenar por timestamp) — ver `research.md` §2/§3 para el código de referencia exacto y la
      justificación del tiebreaker por `order_id`.
- [X] T008 [US1] Ejecutar `notebooks/NB1_Estrategia_Datos_EDA.ipynb` completo (Run All, cwd=`notebooks/`)
      para regenerar `notebooks/artifacts/X_train.parquet` (y `X_val`/`X_test`/`y_train`/`y_val`/
      `y_test`), `notebooks/artifacts/raw_train.parquet` (y `raw_val`/`raw_test`),
      `notebooks/artifacts/preprocessor.joblib` y `notebooks/artifacts/eda_summary.json`. Depende de
      T007.
- [X] T009 [US1] Verificar FR-001/SC-001 sobre `notebooks/artifacts/raw_train.parquet` y
      `notebooks/artifacts/raw_test.parquet` regenerados: percentil 90 de `order_purchase_timestamp` en
      train ≤ percentil 10 en test. Depende de T008. Si falla, revisar la implementación de T007 antes
      de continuar.
- [X] T010 [US1] Verificar FR-002/SC-002 sobre `notebooks/artifacts/raw_train.parquet`,
      `raw_val.parquet` y `raw_test.parquet` regenerados: la tasa de recompra (`repeat_purchase.mean()`)
      de cada uno está dentro de ±0.5 puntos porcentuales del ~3.3% documentado en
      `notebooks/artifacts/eda_summary.json` (comparar contra la línea base de T003). Depende de T008.
- [X] T011 [US1] Verificar SC-006 sobre `notebooks/artifacts/eda_summary.json` regenerado: todos los
      campos excepto el bloque `splits` (n/rate por partición) son idénticos a la línea base de T003 —
      `n_customers_total`/`n_customers_eligible`, `class_balance` global, `numeric_features`/
      `categorical_features`, `censoring_cutoff`, `raw_table_hashes`, `base_table_hash`. Depende de T008.

**Checkpoint**: la partición de NB1 es verificablemente temporal y preserva el balance de clases,
independiente de NB2 y de `docs/` — MVP de este feature.

---

## Phase 4: User Story 2 - Los artifacts y las métricas aguas abajo reflejan la partición corregida (Priority: P2)

**Goal**: toda cifra de desempeño citada en el Informe y la Presentación corresponde al modelo
entrenado sobre la partición corregida, no a la partición aleatoria anterior.

**Independent Test**: ejecutar NB1 → NB2 de punta a punta (Run All en ambos) y confirmar que todos los
artifacts en `notebooks/artifacts/` quedan regenerados, y que `model_card.json` refleja las métricas
resultantes de la nueva partición.

### Implementation for User Story 2

- [X] T012 [US2] Ejecutar `notebooks/NB2_Modelado_Validacion.ipynb` completo (Run All, cwd=`notebooks/`)
      para regenerar `notebooks/artifacts/repeat_purchase_model.joblib`,
      `notebooks/artifacts/model_card.json` y `notebooks/artifacts/test_predictions.parquet` sobre los
      artifacts de NB1 ya corregidos. Depende de T009, T010 y T011 (US1 completa y verificada).
- [X] T013 [US2] Verificar FR-007/SC-004: el resumen agregado de `customer_segments.parquet`
      regenerado es idéntico a la línea base de T002 (`orders_history.parquet` no depende de la
      partición train/val/test). Depende de T012. Si difiere, detener e investigar antes de continuar.
- [X] T014 [US2] Diff de `model_card.json` regenerado contra la línea base de T001 — documentar
      exactamente qué campos cambiaron (hiperparámetros ganadores, `decision_threshold`, métricas de
      validación/test, `top_features_by_permutation_importance`, `n_train`/`n_val`/`n_test`). Depende de
      T012.
- [X] T015 [US2] Revisar la celda markdown "Resultado de la comparación..." en
      `notebooks/NB2_Modelado_Validacion.ipynb` §3 (cifras `0.0468 vs. 0.0455` hardcodeadas en prosa)
      contra la salida real de `comparison_df` tras el Run All; editarla si las cifras reales difieren
      (`data-model.md` instancia 2). Depende de T012.
- [X] T016 [P] [US2] Actualizar `docs/Informe_Tecnico_ShopMart_Olist.docx` §3.1/§3.3 (comparación de
      candidatos y tabla de métricas finales) y §3.4/§5.1 si aplica (importancia de variables, lift en
      conclusiones) con los valores reales de T014/T015 (`data-model.md` instancias 3–7). Depende de
      T014 y T015.
- [X] T017 [P] [US2] Actualizar `docs/Defensa_ShopMart_Olist.pptx`: localizar por contenido (no por
      nombre de archivo, ver `research.md` §5) las slides de baseline (5, 10), comparación de candidatos
      (11), métricas finales (13), impacto de negocio (14) e importancia de variables (15), y
      reemplazar cada cifra por el valor real de T014/T015, incluidas las notas de orador
      (`data-model.md` instancias 8–14). Depende de T014 y T015.
- [ ] T018 [US2] Regenerar/actualizar el PDF exportado del Informe
      (`docs/5 - Personalización Data-Driven en E-Commerce_ Potenciando la Retención de Clientes.pdf`)
      a partir del `.docx` actualizado en T016. Depende de T016. **Bloqueada**: este entorno de
      ejecución no tiene LibreOffice/Word disponible (`soffice` no está instalado; el shim del script
      de la skill de docx asume un socket Unix, no disponible en Windows nativo) — el `.docx` ya está
      actualizado y validado (ver T019), pero el PDF exportado queda desincronizado hasta que el
      usuario lo regenere manualmente (abrir el `.docx` en Word y exportar a PDF con el mismo nombre).
- [X] T019 [US2] Validar SC-003 con `quickstart.md` Pasos 3–5: el 100% de las cifras de desempeño
      citadas en el Informe y la Presentación coincide exactamente con `model_card.json` regenerado.
      Depende de T016, T017 y T018. **Resultado**: barrido completo de `docs/Informe_Tecnico_ShopMart_Olist.docx`
      y `docs/Defensa_ShopMart_Olist.pptx` (párrafos, tabla, slides y notas de orador) sin cifras
      obsoletas (0.059, 0.0468/0.0455, 0.6308, 2.27x, "62 rondas", "empate técnico"); las cifras nuevas
      (0.037, 0.0619/0.0507, 0.4627/0.1252, ≈1.00x, 118 rondas) presentes y consistentes en ambos
      documentos. El PDF exportado del Informe queda pendiente de T018 (bloqueada).

**Checkpoint**: Informe y Presentación citan exclusivamente números de la partición corregida — ninguna
cifra de la ejecución anterior sobrevive en ningún documento.

---

## Phase 5: User Story 3 - La narrativa de "validez temporal" describe con precisión el mecanismo real (Priority: P3)

**Goal**: NB1 §6, el Informe §3.2 y la Presentación slide 12 describen el mecanismo de partición real
(temporal dentro de cada clase) de forma consistente entre sí, sin afirmar un comportamiento que el
código no ejecuta.

**Independent Test**: leer la celda markdown de partición de NB1, el Informe §3.2 y la Presentación
(slide 12 + nota de orador) tras la corrección, y confirmar que los tres describen el mismo mecanismo
sin contradicción.

### Implementation for User Story 3

- [X] T020 [US3] Reescribir la celda markdown adyacente a la celda de partición de NB1 §6 (localizada en
      T006) para describir con precisión el mecanismo real de T007: orden temporal + preservación del
      balance de clases mediante partición independiente por clase, en vez de la descripción actual
      (que no distingue este matiz). No requiere re-ejecutar NB1 (edición de markdown, sin efecto en el
      cómputo). Depende de T007 (la implementación ya debe ser definitiva antes de describirla).
      **Resultado**: ejecutada junto con T007 en la misma edición de NB1 (celdas 55/56), antes del Run
      All — evita describir un mecanismo que luego cambiara.
- [X] T021 [P] [US3] Actualizar `docs/Informe_Tecnico_ShopMart_Olist.docx` §3.2 con la misma descripción
      de mecanismo de T020. Depende de T020 y de T016 (evitar reabrir el mismo `.docx` en paralelo con
      la edición de US2).
- [X] T022 [P] [US3] Actualizar `docs/Defensa_ShopMart_Olist.pptx` slide 12 (+ nota de orador) con la
      misma descripción de mecanismo de T020. Depende de T020 y de T017 (evitar reabrir el mismo
      `.pptx` en paralelo con la edición de US2).
- [X] T023 [US3] Validar SC-005: lectura comparativa de NB1 §6 (markdown), Informe §3.2 y Presentación
      slide 12 — mismo mecanismo descrito, sin contradicción entre los tres. Depende de T020, T021 y
      T022. **Resultado**: los tres describen "orden temporal dentro de cada clase, no un corte
      cronológico global" + preservación del ~3.3% + "test compuesto en su mayoría por los clientes más
      recientes" — mismo mecanismo, sin contradicción.

**Checkpoint**: las tres historias de usuario completas — el split es temporal, verificado, sincronizado
y descrito con precisión en todos los documentos.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: validación final de extremo a extremo

- [X] T024 [P] Ejecutar el checklist completo de `specs/003-corregir-split-temporal/quickstart.md`
      (Pasos 0–7) de punta a punta, confirmando SC-001 a SC-006 de `spec.md`. **Resultado**: SC-001,
      SC-002, SC-004, SC-005, SC-006 confirmados (ver T009-T011, T013, T023). SC-003 confirmado para
      `.docx`/`.pptx` (T019); el PDF exportado queda pendiente de T018 (bloqueada por falta de
      LibreOffice/Word en este entorno).
- [ ] T025 Revisión visual humana en Word/PowerPoint (o LibreOffice) de ambos documentos — si el entorno
      de ejecución no tiene Office/LibreOffice disponible, esta verificación queda pendiente del usuario
      (mismo criterio que en `specs/001-...` y `specs/002-...`). **Pendiente**: esta sesión no tiene
      acceso a Office/LibreOffice (`soffice` no instalado, el shim de la skill de docx no funciona en
      Windows nativo) — la validación estructural (XSD, conteo de párrafos/slides) pasó en ambos
      documentos vía `office/validate.py`, pero la revisión visual real queda pendiente del usuario.
- [X] T026 Confirmar con `git diff --stat` que los archivos modificados son exactamente:
      `notebooks/NB1_Estrategia_Datos_EDA.ipynb`, `notebooks/NB2_Modelado_Validacion.ipynb`, todos los
      artifacts de `notebooks/artifacts/` **excepto** `customer_segments.parquet` (que debe aparecer sin
      cambios — confirmar explícitamente, no solo por omisión), `docs/Informe_Tecnico_ShopMart_Olist.docx`,
      y `docs/Defensa_ShopMart_Olist.pptx`. `requirements.txt` y cualquier otro archivo fuera de esta
      lista NO deben aparecer en el diff. **Resultado**: `git diff --stat` confirma exactamente ese
      conjunto (18 archivos): NB1, NB2, `model_card.json`, `preprocessor.joblib`,
      `repeat_purchase_model.joblib`, `test_predictions.parquet`, `X/y_train/val/test.parquet`,
      `raw_train/val/test.parquet`, ambos documentos de `docs/` y `.specify/feature.json` (tracking de
      spec-kit). `customer_segments.parquet`, `orders_history.parquet`, el PDF del Informe y
      `requirements.txt` NO aparecen — sin cambios, como se esperaba.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciarse de inmediato.
- **Foundational (Phase 2)**: depende de Setup — bloquea la edición de NB1 en US1.
- **User Story 1 (Phase 3)**: depende de Foundational. Es el MVP — verificable de forma
  completamente independiente de NB2 y de `docs/`.
- **User Story 2 (Phase 4)**: depende de que US1 esté completa y verificada (T009–T011) — no tiene
  sentido re-ejecutar NB2 sobre una partición todavía no confirmada.
- **User Story 3 (Phase 5)**: depende de que la implementación de US1 (T007) sea definitiva; sus
  ediciones de `docs/` (T021, T022) se secuencian después de las de US2 (T016, T017) sobre los mismos
  archivos, pero el contenido en sí (descripción del mecanismo) no depende de las métricas de US2.
- **Polish (Phase 6)**: depende de que las tres historias estén completas.

### Parallel Opportunities

- T001, T002, T003, T004 (Setup) en paralelo.
- T016 y T017 (US2) en paralelo entre sí (archivos distintos: `.docx` vs `.pptx`), ambos después de
  T014/T015.
- T021 y T022 (US3) en paralelo entre sí (archivos distintos), cada uno después de su respectiva tarea
  de US2 sobre el mismo archivo (T021←T016, T022←T017).
- T024 y T025 (Polish) en paralelo (una es automatizable, la otra manual).

---

## Parallel Example: User Story 2

```bash
# T016 y T017 pueden ejecutarse a la vez — tocan archivos distintos:
Task: "Actualizar docs/Informe_Tecnico_ShopMart_Olist.docx §3.1/§3.3/§3.4/§5.1 con las métricas reales"
Task: "Actualizar docs/Defensa_ShopMart_Olist.pptx (slides 5,10,11,13,14,15) con las métricas reales"
```

---

## Implementation Strategy

### MVP First (Setup + Foundational + User Story 1)

1. Completar Phase 1: Setup (líneas base de `model_card.json`, `customer_segments.parquet`,
   `eda_summary.json`).
2. Completar Phase 2: Foundational (respaldo y localización de la celda de NB1).
3. Completar Phase 3: User Story 1 (mecanismo de partición corregido, verificado con T009–T011).
4. **Validar**: la discrepancia raíz — el split no era realmente temporal pese a estar documentado como
   tal — queda cerrada con evidencia empírica, de forma independiente de NB2 y de `docs/`.

### Incremental Delivery

1. Setup + Foundational → NB1 listo para editarse con seguridad.
2. Agregar User Story 1 (split corregido) → validar con T009–T011 → MVP entregable.
3. Agregar User Story 2 (re-ejecución en cascada + sync de métricas) → validar con T019 → feature
   funcionalmente completo.
4. Agregar User Story 3 (precisión narrativa) → validar con T023 → feature completo.
5. Phase 6 (Polish) → validación cruzada final.

---

## Notes

- [P] = archivos distintos, sin dependencias pendientes.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad.
- No hay tareas de test automatizado (repo sin framework de tests, por diseño — Principio I de la
  constitución); toda verificación es manual vía `quickstart.md`.
- T009/T010/T011 son gates de seguridad de US1: si el mecanismo no cumple orden temporal o balance de
  clases, el feature se detiene ahí — no se re-ejecuta NB2 ni se editan documentos sobre una partición
  inválida.
- T013 es el gate de seguridad equivalente en US2 para RFM: si `customer_segments.parquet` cambió,
  algo distinto de lo esperado tocó `orders_history.parquet` y hay que investigar antes de seguir.
- Confirmar en T026 que este feature no dejó rastro fuera de NB1, NB2 (si aplica), sus artifacts
  regenerados (excepto `customer_segments.parquet`, intacto) y los dos documentos de `docs/` — ni en
  `requirements.txt`.
