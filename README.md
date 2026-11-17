# Predicción de demoras de vuelos

**Trabajo Final Integrador — Analítica de Negocios Avanzada (UCEMA, 2026)**

MVP de un producto de datos que estima la **probabilidad de que un vuelo salga con más de 15 minutos de demora**. Combina vuelos domésticos de EE.UU. (Kaggle) con el clima diario de cada aeropuerto (Open-Meteo). El resultado se traduce en un semáforo de riesgo con acciones sugeridas para el equipo de operaciones.

## Contenido del repositorio

| Archivo / carpeta | Qué es |
|---|---|
| `tp.ipynb` | **Notebook principal:** datos, EDA, armado de la ABT, modelado, interpretabilidad (SHAP), semáforo de riesgo y validación fuera de la ventana (2023). |
| `simulacion_impacto.ipynb` | Simulación del impacto económico del semáforo (FASE 5). Usa **datos sintéticos** y las predicciones que genera `tp.ipynb`. |
| `requirements.txt` | Dependencias con versiones fijas. |
| `clima_2022_por_aeropuerto.parquet` | Caché del clima diario de 2022 por aeropuerto (Open-Meteo). |
| `generated/clima_2023_por_aeropuerto.parquet` | Caché del clima diario de 2023 por aeropuerto (Open-Meteo). |
| `generated/` | Salidas que generan los notebooks. |
| `dictionary.md` / `dictionary.html` | Diccionario de datos del dataset de vuelos. |

## Requisitos

- **Python 3.13** (el proyecto se desarrolló con 3.13 en Windows).
- **Jupyter** para abrir los notebooks: Jupyter Notebook, JupyterLab o VS Code con la extensión de Jupyter.
- Las librerías de `requirements.txt`:

| Librería | Versión | Para qué se usa |
|---|---|---|
| `pandas` | 2.2.3 | Manejo de datos y armado de la ABT |
| `numpy` | 2.5.4 | Cálculo numérico |
| `matplotlib` | 3.11.2 | Gráficos |
| `seaborn` | 0.13.2 | Gráficos estadísticos (heatmaps, boxplots) |
| `pyarrow` | 26.0.0 | Lectura y escritura de los cachés de clima en parquet |
| `requests` | 2.34.2 | Consultas a la API de clima (solo si falta el caché) |
| `scikit-learn` | 1.9.1 | Split, validación cruzada, RFECV, regresión logística, red neuronal, métricas |
| `scipy` | 1.18.1 | Tests estadísticos, KS y distribuciones para la búsqueda de hiperparámetros |
| `xgboost` | 3.4.1 | Modelo elegido (XGBoost) |
| `lightgbm` | 4.7.0 | Modelo comparado (LightGBM) |
| `shap` | 0.54.1 | Interpretabilidad del modelo (sección 6.12) |
| `ipykernel` | 7.4.0 | Registrar el entorno como kernel de Jupyter |

## Instalación

Se recomienda crear un entorno virtual propio para no mezclar versiones con otras instalaciones de Python.

**Windows (PowerShell):**

```powershell
# 1. Crear el entorno virtual
py -3.13 -m venv $HOME\.virtualenvs\tp_ia

# 2. Instalar las dependencias (desde la carpeta del repositorio)
& $HOME\.virtualenvs\tp_ia\Scripts\python.exe -m pip install -r requirements.txt

# 3. Registrar el entorno como kernel de Jupyter
& $HOME\.virtualenvs\tp_ia\Scripts\python.exe -m ipykernel install --user --name tp_ia --display-name "Python (TP IA)"
```

**macOS / Linux:**

```bash
python3.13 -m venv ~/.virtualenvs/tp_ia
~/.virtualenvs/tp_ia/bin/python -m pip install -r requirements.txt
~/.virtualenvs/tp_ia/bin/python -m ipykernel install --user --name tp_ia --display-name "Python (TP IA)"
```

Si todavía no tenés Jupyter, podés instalarlo en el mismo entorno con `python -m pip install notebook`, o abrir los notebooks desde VS Code.

## Datos

El dataset de vuelos **no está en el repositorio**, porque pesa unos 600 MB. Hay que descargarlo de Kaggle:

