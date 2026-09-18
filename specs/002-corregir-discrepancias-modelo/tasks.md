---

description: "Task list for Corregir discrepancias Informe/Presentación vs. código real"
---

# Tasks: Corregir discrepancias Informe/Presentación vs. código real

**Input**: Design documents from `specs/002-corregir-discrepancias-modelo/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, quickstart.md

**Tests**: no se generan tareas de test automatizado — este repo no tiene framework de tests
(Principio I de la constitución) y el spec no lo solicitó. La validación es manual, vía `quickstart.md`
y comparación directa de `model_card.json` antes/después.

**Organization**: las tareas están agrupadas por historia de usuario (US1 = auditoría real de
`customer_state`, US2 = corrección de la cifra de iteraciones), sobre una base Foundational compartida
que edita y re-ejecuta NB2 una sola vez para ambas.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: se puede ejecutar en paralelo (archivos distintos, sin dependencias pendientes)
- **[Story]**: a qué historia de usuario pertenece (US1, US2)
- Cada tarea incluye la ruta de archivo exacta

## ⚠️ Notas sobre paralelismo

1. **NB2 se edita y se re-ejecuta una sola vez para ambas historias** (Foundational): agregar dos
   celdas/campos y correr Run All dos veces sería redundante y arriesgaría que una ejecución no
   refleje el estado final del código. Por eso el código de NB2 para US1 y US2 vive en la fase
   Foundational, no dentro de cada historia.
2. **US1 y US2 editan los mismos dos archivos de `docs/`** (`.docx`/`.pptx`, binarios, sin fusión por
   diff) — igual que en `specs/001-informe-metricas-negocio`, US2 debe editar cada archivo **después**
   de que US1 termine de editarlo, no en paralelo.

---

## Phase 1: Setup

**Purpose**: confirmar el estado previo (para poder verificar SC-003 después) y la herramienta de
edición

- [X] T001 Registrar los valores actuales de `notebooks/artifacts/model_card.json` (PR-AUC, ROC-AUC,
      Brier score, lift de test) como línea base para comparar después de re-ejecutar NB2 (SC-003).
- [X] T002 [P] Confirmar que `python-docx` y `python-pptx` siguen disponibles en el entorno de trabajo
      (ya instalados como herramienta efímera durante `specs/001-informe-metricas-negocio` — no se
      reinstalan ni se agregan a `requirements.txt`).

**Checkpoint**: línea base registrada, herramienta de edición confirmada.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: aplicar ambos cambios de código a NB2 y re-ejecutarlo una sola vez, dejando los artifacts
regenerados listos para que ambas historias de usuario puedan citarlos

**⚠️ CRITICAL**: ninguna historia de usuario debe editar `docs/` sin que esta fase esté completa y
validada (SC-003)

- [X] T003 Respaldar `notebooks/NB2_Modelado_Validacion.ipynb` (copia sin editar en el scratchpad de
      la sesión) antes de cualquier edición.
- [X] T004 Agregar la celda de análisis de subgrupo por `customer_state` en
      `notebooks/NB2_Modelado_Validacion.ipynb` §5.2, inmediatamente después de la celda existente de
      `subgroup_perf` (mismo patrón, ver `research.md` §1).
- [X] T005 Agregar el campo `n_iter_actual` (`final_model.n_iter_`) al diccionario `model_card` de
      `notebooks/NB2_Modelado_Validacion.ipynb` §7, sin modificar `hyperparameters.max_iter` (ver
      `research.md` §2). Depende de T004 (mismo archivo — edición secuencial).
- [X] T006 Ejecutar `notebooks/NB2_Modelado_Validacion.ipynb` completo (Run All, cwd=`notebooks/`) para
      regenerar `notebooks/artifacts/model_card.json` y el resto de artifacts derivados. Depende de
      T005. **Resultado**: 47 celdas, 0 errores.
- [X] T007 Verificar SC-003: comparar PR-AUC/ROC-AUC/Brier/lift de test del `model_card.json`
      regenerado contra la línea base de T001 — deben ser idénticos. Si difieren, detener y
      diagnosticar antes de continuar (algo tocó el modelado). Depende de T006. **Resultado**: ROC-AUC,
      Brier y lift bit-a-bit idénticos; PR-AUC difiere solo en el último dígito de precisión float64
      (0.05901339211714865 vs. ...864 — ruido de punto flotante de la paralelización, idéntico a 4+
      decimales). `n_iter_actual=62` persistido correctamente.

**Checkpoint**: NB2 re-ejecutado, métricas globales verificadas sin drift — ambas historias de usuario
pueden empezar.

---

## Phase 3: User Story 1 - Auditoría real de desempeño por customer_state (Priority: P1) 🎯 MVP

**Goal**: la afirmación sobre ausencia de disparidades por estado en el Informe y la Presentación
queda respaldada por la tabla real de `notebooks/artifacts/model_card.json`/NB2 §5.2.

**Independent Test**: ejecutar NB2 completo y verificar que produce la tabla de subgrupo por
`customer_state`; confirmar que el Informe §3.4/§4.1 y las slides correspondientes de la Presentación
citan ese hallazgo real, no una afirmación genérica.

### Implementation for User Story 1

- [X] T008 [US1] Leer la tabla `subgroup_perf_state` en la salida de la celda de NB2 §5.2 (tras T006) y
      extraer: rango de `avg_precision` observado, estado con mejor/peor desempeño, y cuáles estados
      (si alguno) tienen `n` muy pequeño o `avg_precision` no disponible (candidatos ya identificados:
      `RR`, `AP`, `AC`, `AM`). **Resultado real**: suma de `n` = 12,643 (100% del test). 3 estados (SE,
      AM, RR) sin ningún caso positivo en test → `avg_precision` no disponible. De los 24 estados
      restantes, rango 0.025 (RN) a 0.333 (AP, n=9 — ruido de muestra minúscula); rango "robusto"
      (n≥15) 0.025 (RN) a 0.208 (PI). Los 5 estados de mayor volumen (SP 42% del test, RJ, MG, RS, PR)
      tienen `avg_precision` entre 0.033 y 0.097, en el mismo orden de magnitud que el PR-AUC global de
      test (0.059) — sin señal de disparidad grande en los estados con más peso.
- [X] T009 [P] [US1] Actualizar `docs/Informe_Tecnico_ShopMart_Olist.docx` §3.4 y/o §4.1: reemplazar la
      afirmación actual sin respaldo por el hallazgo real de T008, incluyendo la salvedad de los
      estados con muestra pequeña si aplica (FR-005). **Bloqueada temporalmente** porque el archivo
      estaba abierto en Word (el usuario lo cerró y se confirmó liberado antes de escribir).
- [X] T010 [P] [US1] Actualizar `docs/Defensa_ShopMart_Olist.pptx`: localizar por contenido (no por
      nombre de archivo, ver `research.md` §3) la slide de "desempeño por subgrupo" y la slide de
      ética/sesgos, y reemplazar la afirmación por el mismo hallazgo real de T008.
- [X] T011 [US1] Validar T009/T010 contra `specs/002-corregir-discrepancias-modelo/quickstart.md`
      (Pasos 2–4) y contra SC-001/SC-005 de `spec.md`. Depende de T009 y T010. **Resultado**: 7
      menciones de `customer_state` en el Informe, cifras 0.025/0.208 presentes en ambos documentos,
      salvedad de los 3 estados sin casos y de las muestras chicas presente en las 3 ubicaciones.

**Checkpoint**: la sección de ética del Informe y las slides correspondientes citan un hallazgo real
sobre `customer_state` — historia demostrable de forma independiente (MVP de este feature).

---

## Phase 4: User Story 2 - Corrección de la cifra de iteraciones del modelo (Priority: P2)

**Goal**: la cifra de "iteraciones del modelo" en el Informe y la Presentación coincide con
`n_iter_actual` (rondas de boosting reales), no con `max_iter=150`.

**Independent Test**: leer Informe §3.5 y la slide de prototipo/despliegue de la Presentación y
confirmar que la cifra citada coincide exactamente con `n_iter_actual` de `model_card.json`.

### Implementation for User Story 2

- [X] T012 [US2] Leer `n_iter_actual` de `notebooks/artifacts/model_card.json` (tras T006) y confirmar
      que es `<= hyperparameters.max_iter`. **Resultado**: `n_iter_actual=62 <= max_iter=150` ✓.
- [X] T013 [P] [US2] Actualizar `docs/Informe_Tecnico_ShopMart_Olist.docx` §3.5: reemplazar "150
      iteraciones" por el valor real de T012. Depende de T009 (mismo archivo — ejecutar después de que
      US1 termine de editar el Informe).
- [X] T014 [P] [US2] Actualizar `docs/Defensa_ShopMart_Olist.pptx`: localizar por contenido la slide de
      "Prototipo y Despliegue" y reemplazar "150 iteraciones" por el mismo valor real de T012. Depende
      de T010 (mismo archivo — ejecutar después de que US1 termine de editar la Presentación).
- [X] T015 [US2] Validar T013/T014 contra `quickstart.md` (Pasos 1 y 3) y contra SC-002 de `spec.md`:
      la cifra coincide exactamente entre `model_card.json`, el Informe y la Presentación. Depende de
      T013 y T014. **Resultado**: "62 rondas" presente en el Informe §3.5, "62 rondas de boosting
      entrenadas, techo buscado de 150" en la Presentación — mismo valor, mismo mensaje.

**Checkpoint**: ambas discrepancias corregidas con evidencia real — todas las historias de usuario
completas.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: validación final de extremo a extremo

- [X] T016 [P] Ejecutar el checklist completo de `specs/002-corregir-discrepancias-modelo/quickstart.md`
      (Pasos 1–4) sobre NB2 y ambos documentos juntos, confirmando SC-001 a SC-005 de `spec.md`.
      **Resultado**: SC-001 a SC-005 confirmados (ver T011/T015 y verificación de headings/slides).
- [ ] T017 Revisión visual humana en Word/PowerPoint (o LibreOffice) de ambos documentos — mismo
      recordatorio que en `specs/001-informe-metricas-negocio/tasks.md`: si el entorno de ejecución no
      tiene Office/LibreOffice disponible, esta verificación queda pendiente del usuario. **Pendiente**:
      esta sesión no tiene acceso de GUI a Office (Word se cerró vía terminación de proceso, no se
      renderizó visualmente) — la revisión visual real queda pendiente del usuario.
- [X] T018 Confirmar con `git diff --stat` que los únicos archivos modificados son
      `notebooks/NB2_Modelado_Validacion.ipynb`, los artifacts bajo `notebooks/artifacts/` que NB2
      regenera, `docs/Informe_Tecnico_ShopMart_Olist.docx` y `docs/Defensa_ShopMart_Olist.pptx` — que
      `notebooks/NB1_Estrategia_Datos_EDA.ipynb` y `requirements.txt` NO cambiaron (FR-003, FR-008).
      **Resultado**: `git status` confirma exactamente ese conjunto de archivos (más
      `customer_segments.parquet`, `repeat_purchase_model.joblib` y `test_predictions.parquet` — los
      otros artifacts que NB2 regenera junto con `model_card.json`; contenido semánticamente idéntico
      al anterior, verificado por separado). NB1 y `requirements.txt` sin cambios.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup — bloquea ambas historias de usuario. Incluye el único
  Run All de NB2 para todo el feature.
- **User Story 1 (Phase 3)**: depende de Foundational. Primera en editar cada archivo de `docs/`.
- **User Story 2 (Phase 4)**: depende de Foundational **y** de que US1 termine de editar cada archivo
  de `docs/` (T013←T009, T014←T010) — ver nota de paralelismo al inicio del documento.
- **Polish (Phase 5)**: depende de que ambas historias estén completas.

### Parallel Opportunities

- T001 y T002 (Setup) en paralelo.
- T004 y T005 (Foundational) son sobre el mismo archivo — secuenciales, no paralelos.
- T009 y T010 (US1) en paralelo (archivos distintos: `.docx` vs `.pptx`).
- T013 y T014 (US2) en paralelo entre sí, pero solo después de que T009/T010 hayan terminado
  respectivamente.
- T016 y T017 en paralelo (una es automatizable, la otra manual).

---

## Parallel Example: User Story 1

```bash
# T009 y T010 pueden ejecutarse a la vez — tocan archivos distintos:
Task: "Actualizar docs/Informe_Tecnico_ShopMart_Olist.docx §3.4/§4.1 con el hallazgo real de customer_state"
Task: "Actualizar docs/Defensa_ShopMart_Olist.pptx (slides de subgrupo y ética) con el mismo hallazgo"
```

---

## Implementation Strategy

### MVP First (Setup + Foundational + User Story 1)

1. Completar Phase 1: Setup (línea base de métricas).
2. Completar Phase 2: Foundational (código de NB2 para ambas historias + Run All + verificación de
   no-drift).
3. Completar Phase 3: User Story 1 (customer_state).
4. **Validar** con T011: la discrepancia de mayor severidad (afirmación de fairness sin respaldo en la
   sección de ética) queda cerrada con evidencia real, de forma independiente y verificable.

### Incremental Delivery

1. Setup + Foundational → NB2 re-ejecutado, métricas sin drift confirmado.
2. Agregar User Story 1 (customer_state) → validar con T011 → MVP entregable.
3. Agregar User Story 2 (iteraciones) → validar con T015 → feature completo.
4. Phase 5 (Polish) → validación cruzada final.

---

## Notes

- [P] = archivos distintos, sin dependencias pendientes.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad.
- No hay tareas de test automatizado (repo sin framework de tests, por diseño — Principio I de la
  constitución); toda verificación es manual vía `quickstart.md`.
- T007 es un gate de seguridad: si las métricas globales cambiaron tras el Run All, el feature se
  detiene ahí — no se editan los documentos con datos posiblemente inválidos.
- Confirmar en T018 que este feature no dejó rastro fuera de NB2, sus artifacts regenerados, y los dos
  documentos de `docs/` — ni en `requirements.txt`, ni en NB1.
