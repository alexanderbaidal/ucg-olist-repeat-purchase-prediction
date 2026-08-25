# Ejecución en Databricks

Guía específica para correr `NB1_Estrategia_Datos_EDA.ipynb` y `NB2_Modelado_Validacion.ipynb` en Databricks. Para la visión general del proyecto ver [README.md](README.md); para ejecución local o Colab ver [README.local.md](README.local.md) / [README.colab.md](README.colab.md).

## Requisitos previos

- Acceso a un workspace de Databricks con permiso para crear/usar un clúster y para usar **Databricks Repos**.
- Un proveedor Git conectado al workspace (GitHub) con acceso a este repositorio.

## 1. Clonar el repo con Databricks Repos (no lo importes como notebook suelto)

En el panel lateral: **Repos > Add Repo**, pega la URL del repositorio (`https://github.com/alexanderbaidal/ucg-olist-repeat-purchase-prediction.git`) y confirma. Si el repo es privado, necesitarás configurar credenciales Git en el workspace primero (`Settings > Linked accounts`).

**Por qué Repos y no "Import" de un `.ipynb` suelto:** ambos notebooks usan rutas relativas (`Path("dataset")`, `Path("artifacts")`, `../requirements.txt`) asumiendo que el notebook vive junto al resto del repo (`notebooks/dataset/`, `notebooks/artifacts/`, `requirements.txt` en la raíz). Con Databricks Repos, el checkout completo queda disponible en el filesystem del clúster junto al notebook; con un `.ipynb` importado sin el resto del repo, esas rutas no resuelven a nada.

## 2. Adjuntar un clúster

Cualquier clúster de propósito general sirve — no se necesita GPU (el pipeline es tabular, pandas/scikit-learn). Verifica que el runtime del clúster traiga Python 3.10+.

## 3. Ejecutar NB1 completo

Abre `notebooks/NB1_Estrategia_Datos_EDA.ipynb` dentro del Repo, adjunta el clúster y **Run All**. La celda de bootstrap (sección 1) detecta Databricks (`DATABRICKS_RUNTIME_VERSION` en el entorno) e instala las dependencias con:

```python
%pip install -q -r ../requirements.txt
```

Como los 9 CSV de Olist ya están commiteados en `notebooks/dataset/` (y llegan junto con el Repo), **no necesitas configurar Kaggle** en el flujo normal. El fallback de descarga vía `kagglehub` solo se activa si esa carpeta faltara o estuviera incompleta; en ese caso necesitarías credenciales de Kaggle expuestas como variables de entorno `KAGGLE_USERNAME` y `KAGGLE_KEY` (por ejemplo vía [Databricks secrets](https://docs.databricks.com/en/security/secrets/index.html) leídos en una celda: `os.environ["KAGGLE_USERNAME"] = dbutils.secrets.get(scope="...", key="kaggle-username")` y `os.environ["KAGGLE_KEY"] = dbutils.secrets.get(scope="...", key="kaggle-key")`). El notebook también intenta cargar un `.env` local vía `python-dotenv`, pero no aplica aquí salvo que subas uno manualmente al Repo — usa Databricks secrets en su lugar.

NB1 persiste sus artefactos en `notebooks/artifacts/`, dentro del propio checkout del Repo.

## 4. Ejecutar NB2 completo

Abre `notebooks/NB2_Modelado_Validacion.ipynb` (mismo Repo), adjunta un clúster — idealmente el mismo que usaste para NB1, en la misma sesión de trabajo — y **Run All**. Su celda de bootstrap instala dependencias igual que NB1, y valida al inicio que los artefactos de NB1 existan en `artifacts/`; si faltan, falla con un mensaje explícito indicando qué archivo falta.

## Nota sobre persistencia de `artifacts/`

Este proyecto asume el patrón más simple: **Databricks Repos + filesystem local del checkout**, igual que en local/Colab — NB1 y NB2 comparten `artifacts/` porque ambos leen/escriben la misma carpeta del Repo. Esto funciona sin configuración adicional mientras seguido NB1 y NB2 se ejecuten contra el mismo Repo checkout, típicamente en el mismo clúster/sesión de trabajo.

Databricks generalmente persiste el contenido de un Repo (incluyendo archivos no versionados como los parquet/joblib generados) más allá de la vida de un clúster individual, pero esta garantía puede variar según la versión de runtime y la configuración del workspace — no la trates como equivalente a un storage administrado. Si tu equipo necesita persistencia garantizada de `artifacts/` entre clústeres o sesiones (por ejemplo, para servir el modelo desde otro job), la alternativa más robusta es escribir esos archivos a **DBFS** (`/dbfs/...`) o a un **Volume de Unity Catalog** en lugar de (o además de) la carpeta local del Repo — eso requeriría adaptar `ARTIFACTS_DIR` en ambos notebooks a esa ruta cuando `IN_DATABRICKS` sea verdadero, algo fuera del alcance de esta guía introductoria.

## Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| `AssertionError` / `FileNotFoundError` al buscar `dataset/` o `artifacts/` | El notebook se importó suelto (sin Repos) o el Repo no incluye la carpeta esperada | Usa **Repos > Add Repo** para clonar el repo completo, no `Workspace > Import` de un `.ipynb` aislado |
| `%pip install -q -r ../requirements.txt` falla con "file not found" | La ruta relativa `../requirements.txt` no resuelve porque el cwd del notebook no es `notebooks/` | Confirma que abriste el notebook dentro del Repo clonado (no una copia movida a otra carpeta del workspace) |
| NB2 lanza `FileNotFoundError: Faltan artefactos de NB1 en 'artifacts/': ...` | NB1 no corrió antes contra el mismo Repo/clúster, o los artefactos no persistieron | Corre NB1 completo (Run All) antes de NB2 contra el mismo checkout del Repo |
| Falla la descarga de `kagglehub` | Falta el token de Kaggle como secret/variable de entorno (solo debería ocurrir si `dataset/` no llegó con el Repo) | Configura un Databricks secret con tu token de Kaggle, o verifica que `notebooks/dataset/` sí tenga los 9 CSV en el Repo |
