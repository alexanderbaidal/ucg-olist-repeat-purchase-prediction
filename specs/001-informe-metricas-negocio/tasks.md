---

description: "Task list for Interpretación de negocio para ROC-AUC y Brier score"
---

# Tasks: Interpretación de negocio para ROC-AUC y Brier score

**Input**: Design documents from `specs/001-informe-metricas-negocio/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, quickstart.md

**Tests**: no se generan tareas de test automatizado — este repo no tiene framework de tests
(Principio I de la constitución) y el spec no lo solicitó. La validación es manual, vía `quickstart.md`.

**Organization**: las tareas están agrupadas por historia de usuario (US1 = ROC-AUC, US2 = Brier
score) para que cada una sea entregable y verificable de forma independiente.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: se puede ejecutar en paralelo (archivos distintos, sin dependencias pendientes)
- **[Story]**: a qué historia de usuario pertenece (US1, US2)
- Cada tarea incluye la ruta de archivo exacta

## ⚠️ Nota sobre paralelismo entre historias

US1 y US2 son **lógicamente independientes** (cada una es verificable por separado, ver
`spec.md`), pero ambas editan los **mismos dos archivos binarios**
(`docs/Informe_Tecnico_ShopMart_Olist.docx`, `docs/Defensa_ShopMart_Olist.pptx`). Los archivos
`.docx`/`.pptx` no se pueden fusionar línea por línea como texto plano, así que — a diferencia de un
proyecto de código donde historias independientes pueden avanzar en paralelo por distintas personas —
aquí **US2 debe ejecutarse después de que US1 termine de editar cada archivo**, para no sobrescribir
esos cambios. Esta dependencia es de ejecución (evitar conflictos de edición), no de lógica de negocio.

---

## Phase 1: Setup

**Purpose**: confirmar las cifras fuente y preparar la herramienta de edición

- [X] T001 Confirmar `roc_auc` y `brier_score` de test leyendo `notebooks/artifacts/model_card.json`
      (Paso 1 de `specs/001-informe-metricas-negocio/quickstart.md`); anotar los valores exactos
      (0.6308.../0.0317...) que se van a citar en los pasos siguientes.
- [X] T002 [P] Preparar la herramienta de edición local: instalar `python-docx` y `python-pptx` en el
      entorno de trabajo (instalación efímera — NO agregar a `requirements.txt`, ver `research.md` §1).

**Checkpoint**: cifras confirmadas y herramienta de edición lista.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: respaldar los dos documentos antes de editarlos, dado que son archivos binarios sin
control de versiones por diff/merge

**⚠️ CRITICAL**: ninguna historia de usuario debe empezar a editar sin este respaldo

- [X] T003 [P] Respaldar `docs/Informe_Tecnico_ShopMart_Olist.docx` (copia sin editar en el
      scratchpad de la sesión) antes de cualquier edición.
- [X] T004 [P] Respaldar `docs/Defensa_ShopMart_Olist.pptx` (copia sin editar en el scratchpad de la
      sesión) antes de cualquier edición.

**Checkpoint**: respaldo listo — puede comenzar la edición de las historias de usuario.

---

## Phase 3: User Story 1 - Interpretar el ROC-AUC reportado (Priority: P1) 🎯 MVP

**Goal**: un lector no técnico que revise el Informe o la Presentación encuentra, junto a la cifra de
ROC-AUC, una interpretación de negocio (mismo tratamiento que ya tiene PR-AUC).

**Independent Test**: leer únicamente el Informe §3.2/§3.4 (o la slide 11 de la Presentación), sin
abrir notebooks ni `model_card.json`, y confirmar que se puede enunciar en una frase qué significa el
ROC-AUC reportado y si es razonable.

### Implementation for User Story 1

- [X] T005 [P] [US1] Insertar la interpretación de negocio del ROC-AUC como prosa narrativa (ancla
      "0.50 = azar", ver `research.md` §2) en `docs/Informe_Tecnico_ShopMart_Olist.docx`, §3.2 (tabla
      de métricas) o inmediatamente después en §3.4, con el mismo estilo tipográfico del párrafo/tabla
      circundante.
- [X] T006 [P] [US1] Insertar el callout "¿Qué significa un ROC-AUC de 0.63?" (mismo mensaje que T005,
      ver `research.md` §2) en `docs/Defensa_ShopMart_Olist.pptx`, slide 11 o una slide adyacente,
      replicando el patrón visual del callout ya existente de PR-AUC. **Nota de implementación**: la
      slide de resultados solo tenía ~1.1in libres — insuficiente para 2 bloques Q&A completos (4
      shapes) — así que ROC-AUC y Brier score comparten un único shape compacto de 2 párrafos (mismo
      estilo: Arial 12pt, color `#475569`, sin negrita), en vez de 2 shapes por métrica. Mensaje y
      cifra exacta preservados.
