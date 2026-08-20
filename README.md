# Personalización Data-Driven en E-Commerce (Caso Olist) — Trabajo de Titulación

Trabajo de Titulación — Maestría en Data Science e IA (UCG). Caso de ciencia de datos que predice la **recompra de clientes** a partir de su primer pedido y complementa el análisis con una **segmentación RFM**, usando el dataset público [Olist Brazilian E-Commerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) como sustituto curado de los datos de la empresa ficticia "ShopMart". El contexto completo del caso está en [`caso-de-uso/data-driven-e-commerce.md`](caso-de-uso/data-driven-e-commerce.md).

## Estructura del repositorio

```
notebooks/
  NB1_Estrategia_Datos_EDA.ipynb      # Carga, limpieza, EDA, feature engineering, split train/val/test
  NB2_Modelado_Validacion.ipynb       # Modelado, tuning, evaluación, interpretabilidad, segmentación RFM
  dataset/                            # CSV crudos de Olist (ya incluidos en el repo)
  artifacts/                          # Artefactos generados por los notebooks (parquet, joblib, json)
entregables/
  Informe_Tecnico_ShopMart_Olist.docx # Informe técnico final
  Defensa_ShopMart_Olist.pptx         # Presentación de defensa (16 diapositivas)
plantillas/                           # Plantillas oficiales (.docx/.pptx) usadas como base de los entregables
caso-de-uso/                          # Brief del caso y plantilla/rúbrica de los notebooks
docs/                                 # PDFs del proceso de titulación y portada/cláusulas
```

## Requisitos

- **Python 3.10+** (el `.venv` del repo usa Python 3.14 vía `uv`).
- Jupyter (el `.venv` ya trae `ipykernel`/`jupyter_client`, pero **no** las librerías de análisis).
- Cuenta de Kaggle (opcional — solo si el dataset no está ya en `notebooks/dataset/`; ver sección de datos).

## Instalación de dependencias

El `.venv` incluido solo trae lo esencial para el kernel de Jupyter. Antes de ejecutar los notebooks, instala las librerías de análisis y modelado:

```bash
# Activa el entorno virtual del proyecto
# Windows: .venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate

pip install pandas numpy scikit-learn matplotlib seaborn joblib pyarrow kagglehub
```

> No existe un `requirements.txt` en el repo a propósito — instala solo lo necesario en el entorno que uses para no acoplar el proyecto a una lista de versiones específica.

## Datos

Los 9 CSV de Olist ya están incluidos en `notebooks/dataset/`, así que **no necesitas descargar nada** para correr los notebooks tal como están en el repositorio.

Si por alguna razón esa carpeta está vacía o incompleta, el propio NB1 (sección 1, "Configuración del entorno") descarga el dataset automáticamente vía [`kagglehub`](https://github.com/Kaggle/kagglehub):

1. Crea una cuenta en [kaggle.com](https://www.kaggle.com) si no tienes una.
2. Genera un token de API en [kaggle.com/settings/api](https://www.kaggle.com/settings/api).
3. Expórtalo como variable de entorno antes de abrir Jupyter:
   ```bash
   export KAGGLE_API_TOKEN=tu_token_aqui
   ```
4. Al ejecutar NB1, si no encuentra los CSV localmente, los descarga y cachea en `~/.cache/kagglehub/` (no vuelve a descargarlos en ejecuciones posteriores).

## Cómo ejecutar los notebooks

**Importante:** ambos notebooks usan rutas relativas (`dataset/`, `artifacts/`), así que deben ejecutarse **con el directorio de trabajo en `notebooks/`**, no en la raíz del repositorio. Si usas Jupyter Lab/Notebook, ábrelo desde esa carpeta; si usas VS Code, confirma que el kernel resuelva las rutas relativas a `notebooks/`.

1. **Ejecuta `NB1_Estrategia_Datos_EDA.ipynb` completo, de principio a fin**, en orden (Run All). Este notebook:
   - Carga y perfila las 9 tablas de Olist.
   - Limpia los datos y construye la tabla analítica a nivel cliente.
   - Corre el análisis exploratorio (EDA) completo.
   - Genera las features, ajusta el pipeline de preprocesamiento y parte los datos en train/val/test.
   - Persiste todo en `notebooks/artifacts/` (parquet, `preprocessor.joblib`, `eda_summary.json`).

2. **Luego ejecuta `NB2_Modelado_Validacion.ipynb` completo, de principio a fin.** Este notebook depende de los artefactos que produce NB1, así que **no puede correrse antes ni de forma independiente**. NB2:
   - Carga los artefactos de NB1.
   - Compara modelos candidatos por validación cruzada.
   - Optimiza hiperparámetros del modelo final (`HistGradientBoostingClassifier`).
   - Evalúa en el conjunto de test y genera interpretabilidad (permutation importance).
   - Construye la segmentación RFM complementaria.
   - Empaqueta el modelo final, `model_card.json` y las predicciones de test en `notebooks/artifacts/`.

**Semilla fija:** `RANDOM_STATE = 42` en ambos notebooks — no la cambies si buscas reproducir exactamente los resultados reportados en el informe.

**Tiempo estimado:** cada notebook corre en menos de un minuto en una máquina estándar (el dataset completo son ~100k pedidos).

## Artefactos generados (`notebooks/artifacts/`)

| Archivo | Descripción |
|---|---|
| `X_train/X_val/X_test.parquet`, `y_train/y_val/y_test.parquet` | Splits preprocesados (features transformadas + target) |
| `raw_train/val/test.parquet` | Splits crudos (pre-transformación), para trazabilidad e interpretabilidad |
| `preprocessor.joblib` | Pipeline de preprocesamiento (`ColumnTransformer`) ajustado solo en train |
| `eda_summary.json` | Resumen de hallazgos y decisiones del EDA (NB1) |
| `orders_history.parquet` | Historial completo de pedidos por cliente, insumo de la segmentación RFM |
| `repeat_purchase_model.joblib` | Modelo final serializado |
| `model_card.json` | Métricas, hiperparámetros, semilla y limitaciones conocidas del modelo |
| `test_predictions.parquet` | Predicciones del modelo sobre el conjunto de test |
| `customer_segments.parquet` | Segmentación RFM de toda la base de clientes |

Si regeneras los notebooks, estos artefactos se sobrescriben — y con ellos deja de estar sincronizado el contenido de `entregables/` (que embebe cifras y figuras como texto/imágenes estáticas, no referencias vivas). Si cambias resultados, actualiza también el informe y la presentación.

## Entregables

- **Informe técnico:** [`entregables/Informe_Tecnico_ShopMart_Olist.docx`](entregables/Informe_Tecnico_ShopMart_Olist.docx) — documento completo con metodología, EDA, modelado, ética y conclusiones, construido sobre `plantillas/Plantilla Informe Tecnico.docx`.
- **Presentación de defensa:** [`entregables/Defensa_ShopMart_Olist.pptx`](entregables/Defensa_ShopMart_Olist.pptx) — 16 diapositivas construidas sobre `plantillas/Plantilla de Presentación para la Defensa de Análisis de Caso.pptx`.

## Más contexto para quien edite este repositorio

Ver [`CLAUDE.md`](CLAUDE.md) para el detalle del contrato de artefactos entre NB1 y NB2, decisiones de modelado ya validadas contra los números reales (evitar reintroducir narrativas ya descartadas), y notas sobre el entorno de ejecución.
