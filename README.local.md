# Ejecución local (VS Code / JupyterLab)

Guía específica para correr `NB1_Estrategia_Datos_EDA.ipynb` y `NB2_Modelado_Validacion.ipynb` en tu máquina. Para la visión general del proyecto ver [README.md](README.md); para Colab o Databricks ver [README.colab.md](README.colab.md) / [README.databricks.md](README.databricks.md).

## Requisitos previos

- **Python 3.10+**.
- **VS Code** con la extensión "Jupyter" (recomendado), o **JupyterLab/Jupyter Notebook** instalado.
- Git (para clonar el repo).
- Cuenta de Kaggle (opcional — solo si `notebooks/dataset/` no trae ya los 9 CSV, ver sección de datos más abajo).

## 1. Clonar el repositorio e instalar dependencias

```bash
git clone https://github.com/alexanderbaidal/ucg-olist-repeat-purchase-prediction.git
cd ucg-olist-repeat-purchase-prediction

pip install -r requirements.txt
```

`requirements.txt` es la única fuente de verdad de dependencias (pandas, scikit-learn, matplotlib, seaborn, joblib, pyarrow, kagglehub, python-dotenv) — no instales versiones distintas a mano.

> Este proyecto corre localmente contra la instalación **global** de Python de la máquina, sin `.venv`. Si tu Python global tiene paquetes rotos o en conflicto con otros proyectos, considera igualmente un entorno virtual (`python -m venv .venv`) — pero en ese caso instala `requirements.txt` y selecciona ese intérprete como kernel de forma explícita en VS Code (ver siguiente sección), para no terminar con dos "Python" que se confunden entre sí.

## 2. Abrir los notebooks con el directorio de trabajo correcto

Ambos notebooks usan rutas **relativas** (`Path("dataset")`, `Path("artifacts")`) asumiendo que el proceso arranca con el cwd en `notebooks/`, no en la raíz del repo.

- **VS Code:** el repo ya incluye [`.vscode/settings.json`](.vscode/settings.json) con `"jupyter.notebookFileRoot": "${fileDirname}"`, que fija el cwd del kernel a la carpeta del notebook abierto. Esto funciona sin importar si abriste el repo completo o solo `notebooks/` como carpeta del workspace — no necesitas configurar nada adicional. Si de todas formas el kernel arranca mal (por ejemplo, por una configuración de usuario que sobrescribe el setting del workspace), la celda de bootstrap (sección 1) de cada notebook detecta el problema y hace `os.chdir("notebooks")` automáticamente como red de seguridad.
- **JupyterLab / Jupyter Notebook:** ábrelo directamente desde la carpeta `notebooks/` (`jupyter lab` ejecutado con ese cwd, o navega ahí antes de abrir los archivos).

**Verificá el kernel/intérprete antes de correr:** si VS Code detecta más de un "Python" (por ejemplo, si además tenés un entorno de Anaconda, otra versión de Python, o un `.venv` viejo dando vueltas), asegurate de seleccionar el intérprete global donde corriste `pip install -r requirements.txt` — click en el selector de kernel (arriba a la derecha del notebook) → **"Select Another Kernel" → "Python Environments"**. Para confirmar cuál está activo, corré en una celda:
```python
import sys; print(sys.executable)
```

## 3. Ejecutar los notebooks en orden

1. **`NB1_Estrategia_Datos_EDA.ipynb`** — Run All. Carga y perfila las 9 tablas de Olist, limpia los datos, corre el EDA, genera features/preprocesador y particiona train/val/test. Persiste todo en `notebooks/artifacts/`.
2. **`NB2_Modelado_Validacion.ipynb`** — Run All. **No puede correrse antes ni de forma independiente de NB1** — su celda de bootstrap valida al inicio que existan los artefactos requeridos en `artifacts/` y falla con un mensaje explícito (listando qué archivo falta) si no es así. Entrena y tunea el modelo, evalúa en test, calcula interpretabilidad, construye la segmentación RFM y empaqueta `model_card.json` + el modelo serializado.

**Semilla fija:** `RANDOM_STATE = 42` en ambos notebooks — no la cambies si buscas reproducir exactamente los resultados del informe.

**Tiempo estimado:** menos de un minuto por notebook en una máquina estándar (el dataset completo son ~100k pedidos).

## Datos

Los 9 CSV de Olist ya están incluidos en `notebooks/dataset/`, así que no necesitas descargar nada. Si esa carpeta estuviera vacía o incompleta, NB1 descarga el dataset automáticamente vía [`kagglehub`](https://github.com/Kaggle/kagglehub):

1. Crea una cuenta en [kaggle.com](https://www.kaggle.com) si no tienes una.
2. Genera un token de API en [kaggle.com/settings/api](https://www.kaggle.com/settings/api).
3. Copia `.env.example` a `.env` en la raíz del repo y completa tus credenciales:
   ```bash
   cp .env.example .env
   ```
   ```
   KAGGLE_USERNAME=tu_usuario
   KAGGLE_KEY=tu_token_aqui
   ```
   NB1 carga este archivo automáticamente (vía `python-dotenv`, ya incluido en `requirements.txt`) antes de intentar la descarga. `.env` está en `.gitignore` — nunca se comitea. Alternativa sin archivo: exportar las mismas variables antes de abrir Jupyter/VS Code (`export KAGGLE_USERNAME=... KAGGLE_KEY=...`).
4. NB1 lo descarga y cachea en `~/.cache/kagglehub/` (no lo vuelve a descargar en corridas posteriores).

## Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| `AssertionError: No se encontró la carpeta de datos` / `FileNotFoundError` al leer `dataset/` o `artifacts/` | El kernel arrancó con el cwd en la raíz del repo, no en `notebooks/` | Revisa que `.vscode/settings.json` esté presente y que VS Code lo esté respetando; como alternativa, reinicia el kernel desde la carpeta `notebooks/` |
| `ModuleNotFoundError: No module named 'sklearn'` (o pandas/seaborn/etc.) | Faltan dependencias en el kernel activo | `pip install -r requirements.txt` en el mismo entorno/kernel que usa el notebook |
| NB2 lanza `FileNotFoundError: Faltan artefactos de NB1 en 'artifacts/': ...` | NB1 no se ejecutó completo antes, o se ejecutó en otra carpeta/entorno | Corre `NB1_Estrategia_Datos_EDA.ipynb` completo (Run All) primero, en este mismo checkout local |
| Falla la descarga de Kaggle Hub | Falta `.env` (o las variables `KAGGLE_USERNAME`/`KAGGLE_KEY`) o el token es inválido | Copia `.env.example` a `.env` y completa un token nuevo de kaggle.com/settings/api — o simplemente usa los CSV ya incluidos en `notebooks/dataset/` (no necesitas Kaggle si esa carpeta ya está poblada) |
