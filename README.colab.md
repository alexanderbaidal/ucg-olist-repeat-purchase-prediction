# Ejecución en Google Colab

Guía específica para correr `NB1_Estrategia_Datos_EDA.ipynb` y `NB2_Modelado_Validacion.ipynb` en Google Colab. Para la visión general del proyecto ver [README.md](README.md); para ejecución local o Databricks ver [README.local.md](README.local.md) / [README.databricks.md](README.databricks.md).

## Lo que hay que entender antes de empezar

Colab **no clona el repositorio** cuando abres un notebook desde GitHub — solo trae el archivo `.ipynb` suelto a una VM efímera (`/content`). Eso significa que:

- `notebooks/dataset/` y `notebooks/artifacts/` **no existen** al abrir el notebook en Colab; la propia celda de bootstrap (sección 1) los crea vacíos.
- El dataset se obtiene siempre vía el fallback de `kagglehub` (no hay CSV locales que reutilizar).
- **NB2 depende de los artefactos que deja NB1** (`X_train.parquet`, `preprocessor.joblib`, etc.). Como no hay repo clonado ni artefactos persistidos entre sesiones, **NB1 y NB2 deben ejecutarse dentro del mismo runtime/sesión de Colab**, uno después del otro. Si el runtime se desconecta o expira entre medio, hay que volver a correr NB1 completo.

## 1. Abrir NB1 en Colab

Opciones, de más a menos recomendada:

- **Desde GitHub:** en [colab.research.google.com](https://colab.research.google.com), `File > Open notebook > GitHub`, pega la URL del repo (`https://github.com/alexanderbaidal/ucg-olist-repeat-purchase-prediction`) y selecciona `notebooks/NB1_Estrategia_Datos_EDA.ipynb`.
- **Subiendo el archivo:** `File > Upload notebook` y selecciona `NB1_Estrategia_Datos_EDA.ipynb` desde tu máquina.

No hace falta seleccionar GPU/TPU — el pipeline es tabular (pandas/scikit-learn), un runtime CPU estándar es suficiente.

## 2. Ejecutar NB1 completo

`Run All` (`Runtime > Run all`). La celda de bootstrap (sección 1) detecta Colab automáticamente y:

1. Instala las dependencias necesarias vía `%pip install` (pandas, numpy, scikit-learn, matplotlib, seaborn, joblib, pyarrow, kagglehub).
2. Crea `dataset/` y `artifacts/` en `/content`.
3. Deja lista la descarga automática del dataset vía `kagglehub` en la celda siguiente.

**Kaggle:** como no hay CSV locales, NB1 siempre descargará el dataset desde Kaggle Hub en Colab. Necesitas credenciales de Kaggle de alguna de estas formas:

- **Recomendado — Colab Secrets:** en el panel izquierdo, icono de llave ("Secrets"), agrega un secreto llamado `KAGGLE_API_TOKEN` con tu token (generado en [kaggle.com/settings/api](https://www.kaggle.com/settings/api)). Luego, antes de la celda de descarga, agrega una celda con:
  ```python
  import os
  from google.colab import userdata
  os.environ["KAGGLE_API_TOKEN"] = userdata.get("KAGGLE_API_TOKEN")
  ```
- **Alternativa rápida (menos segura):** descomenta y completa la línea `# os.environ["KAGGLE_API_TOKEN"] = "tu_token_aqui"` dentro de la celda de bootstrap, pegando el token directamente. Evita compartir o dejar público ese notebook con el token escrito.

## 3. Abrir y ejecutar NB2 — en el mismo runtime

Sin desconectar el runtime de NB1 (no cierres esa pestaña ni uses `Runtime > Disconnect and delete runtime`):

1. Desde ese mismo notebook, `File > Open notebook` y abre `notebooks/NB2_Modelado_Validacion.ipynb` (mismo repo de GitHub, o súbelo). Al abrirlo en la misma pestaña/sesión de Colab, normalmente se reutiliza el runtime ya conectado (y por tanto `artifacts/` sigue estando en `/content`).
2. Verifica que sigues conectado al mismo runtime activo (arriba a la derecha debería mostrar RAM/Disco en uso, no "Connect").
3. `Run All`. La celda de bootstrap de NB2 instala sus dependencias y valida que los artefactos de NB1 existan en `artifacts/` — si no los encuentra, falla con un mensaje explícito indicando qué archivo falta y que corras NB1 primero **en este mismo runtime**.

**Si Colab no reutiliza el runtime** (por ejemplo, se abrió en una pestaña nueva con una VM distinta): la forma más simple de no perder el trabajo de NB1 es descargar `artifacts/` antes de cambiar de notebook y volver a subirlo en la sesión de NB2:

```python
# Al final de NB1, en una celda nueva
!zip -r artifacts.zip artifacts
from google.colab import files
files.download("artifacts.zip")
```

Y al inicio de NB2 (después de la celda de bootstrap, antes de cargar los artefactos):

```python
from google.colab import files
uploaded = files.upload()  # selecciona artifacts.zip
!unzip -o artifacts.zip
```

## Límites del runtime gratuito de Colab a tener en cuenta

- Se desconecta tras ~90 minutos de inactividad, y hay un límite de sesión de ~12 horas.
- Si se desconecta entre NB1 y NB2, hay que volver a correr NB1 completo (o restaurar `artifacts/` desde el zip descargado, ver arriba).
- El pipeline completo (NB1 + NB2) corre en menos de un par de minutos en un runtime CPU estándar, así que en la práctica no debería chocar con estos límites si se ejecuta todo seguido.

## Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| NB2 lanza `FileNotFoundError: Faltan artefactos de NB1 en 'artifacts/': ...` | NB2 se abrió en un runtime distinto al de NB1 | Vuelve a correr NB1 completo en el runtime actual, o sube el zip de `artifacts/` (ver sección anterior) |
| Falla la descarga de `kagglehub` con error de autenticación | No configuraste `KAGGLE_API_TOKEN` (ni como Colab Secret ni como variable de entorno) | Sigue la sección "Kaggle" de arriba |
| `%pip install` tarda mucho o falla por conflicto de versiones | Poco común en Colab, pero puede pasar tras una actualización de la imagen base | Reinicia el runtime (`Runtime > Restart runtime`) y vuelve a correr `Run All` desde cero |
| El runtime se desconectó a medio proceso | Límite de inactividad/sesión de Colab gratuito | Reconéctate y vuelve a correr NB1 completo antes de NB2 |