1. Descargar el dataset **Flight Delay and Cancellation Dataset (2019-2023)**:
   <https://www.kaggle.com/datasets/patrickzel/flight-delay-and-cancellation-dataset-2019-2023>
2. Copiar el archivo `flights_sample_3m.csv` en la **carpeta raíz del repositorio**, al lado de `tp.ipynb`, sin cambiarle el nombre.

La primera vez que se corre, el notebook genera a partir de ese archivo:
- `flights_2022_es.csv`: vuelos de 2022 con las columnas en español;
- `generated/flights_2023_es.csv`: vuelos de 2023, para la validación fuera de la ventana.

Las corridas siguientes leen esos archivos y ya no vuelven a cargar el CSV original. Los tres archivos están en `.gitignore`.

**Clima:** los cachés de clima de 2022 y 2023 ya están en el repositorio, así que **no hace falta internet ni llamar a la API**. Solo si se borra algún caché, el notebook lo vuelve a armar consultando OurAirports (coordenadas de los aeropuertos) y la API histórica de Open-Meteo.

## Cómo correr

Los notebooks se corren **en este orden**, porque el segundo usa archivos que genera el primero.

1. **Abrir Jupyter desde la carpeta del repositorio.** Las rutas de los archivos son relativas a esa carpeta.
2. **Correr `tp.ipynb`:**
   - Elegir el kernel **"Python (TP IA)"**.
   - Ejecutar todo con *Kernel → Restart Kernel and Run All Cells* (o *Restart* + *Run All* en VS Code).
   - Las celdas se tienen que correr **de arriba hacia abajo**: cada sección usa variables de las anteriores.
   - Las secciones más lentas son la selección de variables (RFECV, 6.2) y las búsquedas de hiperparámetros (6.4 a 6.6).
3. **Correr `simulacion_impacto.ipynb`** con el mismo kernel y *Run All*. Lee las predicciones que exportó `tp.ipynb` en la sección 6.11.

## Qué genera cada notebook

| Archivo | Lo genera | Contenido |
|---|---|---|
| `flights_2022_es.csv` | `tp.ipynb` (2.1) | Vuelos de 2022 con columnas en español |
| `generated/flights_2023_es.csv` | `tp.ipynb` (6.13) | Vuelos de 2023 (enero a agosto) |
| `generated/predicciones_test.csv` | `tp.ipynb` (6.11) | Probabilidad y nivel del semáforo de los vuelos de test |
| `generated/predicciones_validacion.csv` | `tp.ipynb` (6.11) | Lo mismo, para los vuelos de train predichos con validación cruzada |
| `generated/sintetico/` | `simulacion_impacto.ipynb` | Datos **sintéticos** de la simulación (prefijo `sintetico_`) |

## Reproducibilidad

- **Semilla fija:** todo lo que tiene aleatoriedad (muestreo, split, validación cruzada, modelos) usa `random_state = 42`. Con las mismas versiones de `requirements.txt`, los resultados se repiten.
- **Configuración centralizada:** las constantes del proyecto (umbral de demora, horizonte de predicción, umbrales del semáforo, nombres de archivos) están en la sección **1. Configuración** de `tp.ipynb`.
- **Sin leakage:** las variables que solo se conocen después del vuelo se excluyen del modelo, las tasas históricas se calculan solo con el pasado de los vuelos de train, y todo lo que usa el target se ajusta solo con train.
- **Datos sintéticos separados:** la simulación de impacto guarda sus datos en `generated/sintetico/`, con prefijo `sintetico_`, y nunca se mezclan con los datos reales.

## Problemas frecuentes

- **No aparece el kernel "Python (TP IA)":** repetir el paso 3 de la instalación y reiniciar Jupyter o VS Code.
- **`FileNotFoundError: flights_sample_3m.csv`:** falta el dataset de Kaggle en la carpeta raíz (ver [Datos](#datos)).
- **`FileNotFoundError: generated/predicciones_test.csv`** en la simulación: primero hay que correr `tp.ipynb` completo.
- **`ModuleNotFoundError`:** el notebook está usando otro kernel. Elegir "Python (TP IA)" o volver a instalar `requirements.txt` en el entorno activo.
