# Quickstart: validar la corrección de discrepancias

Guía de validación manual/por inspección — sin framework de tests (Principio I de la constitución).

## Prerrequisitos

- NB2 editado según `tasks.md` (nueva celda en §5.2, nuevo campo en §7) y re-ejecutado completo
  (Run All) en `notebooks/`, cwd = `notebooks/`.
- `docs/Informe_Tecnico_ShopMart_Olist.docx` y `docs/Defensa_ShopMart_Olist.pptx` ya editados con el
  hallazgo real.

## Paso 1 — Confirmar que NB2 corrió limpio y regeneró los artifacts

```bash
python -c "
import json
mc = json.load(open('notebooks/artifacts/model_card.json', encoding='utf-8'))
print('n_iter_actual:', mc.get('n_iter_actual', mc.get('hyperparameters', {}).get('n_iter_actual', 'NO ENCONTRADO')))
print('max_iter (techo buscado):', mc['hyperparameters']['max_iter'])
print('metrics test PR-AUC:', mc['metrics']['test']['average_precision'])
print('metrics test ROC-AUC:', mc['metrics']['test']['roc_auc'])
print('metrics test brier:', mc['metrics']['test']['brier_score'])
print('metrics test lift:', mc['metrics']['test']['top_decile_lift'])
"
```

**Esperado**: `n_iter_actual` es un entero presente (no `NO ENCONTRADO`) y `n_iter_actual <= max_iter`.
Las 4 métricas de test (PR-AUC, ROC-AUC, Brier, lift) son **idénticas** a los valores ya reportados
antes de este feature (SC-003) — si cambiaron, algo tocó el modelado y hay que investigar antes de
seguir.

## Paso 2 — Confirmar la tabla de subgrupo por `customer_state`

Abrir `notebooks/NB2_Modelado_Validacion.ipynb` (con outputs guardados tras el Run All) y verificar en
la celda nueva de §5.2 que:
- La tabla tiene una fila por estado presente en test (hasta 27).
- La suma de la columna `n` es igual a `len(raw_test)` (12,643 en la última ejecución confirmada; puede
  variar levemente si el split cambia, pero no debería cambiar porque `RANDOM_STATE=42` no se toca).
- Se identifica cuál es el estado con `avg_precision` más alto/bajo, y si algún estado tiene `NaN` o un
  `n` muy pequeño (candidatos: `RR`, `AP`, `AC`, `AM`, que ya se sabía que tienen <25 filas en test).

## Paso 3 — Extraer texto de ambos documentos y verificar las 5 inserciones

Reutiliza el mismo método de extracción de solo lectura de `specs/001-informe-metricas-negocio/quickstart.md`:

```bash
python - <<'PYEOF'
import zipfile, re

def docx_text(path):
    with zipfile.ZipFile(path) as z:
        xml = z.read('word/document.xml').decode('utf-8', errors='replace')
    return ''.join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', xml, re.S))

def pptx_slides_by_order(path):
    # Reordena por presentation.xml (sldIdLst), NO por nombre de archivo — ver research.md §3
    import re as _re
    with zipfile.ZipFile(path) as z:
        pres_xml = z.read('ppt/presentation.xml').decode('utf-8', errors='replace')
        rels_xml = z.read('ppt/_rels/presentation.xml.rels').decode('utf-8', errors='replace')
        rid_to_target = dict(_re.findall(r'Id="(rId\d+)"[^>]*Target="([^"]+)"', rels_xml))
        ordered_rids = _re.findall(r'<p:sldId[^>]*r:id="(rId\d+)"', pres_xml)
        out = []
        for rid in ordered_rids:
            target = rid_to_target.get(rid)
            if not target:
                continue
            slide_path = 'ppt/' + target.replace('../', '')
            xml = z.read(slide_path).decode('utf-8', errors='replace')
            text = ''.join(re.findall(r'<a:t>(.*?)</a:t>', xml, re.S))
            out.append((slide_path, text))
        return out

informe = docx_text('docs/Informe_Tecnico_ShopMart_Olist.docx')
print('customer_state mentions in Informe:', informe.count('customer_state') + informe.lower().count('por estado'))
print('n_iter_actual value referenced? search manually for the real number near "iteraciones"')

for path, text in pptx_slides_by_order('docs/Defensa_ShopMart_Olist.pptx'):
    if 'customer_state' in text or 'estado' in text.lower() or 'iteraciones' in text.lower():
        print(path, '->', text[:200].replace(chr(10), ' | '))
PYEOF
```

**Esperado**: el Informe y la Presentación citan el hallazgo real de la tabla del Paso 2 (no una
afirmación genérica sin cifra), y el valor de iteraciones citado coincide con `n_iter_actual` del
Paso 1, no con 150.

## Paso 4 — Checklist de criterios de éxito (de `spec.md`)

- [ ] **SC-001**: toda mención a disparidades por `customer_state`/estado en ambos documentos es
  trazable a la tabla real de §5.2.
- [ ] **SC-002**: toda mención a la cantidad de iteraciones del modelo coincide exactamente con
  `n_iter_actual` de `model_card.json`.
- [ ] **SC-003**: PR-AUC, ROC-AUC, Brier score y lift de test en `model_card.json` no cambiaron
  respecto a la versión anterior a este feature.
- [ ] **SC-004**: no se crearon secciones ni slides nuevas — mismas secciones (§3.4/§3.5/§4.1) y mismas
  slides editadas, no agregadas.
- [ ] **SC-005**: si algún estado con muestra pequeña mostró un resultado atípico, la salvedad
  correspondiente aparece explícitamente en el texto actualizado.

## Paso 5 — Revisión humana final

Abrir ambos documentos en Word/PowerPoint (o LibreOffice) y confirmar visualmente que el texto nuevo
hereda el estilo del párrafo/shape circundante, igual que en `specs/001-informe-metricas-negocio`.