- [X] T007 [US1] Validar las dos inserciones de ROC-AUC (T005, T006) contra los Pasos 1–2 de
      `specs/001-informe-metricas-negocio/quickstart.md` y contra SC-001/SC-004 de `spec.md`: la cifra
      citada coincide exactamente con `model_card.json` y la frase interpretativa está presente en
      ambos documentos. Depende de T005 y T006.

**Checkpoint**: ROC-AUC interpretado y consistente en ambos documentos — historia demostrable de forma
independiente (MVP de este feature).

---

## Phase 4: User Story 2 - Interpretar el Brier score reportado (Priority: P2)

**Goal**: un lector no técnico que revise el Informe o la Presentación encuentra, junto a la cifra de
Brier score, una interpretación de negocio que incluye la salvedad de desbalance de clases (para no
sobrevender el resultado).

**Independent Test**: leer el Informe/Presentación y confirmar que se puede explicar en una frase qué
comunica el Brier score reportado — incluida la salvedad de que un valor bajo es esperable bajo el
desbalance de clases de este dataset — sin consultar fuentes externas.

### Implementation for User Story 2

- [X] T008 [P] [US2] Insertar la interpretación de negocio del Brier score como prosa narrativa
      (calibración de probabilidad + salvedad de desbalance de clases, ver `research.md` §3) en
      `docs/Informe_Tecnico_ShopMart_Olist.docx`, §3.2 o §3.4, con el mismo estilo tipográfico del
      párrafo/tabla circundante. Depende de T005 (mismo archivo — ejecutar después para no
      sobrescribirlo).
- [X] T009 [P] [US2] Insertar el callout "¿Qué significa un Brier score de 0.0317?" (mismo mensaje que
      T008, incluida la salvedad de desbalance) en `docs/Defensa_ShopMart_Olist.pptx`, slide 11 o una
      slide adyacente. Depende de T006 (mismo archivo — ejecutar después para no sobrescribirlo).
      **Ejecutada junto con T006**: por la restricción de espacio documentada ahí, el callout de Brier
      score es el segundo párrafo del mismo shape compacto agregado en T006, no un shape separado.
- [X] T010 [US2] Validar las dos inserciones de Brier score (T008, T009) contra los Pasos 1–2 de
      `specs/001-informe-metricas-negocio/quickstart.md` y contra SC-002/SC-004 de `spec.md`: la cifra
      citada coincide exactamente con `model_card.json`, la frase interpretativa está presente en
      ambos documentos, y la salvedad de desbalance de clases aparece en **ambos**, no solo en el
      Informe. Depende de T008 y T009.

**Checkpoint**: Brier score interpretado y consistente en ambos documentos, con la salvedad de
desbalance presente en los dos — todas las historias de usuario completas.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: validación final de extremo a extremo sobre ambos documentos juntos

- [X] T011 [P] Ejecutar el Paso 2 completo de `specs/001-informe-metricas-negocio/quickstart.md`
      (extracción de texto vía `zipfile`+regex) sobre `docs/Informe_Tecnico_ShopMart_Olist.docx` y
      `docs/Defensa_ShopMart_Olist.pptx` juntos, y confirmar SC-001 a SC-004 de `spec.md`. **Resultado**:
      ROC-AUC y Brier score aparecen con su cifra exacta e interpretación en ambos documentos; slide
      count (23) y secciones de primer nivel del Informe (6) sin cambios (SC-005).
