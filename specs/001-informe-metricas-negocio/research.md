# Research: Interpretación de negocio para ROC-AUC y Brier score

## 1. Enfoque técnico para editar `.docx` / `.pptx` preservando formato

**Decision**: usar `python-docx` y `python-pptx` como herramienta de edición local de un solo uso
(instalación efímera, sin agregarla a `requirements.txt`), abriendo el archivo original, localizando el
párrafo/tabla o el shape de texto correspondiente, e insertando los `runs` nuevos con el mismo estilo
de los `runs` vecinos (misma fuente, tamaño, si corresponde negrita/cursiva).

**Rationale**: durante la revisión previa (identificación de estos gaps) el texto se extrajo leyendo
directamente el XML interno de los archivos (`word/document.xml`, `ppt/slides/slideN.xml`) vía
`zipfile` + regex, lo cual es apropiado para **lectura** pero riesgoso para **escritura**: reescribir
XML de OOXML a mano fácilmente corrompe `run properties` (`w:rPr`/`a:rPr`), pierde el estilo heredado
de la plantilla de la maestría, o invalida el archivo si el XML resultante no es well-formed.
`python-docx`/`python-pptx` son las librerías estándar de la comunidad Python para esto, manipulan el
árbol OOXML de forma segura y son suficientes para el alcance acotado de este feature (insertar/editar
unos pocos `runs`/párrafos, no reestructurar el documento).

**Alternatives considered**:
- **Edición manual por el usuario en Word/PowerPoint**: descartada como único mecanismo porque no
  garantiza que el texto insertado cite la cifra exacta de `model_card.json` sin un paso de
  verificación programático (riesgo de transcripción manual incorrecta); se mantiene como paso final
  de revisión humana, no como mecanismo de edición.
- **Cirugía directa sobre el XML (mismo método usado para lectura)**: descartada para escritura por el
  riesgo de corromper `w:rPr`/`a:rPr` y el estilo de la plantilla institucional.
- **Regenerar los documentos desde una plantilla/pipeline de reporting**: descartada — no existe tal
  pipeline en el repo y crearlo sería scope creep frente al Principio I de la constitución (proyecto
  académico de dos notebooks + documentos finales, no una plataforma de generación de reportes).

## 2. Framing de negocio para ROC-AUC (0.6308 en test)

**Decision**: explicar el ROC-AUC como una medida de **capacidad de ordenamiento** anclada contra el
azar: "si el modelo toma al azar un cliente que sí recompró y uno que no, le da mayor score al que sí
recompró en el 63% de esas comparaciones (0.50 = azar puro, 1.00 = orden perfecto)". Se enmarca como
una mejora clara pero moderada sobre el azar, consistente con el framing ya usado para PR-AUC
("señal predictiva modesta pero accionable").

**Rationale**: esta es la interpretación probabilística estándar del ROC-AUC (equivalente al estadístico
de Mann-Whitney U normalizado) y es la forma más directa de traducirla sin jerga: no requiere explicar
curvas, sensibilidad/especificidad ni umbrales — solo la idea intuitiva de "ordenar mejor que una
moneda". Anclarla contra 0.50 (azar) le da al lector un punto de referencia inmediato para calibrar si
0.63 es bueno o malo, igual que la interpretación de PR-AUC se ancla contra el baseline (0.033).

**Alternatives considered**:
- **Comparar directamente ROC-AUC con PR-AUC en la misma frase**: descartada — son dos curvas distintas
  con bases de comparación diferentes (ROC-AUC no es sensible al desbalance de clases de la misma forma
  que PR-AUC); mezclarlas en una sola explicación arriesga confundir al lector en vez de aclarar.
  Se mantiene el `model_card.json` como fuente única para ambas cifras, pero se explican por separado.
- **Omitir el ancla de 0.50**: descartada — sin ese punto de referencia, "0.63" no comunica nada por sí
  solo a un lector no técnico.

## 3. Framing de negocio para Brier score (0.0317 en test)

**Decision**: explicar el Brier score como una medida de **qué tan bien calibradas** están las
probabilidades que da el modelo (qué tan cerca está, en promedio, la probabilidad predicha del
resultado real: 0 = probabilidades perfectas, 1 = probabilidades siempre opuestas al resultado real).
Se agrega explícitamente la salvedad de que, con una tasa base de recompra de solo ~3.3%, un valor bajo
de Brier score es *esperable incluso para un modelo poco informativo* (predecir una probabilidad baja
para todos ya produce un Brier score bajo), así que esta cifra debe leerse junto con PR-AUC/lift, no de
forma aislada.

**Rationale**: el Brier score es la métrica con mayor riesgo de "sobrevender" el modelo si se explica
sin matices — bajo un desbalance de clases severo (~3.3% positivos), el Brier score de un clasificador
trivial (probabilidad constante = tasa base) ya es bajo en términos absolutos, por lo que "0.0317 es
bajo" por sí solo no es una señal fuerte de buen desempeño. Incluir esta salvedad es lo que exige el
FR-006 (mantener el mismo nivel de honestidad que el resto del documento) y responde directamente al
edge case ya identificado en el spec ("si la interpretación... sugiere una conclusión más optimista que
la que ya dan PR-AUC y el lift").

**Alternatives considered**:
- **Presentar "cercano a 0 = bueno" sin la salvedad del desbalance**: descartada — es técnicamente
  correcta pero engañosa en este contexto específico, y violaría el requisito de consistencia narrativa
  con el resto del documento (que enfatiza que la señal predictiva es modesta).
- **Omitir el Brier score de la interpretación por ser demasiado matizado para explicar brevemente**:
  descartada — contradice la clarificación ya resuelta con el usuario (Brier score se interpreta con su
  cifra exacta en ambos documentos).

## 4. Addendum (durante implementación): espacio disponible en la slide de resultados

**Hallazgo**: la slide "Resultados: Métricas del Modelo Final" (índice de presentación 12, no
`slideN.xml`=13 — el orden de `sldIdLst` difiere de la numeración de archivo) solo tiene ~1.1 pulgadas
libres debajo del callout de PR-AUC. Cada bloque Q&A de PR-AUC ocupa 2 shapes (etiqueta + respuesta,
~1.1in en total) — no había espacio para replicar ese mismo patrón dos veces más (ROC-AUC y Brier
score) sin desbordar la slide.

**Decision**: usar un único shape nuevo con 2 párrafos compactos (uno por métrica, formato
"¿Métrica cifra? explicación.") en el mismo estilo de texto de cuerpo ya usado en la slide (Arial 12pt,
color `#475569`, sin negrita), en vez de 4 shapes nuevos (2 por métrica).

**Rationale**: preserva el mensaje de negocio y la cifra exacta de cada métrica (cumple FR-003, FR-004,
FR-008) sin desbordar el layout existente ni degradar visualmente la slide — que era la restricción de
mayor peso (plan.md: "cualquier edición debe ser indistinguible en estilo del resto del documento").

**Alternatives considered**: reducir tamaño de fuente de los shapes existentes para ganar espacio
(descartada — alteraría el estilo ya validado de PR-AUC, algo que plan.md pide preservar); mover los
callouts nuevos a una slide adyacente ("Impacto de Negocio", "Qué Aprendió el Modelo", "Límites y Casos
Atípicos") — descartada porque ninguna de esas slides tiene más espacio libre (~1.0–1.3in cada una,
mismo problema) y, temáticamente, la interpretación de una métrica de ajuste del modelo encaja mejor en
la slide de métricas que en las de impacto/interpretabilidad/límites.
