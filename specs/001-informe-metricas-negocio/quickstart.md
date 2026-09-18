# Quickstart: validar la interpretación de negocio de ROC-AUC y Brier score

Guía de validación manual/por inspección — este repo no tiene framework de tests (Principio I de la
constitución), así que "probar" el feature significa verificar, por inspección directa de los
documentos, que las 4 unidades de contenido de `data-model.md` están presentes, son consistentes entre
sí y coinciden con `model_card.json`.

## Prerrequisitos

- Los archivos `docs/Informe_Tecnico_ShopMart_Olist.docx` y `docs/Defensa_ShopMart_Olist.pptx` ya
  editados según `tasks.md`.
- `notebooks/artifacts/model_card.json` presente (no se regenera como parte de este feature).
- Python disponible en el entorno (ya usado en este repo vía `requirements.txt`; no se necesita
  instalar nada nuevo para **validar** — solo se usa la librería estándar `zipfile`).

## Paso 1 — Confirmar las cifras fuente

```bash
python -c "
import json
mc = json.load(open('notebooks/artifacts/model_card.json', encoding='utf-8'))
print('roc_auc test:', mc['metrics']['test']['roc_auc'])
print('brier_score test:', mc['metrics']['test']['brier_score'])
"
```

**Esperado**: `roc_auc test: 0.6308...` y `brier_score test: 0.0317...` — estos son los únicos valores
que las nuevas frases interpretativas pueden citar (redondeados a `0.6308`/`0.63` y `0.0317`).

## Paso 2 — Extraer el texto de ambos documentos y buscar las 4 inserciones

Reutiliza el mismo método de extracción de solo lectura usado para diagnosticar el gap original
(zipfile + regex sobre el XML de OOXML, sin dependencias externas):

```bash
python - <<'PYEOF'
import zipfile, re

def docx_text(path):
    with zipfile.ZipFile(path) as z:
        xml = z.read('word/document.xml').decode('utf-8', errors='replace')
    return ''.join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', xml, re.S))

def pptx_text(path):
    out = []
    with zipfile.ZipFile(path) as z:
        for n in sorted(n for n in z.namelist() if re.match(r'ppt/slides/slide\d+\.xml$', n)):
            xml = z.read(n).decode('utf-8', errors='replace')
            out.append((n, ''.join(re.findall(r'<a:t>(.*?)</a:t>', xml, re.S))))
    return out

informe = docx_text('docs/Informe_Tecnico_ShopMart_Olist.docx')
for term in ['ROC-AUC', 'Brier']:
    print(term, '-> aparece', informe.count(term), 'veces en el Informe')

for slide, text in pptx_text('docs/Defensa_ShopMart_Olist.pptx'):
    if 'ROC-AUC' in text or 'Brier' in text:
        print(slide, '->', text[:200].replace(chr(10), ' '))
PYEOF
```

**Esperado**:
- El Informe menciona `ROC-AUC` y `Brier` al menos dos veces cada uno (la cifra en la tabla §3.2 + la
  frase interpretativa en §3.4).
- Al menos una slide de la Presentación contiene `ROC-AUC` con el callout "¿Qué significa...?" y su
  cifra (`0.63`), y al menos una slide contiene `Brier` con su callout y cifra (`0.0317`).

## Paso 3 — Checklist de criterios de éxito (de `spec.md`)

- [ ] **SC-001**: toda aparición del valor de ROC-AUC en ambos documentos tiene una frase interpretativa
  adyacente (no solo la cifra sola en una tabla).
- [ ] **SC-002**: ídem para Brier score.
- [ ] **SC-003**: el mensaje de negocio de cada métrica es el mismo en el Informe y en la Presentación
  (comparar manualmente el texto extraído en el Paso 2 contra `research.md` §2 y §3).
- [ ] **SC-004**: ninguna cifra relacionada a métricas del modelo en los documentos difiere de
  `model_card.json` (comparar contra la salida del Paso 1).
- [ ] **SC-005**: no se agregó ninguna sección ni slide nueva — confirmar que el número total de slides
  del `.pptx` y de secciones de primer nivel (`1.`–`6.`) del `.docx` no cambió respecto a la versión
  anterior al feature.

## Paso 4 — Revisión humana final

Abrir ambos documentos en Word/PowerPoint (o LibreOffice) y confirmar visualmente que:
- El texto insertado hereda la tipografía/estilo del párrafo o slide donde se insertó (sin fuentes o
  tamaños inconsistentes).
- La salvedad de desbalance de clases en la interpretación del Brier score (ver `research.md` §3) está
  presente en **ambos** documentos, no solo en el Informe.