- [ ] T012 Revisión visual humana en Word/PowerPoint (o LibreOffice) de ambos documentos, siguiendo el
      Paso 4 de `quickstart.md`: estilo tipográfico consistente y la salvedad de Brier score presente
      en los dos documentos. **Pendiente**: este entorno no tiene Word/PowerPoint/LibreOffice
      disponible para abrir los archivos; se verificó programáticamente que la fuente/tamaño/color de
      los runs nuevos coincide con los de los párrafos y shapes vecinos y que ningún shape nuevo se
      desborda de los límites de la slide, pero la revisión visual real queda pendiente del usuario.
- [X] T013 Confirmar con `git diff --stat` que solo cambiaron
      `docs/Informe_Tecnico_ShopMart_Olist.docx` y `docs/Defensa_ShopMart_Olist.pptx`, que
      `requirements.txt` no fue tocado, y que ningún archivo bajo `notebooks/` o
      `notebooks/artifacts/` cambió (FR-004, FR-005, SC-005 de `spec.md`). **Resultado**: `git status`
      confirma que solo esos 2 archivos de `docs/` están modificados (además de los artefactos nuevos
      de `specs/001-informe-metricas-negocio/` y `.specify/feature.json` propios de este flujo
      spec-kit) — `requirements.txt` y todo `notebooks/` sin cambios.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: depende de Setup — bloquea ambas historias de usuario.
- **User Story 1 (Phase 3)**: depende de Foundational. Es la primera en tocar cada archivo.
- **User Story 2 (Phase 4)**: depende de Foundational **y** de que US1 termine de editar cada archivo
  (T008 depende de T005; T009 depende de T006) — ver la nota de paralelismo al inicio del documento.
- **Polish (Phase 5)**: depende de que ambas historias de usuario estén completas.

### Dentro de cada historia

- T005/T006 (US1) son paralelizables entre sí (archivos distintos), pero T007 depende de ambas.
- T008/T009 (US2) son paralelizables entre sí (archivos distintos), pero cada una depende de su
  contraparte de US1 sobre el mismo archivo (T008←T005, T009←T006), y T010 depende de ambas.

### Parallel Opportunities

- T001 y T002 (Setup) en paralelo.
- T003 y T004 (Foundational) en paralelo.
- T005 y T006 (US1) en paralelo.
- T008 y T009 (US2) en paralelo — pero solo después de que T005/T006 hayan terminado respectivamente.
- T011 puede correr en paralelo a T012 (una es automatizable, la otra es revisión manual), ambas
  después de que T007 y T010 estén completas.

---

## Parallel Example: User Story 1

```bash
# T005 y T006 pueden ejecutarse a la vez — tocan archivos distintos:
Task: "Insertar interpretación de negocio del ROC-AUC en docs/Informe_Tecnico_ShopMart_Olist.docx"
Task: "Insertar callout de ROC-AUC en docs/Defensa_ShopMart_Olist.pptx"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup.
2. Completar Phase 2: Foundational (respaldo de ambos documentos).
3. Completar Phase 3: User Story 1 (ROC-AUC).
4. **Validar** con T007 y, si se quiere entregar solo esto, detenerse aquí: el gap de ROC-AUC —el más
   visible, porque aparece junto a PR-AUC en la misma tabla/slide— queda cerrado de forma
   independiente y verificable.

### Incremental Delivery

1. Setup + Foundational → documentos respaldados.
2. Agregar User Story 1 (ROC-AUC) → validar con T007 → MVP entregable.
3. Agregar User Story 2 (Brier score) → validar con T010 → feature completo.
4. Phase 5 (Polish) → validación cruzada final de ambos documentos juntos.

---

## Notes

- [P] = archivos distintos, sin dependencias pendientes.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad.
- No hay tareas de test automatizado (repo sin framework de tests, por diseño — Principio I de la
  constitución); toda verificación es manual vía `quickstart.md`.
- Confirmar en T013 que este feature no dejó rastro fuera de los dos documentos de `docs/` — ni en
  `requirements.txt`, ni en `notebooks/`, ni en `notebooks/artifacts/`.
