# Quickstart: validar la corrección del split temporal

Guía de validación manual/por inspección — sin framework de tests (Principio I de la constitución).

## Prerrequisitos

- NB1 editado según `tasks.md` (celda de partición de §6 reemplazada por `temporal_split_by_class`,
  markdown de §6 actualizado) y re-ejecutado completo (Run All) en `notebooks/`, cwd = `notebooks/`.
- NB2 re-ejecutado completo (Run All) a continuación, sin editar su código (salvo, si aplica, la celda
  markdown de §3 con las cifras de CV — ver `data-model.md` instancia 2).
- `docs/Informe_Tecnico_ShopMart_Olist.docx` (+ PDF) y `docs/Defensa_ShopMart_Olist.pptx` ya editados
  con los valores reales resultantes.

## Paso 0 — Antes de tocar nada: capturar la línea base actual

```bash
python -c "
import json
mc = json.load(open('notebooks/artifacts/model_card.json', encoding='utf-8'))
print(json.dumps(mc, indent=2, ensure_ascii=False))
" > /tmp/model_card_antes.json

python -c "
import pandas as pd
df = pd.read_parquet('notebooks/artifacts/customer_segments.parquet')
print(df.groupby('segment_label').agg(n=('customer_unique_id','count'),
    recency=('recency_days','mean'), frequency=('frequency','mean'), monetary=('monetary','mean')))
" > /tmp/rfm_antes.txt
```

**Por qué**: sin esta línea base no se puede confirmar SC-003 (métricas cambian) ni SC-004
(`customer_segments.parquet` no cambia) después del Run All.

## Paso 1 — Confirmar que la partición cumple orden temporal + balance de clases (FR-001, FR-002)

```bash
python -c "
import pandas as pd
train = pd.read_parquet('notebooks/artifacts/raw_train.parquet')
val = pd.read_parquet('notebooks/artifacts/raw_val.parquet')
test = pd.read_parquet('notebooks/artifacts/raw_test.parquet')
for name, df in [('train', train), ('val', val), ('test', test)]:
    print(f'{name:5s} n={len(df):6d} rate={df[\"repeat_purchase\"].mean():.4f} '
          f'min={df[\"order_purchase_timestamp\"].min()} max={df[\"order_purchase_timestamp\"].max()}')
p90_train = train['order_purchase_timestamp'].quantile(0.90)
p10_test = test['order_purchase_timestamp'].quantile(0.10)
print('train p90:', p90_train, '<= test p10:', p10_test, '->', p90_train <= p10_test)
"
```

**Esperado**:
- `p90_train <= p10_test` es `True` (SC-001).
- Las tres tasas de recompra están dentro de ±0.5 puntos porcentuales del ~3.3% documentado (SC-002) —
  en la verificación de diseño de `research.md` §2 dieron 3.30% / 3.30% / 3.31%, prácticamente
  idénticas a las ya reportadas.

## Paso 2 — Confirmar que `customer_segments.parquet` no cambió (FR-007, SC-004)

```bash
python -c "
import pandas as pd
df = pd.read_parquet('notebooks/artifacts/customer_segments.parquet')
print(df.groupby('segment_label').agg(n=('customer_unique_id','count'),
    recency=('recency_days','mean'), frequency=('frequency','mean'), monetary=('monetary','mean')))
" > /tmp/rfm_despues.txt
diff /tmp/rfm_antes.txt /tmp/rfm_despues.txt && echo "RFM SIN CAMBIOS (esperado)" || echo "RFM CAMBIÓ (investigar antes de seguir)"
```

**Esperado**: `diff` vacío — ningún cambio.

## Paso 3 — Ver qué cambió realmente en `model_card.json`

```bash
python -c "
import json
mc = json.load(open('notebooks/artifacts/model_card.json', encoding='utf-8'))
print(json.dumps(mc, indent=2, ensure_ascii=False))
" > /tmp/model_card_despues.json
diff /tmp/model_card_antes.json /tmp/model_card_despues.json
```

