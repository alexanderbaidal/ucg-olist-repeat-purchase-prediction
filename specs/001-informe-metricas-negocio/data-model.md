# Content Model: Interpretación de negocio para ROC-AUC y Brier score

Este feature no maneja datos de software (no hay entidades, base de datos ni estado). En su lugar, este
documento describe las **4 unidades de contenido** que se insertan — el equivalente al "data model"
para un feature de documentación — para que `/speckit-tasks` pueda descomponerlas en tareas concretas
y verificables.

## Unidad de contenido: Interpretación de métrica

Cada una de las 4 inserciones comparte la misma estructura de campos:

| Campo | Descripción |
|---|---|
| `metrica` | `ROC-AUC` o `Brier score` |
| `cifra_test` | Valor exacto de test tomado de `model_card.json` (`roc_auc: 0.63080840696275` → se cita redondeado a `0.6308`/`0.63`; `brier_score: 0.0317144843278724` → se cita redondeado a `0.0317`) |
| `documento` | `Informe Técnico (.docx)` o `Presentación de Defensa (.pptx)` |
| `ubicacion` | Sección/slide exacta donde se inserta (ver tabla de instancias abajo) |
| `formato` | `prosa narrativa integrada` (Informe) o `callout Q&A` (Presentación) — resuelto en Clarifications del spec |
| `mensaje_clave` | El framing de negocio decidido en `research.md` (§2 para ROC-AUC, §3 para Brier score) |
| `fuente_de_verdad` | Siempre `notebooks/artifacts/model_card.json` — ninguna cifra se recalcula ni se toma de otro lugar |

## Instancias (las 4 inserciones concretas)

| # | Métrica | Documento | Ubicación | Formato |
|---|---|---|---|---|
| 1 | ROC-AUC | Informe Técnico | §3.2 (tabla) o §3.4 (párrafo de resultados) | Prosa narrativa integrada |
| 2 | ROC-AUC | Presentación | Slide 11 ("Resultados: Métricas del Modelo Final") o adyacente | Callout Q&A: "¿Qué significa un ROC-AUC de 0.63?" |
| 3 | Brier score | Informe Técnico | §3.2 (tabla) o §3.4 (párrafo de resultados) | Prosa narrativa integrada |
| 4 | Brier score | Presentación | Slide 11 o adyacente | Callout Q&A: "¿Qué significa un Brier score de 0.0317?" |

## Reglas de consistencia entre instancias

- Las instancias 1 y 2 (ROC-AUC) DEBEN transmitir el mismo `mensaje_clave` (ver FR-008): la lectura de
  "ordenamiento contra el azar" de `research.md` §2, adaptada en extensión a cada formato.
- Las instancias 3 y 4 (Brier score) DEBEN transmitir el mismo `mensaje_clave`: la lectura de
  "calibración de probabilidad + salvedad por desbalance de clases" de `research.md` §3, incluyendo la
  salvedad del desbalance en ambos documentos (no solo en el Informe) — omitirla en la Presentación
  rompería FR-006 (honestidad) en ese documento.
- Ninguna instancia introduce una cifra que no exista ya en `model_card.json` (FR-004).
- Ninguna instancia se inserta fuera de una sección/slide ya existente (SC-005).

## Trazabilidad a `model_card.json`

```json
"metrics": {
  "test": {
    "roc_auc": 0.63080840696275,      // instancias 1, 2
    "brier_score": 0.0317144843278724  // instancias 3, 4
  }
}
```