**Esperado**: probablemente hay diferencias (hiperparámetros ganadores, métricas, umbral, importancia
de variables) — eso es correcto y esperado (US2 del spec), no un error. Este diff es exactamente la
lista de cifras que hay que revisar y, si difieren, actualizar en `docs/` (ver `data-model.md`,
instancias con `depende_del_split = Sí`). Si el diff está vacío, algo salió mal (verificar que NB1 y
NB2 realmente se re-ejecutaron con la celda corregida).

## Paso 4 — Confirmar la celda markdown de comparación de candidatos en NB2

Abrir `notebooks/NB2_Modelado_Validacion.ipynb` (con outputs guardados tras el Run All), localizar la
celda markdown "Resultado de la comparación..." (§3) y comparar las cifras citadas en prosa contra la
salida real de `comparison_df` en la celda de código inmediatamente anterior. Si difieren, editar la
celda markdown para que cite las cifras reales (instancia 2 de `data-model.md`).

## Paso 5 — Verificar las inserciones en Informe y Presentación

Reutiliza el mismo método de extracción de solo lectura de `specs/001-.../quickstart.md` y
`specs/002-.../quickstart.md`:

```bash
python - <<'PYEOF'
import zipfile, re

def docx_text(path):
    with zipfile.ZipFile(path) as z:
        xml = z.read('word/document.xml').decode('utf-8', errors='replace')
    return ''.join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', xml, re.S))

def pptx_slides_by_order(path):
    with zipfile.ZipFile(path) as z:
        pres_xml = z.read('ppt/presentation.xml').decode('utf-8', errors='replace')
        rels_xml = z.read('ppt/_rels/presentation.xml.rels').decode('utf-8', errors='replace')
        rid_to_target = dict(re.findall(r'Id="(rId\d+)"[^>]*Target="([^"]+)"', rels_xml))
        ordered_rids = re.findall(r'<p:sldId[^>]*r:id="(rId\d+)"', pres_xml)
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
for needle in ['0.033', '0.046', '0.047', '0.059', '0.63', 'temporal', 'más recientes']:
    print(f'Informe menciona {needle!r}:', needle in informe)

for path, text in pptx_slides_by_order('docs/Defensa_ShopMart_Olist.pptx'):
    if any(k in text for k in ['0.033', '0.046', '0.047', '0.059', '0.63', 'temporal', 'recientes']):
        print(path, '->', text[:200].replace(chr(10), ' | '))
PYEOF
```

**Esperado**: las cifras encontradas coinciden con `model_card.json` regenerado (Paso 3), no con las
cifras de la línea base (Paso 0). El mecanismo de split ("se ordena...para que el test contenga a los
clientes más recientes") sigue presente y sigue siendo verdadero del código real.

## Paso 6 — Checklist de criterios de éxito (de `spec.md`)

- [ ] **SC-001**: p90(train) ≤ p10(test) — confirmado en Paso 1.
- [ ] **SC-002**: tasa de recompra de cada split dentro de ±0.5pp del ~3.3% documentado — confirmado en
  Paso 1.
- [ ] **SC-003**: el 100% de las cifras de desempeño citadas en Informe/Presentación coinciden con
  `model_card.json` regenerado — confirmado en Pasos 3–5.
- [ ] **SC-004**: `customer_segments.parquet` no cambió — confirmado en Paso 2.
- [ ] **SC-005**: NB1 §6 (markdown), Informe §3.2 y Presentación slide 12 describen el mismo mecanismo
  sin contradicción — confirmado en Paso 4/5 por lectura comparativa.
- [ ] **SC-006**: target, filtros de elegibilidad, censura de 90 días, features, pipeline de
  preprocesamiento y `RANDOM_STATE=42` no cambiaron — confirmado por inspección del diff de
  `eda_summary.json` (campos fuera de `splits` deben ser idénticos).

## Paso 7 — Revisión humana final

Abrir ambos documentos en Word/PowerPoint (o LibreOffice) y confirmar visualmente que el texto editado
hereda el estilo del párrafo/shape circundante, igual que en `specs/001-...` y `specs/002-...`.
