# AGENTS.md — Predicción de demoras de vuelos (TFI Analítica de Negocios Avanzada, UCEMA 2026)

Este archivo le da contexto a cualquier agente de IA (Codex, Claude Code, etc.) que trabaje en este repositorio. Leelo completo antes de proponer o hacer cambios.

## 1. Qué es este proyecto

Es el **Trabajo Final Integrador** de la materia *Analítica de Negocios Avanzada* (UCEMA, 2026). La consigna pide diseñar e implementar un **MVP de producto de datos** con un pipeline completo: procesamiento de datos → EDA → modelado → visualización → despliegue, orientado a un problema real de negocio y con impacto medible.

**Problema de negocio:** vuelos que salen con **más de 15 minutos de demora** sin que nadie lo haya anticipado. Esto provoca conexiones perdidas, retrasos operativos, compensaciones, costos operativos extra y pérdida de confianza del pasajero.

**Solución propuesta:** un sistema de apoyo a la decisión que estima la **probabilidad de que un vuelo se demore más de 15 minutos**. Usa como insumos el riesgo estructural del vuelo, el desempeño reciente de la aerolínea, la situación del aeropuerto y el clima. El resultado se muestra en un **dashboard** con:
- un semáforo de riesgo por vuelo,
- los factores que explican cada alerta,
- acciones sugeridas según el nivel de riesgo.

El umbral de alerta se define según el costo de los falsos positivos y los falsos negativos. La **decisión final la toma el usuario de operaciones**, y su registro retroalimenta el modelo y sirve para medir la adopción.

## 2. Marco teórico: Data Product Canvas (FASE 1, ya aprobado)

El canvas ya fue **corregido y aceptado por la profesora**. Es la referencia conceptual del proyecto: el trabajo técnico tiene que mantenerse alineado con él y no contradecirlo. Si algo del código se aparta del canvas, hay que señalarlo; no cambies el planteo por tu cuenta.

| Bloque | Contenido |
|---|---|
| **Problema** | Vuelos con más de 15 min de demora no anticipada → conexiones perdidas, retrasos operativos, compensaciones, costos extra, pérdida de confianza. |
| **Datos** | Kaggle (fecha, aerolínea, origen/destino, horarios programados, duración, distancia) + API de clima. ABT de 31 features organizadas en bloques: riesgo estructural, desempeño reciente de la aerolínea, situación del aeropuerto y clima. |
| **Hipótesis** | Anticipar la demora mejora la toma de decisiones y reduce los tiempos y costos operativos extra que genera. |
| **Solución** | Probabilidad de demora mayor a 15 min + dashboard con semáforo, factores explicativos y acciones sugeridas. Umbral basado en el costo de FP y FN, y la decisión final queda en manos de operaciones (*human in the loop*). |
| **KPIs técnicos** | Recall, Precision, ROC-AUC. |
| **KPIs estratégicos** | NPS. Costo de compensaciones por vuelo demorado (total pagado en compensaciones / vuelos demorados). Costo de reubicación por conexión perdida (total gastado en reubicaciones / pasajeros reubicados). Costo operativo por vuelo demorado (costos extra de tripulación, aeronave y aeropuerto / vuelos demorados). |
| **Actores** | Usuarios: equipos de planeamiento y operaciones. Consumidores: finanzas y comercial / atención al cliente. |
| **Acciones** | Revisión diaria de los vuelos alertados dentro de una **ventana de 7 días**. Reprogramar los vuelos de alto riesgo cuando hay margen. Avisar con anticipación a los pasajeros. Reubicar de forma preventiva a los pasajeros con conexión. Reasignar tripulación y recursos. Ante una cancelación, reasignar a vuelos y asientos con características parecidas. |
| **Valor** | Ahorro de costos, satisfacción del cliente, eficiencia. |
| **Riesgos** | FP → *alert fatigue* y pérdida de confianza. FN → costos no evitados. Clima: el modelo usa clima **observado**, pero en producción habría **pronóstico a 7 días**. Leakage: el target encoding se tiene que calcular **después del split**. Adopción y drift. |
| **Impacto** | Costo evitado, conexiones perdidas evitadas, compensaciones, reasignaciones, ahorro de costo y de tiempo operativo. |

## 3. ⚠️ Decisión abierta: datos operativos que NO tenemos

La solución del canvas incluye acciones como **reubicar pasajeros, reprogramar vuelos y reasignar tripulación**, y KPIs de costos de compensación, reubicación y operación. **El dataset no tiene esos datos**: no hay pasajeros, conexiones, tripulaciones, costos ni NPS.

La profesora habilitó dos alternativas y **la autora todavía NO eligió**:

1. **Solución teórica:** describir la capa de acciones y los KPIs de costo de forma conceptual (reglas de negocio, flujo de decisión, cómo se mediría), sin implementarlos con datos.
2. **Datos sintéticos:** generar datos simulados (por ejemplo, pasajeros con conexión por vuelo, costos de compensación y reubicación, disponibilidad de tripulación) para implementar y simular esa parte de la solución y estimar el impacto (relacionado con la FASE 5: simulaciones / A/B).

**Reglas para el agente sobre este punto:**
- **No tomes esta decisión ni avances con ninguna de las dos opciones sin que la autora lo pida explícitamente.** Si una tarea depende de esto, preguntá antes.
- Si se pide comparar las opciones, presentá pros y contras de cada una sin dar ninguna por elegida.
- Si en algún momento se eligen datos sintéticos, tienen que estar **claramente separados y etiquetados** como sintéticos (archivo y carpeta propios, prefijo `sintetico_`, semilla fija). Los supuestos de generación tienen que estar documentados y justificados (distribuciones, fuentes de referencia). Nunca se mezclan sin marca con los datos reales.

## 4. Estado actual: notebook `tp.ipynb` (PRIMERA VERSIÓN)

`tp.ipynb` es un **primer borrador** que se va a ir iterando. No es la versión final: se espera que cambie de estructura, se reescriban secciones y en algún momento parte de la lógica se mueva a módulos `.py`. Tomá lo que hay como punto de partida, no como algo cerrado.

Flujo actual del notebook:

1. **Carga y recorte:** lee `flights_sample_3m.csv` (Kaggle, ~3M vuelos de EE.UU.), filtra 2022 y traduce las columnas al español. Guarda el resultado en `flights_2022_es.csv` (si ya existe, lo reutiliza).
2. **Muestreo:** toma el 25% estratificado por mes (`random_state=42`) y se queda con los aeropuertos de origen que cubren el 90% de los vuelos.
3. **Target:** `retrasado = retraso_salida > 15` (≈ 21% positivos → desbalance moderado).
4. **Segunda fuente (clima):** obtiene las coordenadas de los aeropuertos desde OurAirports y el clima diario histórico de 2022 desde la API de Open-Meteo (temperatura media, precipitación, viento máximo). Lo cachea en `clima_2022_por_aeropuerto.parquet` y lo une por `origen` + `fecha_vuelo`.
5. **EDA:** outliers (IQR, con decisión justificada de no tratarlos), univariado (numéricas y categóricas), resumen estadístico, separación de variables con leakage vs. utilizables, bivariado contra el target (día, franja horaria, mes, volumen vs. tasa, aerolínea, aeropuerto, clima por aeropuerto, boxplots) y multivariado (heatmap de correlaciones).
6. **ABT:** excluye cancelados y desviados. Features temporales, lags de 7 días (tasa de retraso por aerolínea y por aeropuerto de origen), congestión (`vuelos_dia_origen`), cascada (`retraso_previo_ruta`), one-hot de aerolínea y target encoding de origen y destino.
7. **Modelos:** Regresión Logística (con y sin `class_weight`) como baseline y XGBoost con `RandomizedSearchCV` (CV estratificado de 5 folds, scoring = recall). Ajuste de umbral (se eligió 0.40), ROC, matrices de confusión e importancia de variables.
8. **Conclusión** preliminar.

### Problemas técnicos conocidos a revisar (no cerrados)
Están detectados en el código actual. Antes de "corregirlos", confirmá con la autora:
- **Target encoding antes del split:** `origen_enc` y `destino_enc` se calculan sobre toda la ABT → leakage. El propio canvas lo marca como riesgo. Tiene que calcularse solo con train (idealmente dentro del pipeline / CV).
- **Split aleatorio con features temporales:** hay lags por fecha, pero el split es aleatorio y estratificado. Para un caso de uso con pronóstico a futuro, conviene evaluar un **split temporal** (entrenar con meses anteriores y testear con meses posteriores).
- **Ventana de 7 días vs. features del día anterior:** el canvas plantea revisar vuelos con hasta 7 días de anticipación, pero las lags usan `shift(1)` (datos hasta ayer), `retraso_previo_ruta` usa el vuelo anterior de la ruta y `vuelos_dia_origen` cuenta vuelos del mismo día. Hay que revisar qué información estaría realmente disponible al momento de predecir.
- **Clima observado vs. pronóstico:** riesgo ya reconocido en el canvas. Mencionarlo en las limitaciones o simular ruido de pronóstico.
- **`reportar_metricas` calcula ROC-AUC con predicciones binarias** en lugar de probabilidades. El AUC correcto se calcula aparte con `predict_proba`.
- **Selección de variables:** la consigna (FASE 3) pide aplicar técnicas de selección de variables. Hoy la selección es manual (leakage + redundancia). Falta una técnica formal (por ejemplo, mutual information, RFE, permutation importance o SHAP).
- **Interpretabilidad:** la consigna menciona SHAP. Hoy solo hay `feature_importances_`.

## 5. Consigna: fases y qué se evalúa

| Fase | Qué pide | Criterios clave |
|---|---|---|
| 0 – Lanzamiento | Tema, contexto, casos inspiradores | Relevancia del problema |
| 1 – Producto y planificación | Canvas (✅ hecho), presentación de 5 a 10 slides, roadmap actualizado mensualmente | Coherencia problema–hipótesis–datos–solución. **Al cierre:** desvíos plan vs. real, tareas replanificadas, riesgos enfrentados |
| 2 – Datos y preparación | Selección de datasets, calidad, limpieza, outliers, imputación, encoding, escalado | Exhaustividad y **justificación técnica**; deseable un mínimo de 2 fuentes (✅ vuelos + clima) |
| **3 – EDA y KPI design** | Análisis del target, ventanas de observación, EDA visual avanzado, multivariado, hipótesis, **selección de variables** | Exhaustividad con foco en el problema, calidad y **tipo correcto de gráfico**, justificación de variables |
| 4 – Modelo | Enfoque, entrenamiento, tuning de hiperparámetros, interpretación (SHAP, importancia), métricas (AUC, F1, precision, recall), conclusiones y mejoras | Adecuación al problema, justificación de train/test/validación |
| 5 – Validación y métricas estratégicas | Integración del modelo en el producto, KPIs de negocio, **A/B o simulaciones** de impacto | Impacto medible y trazable |
| 6 – Visualización y storytelling | Dashboard interactivo en **Looker** (u otra herramienta validada), narrativa centrada en insights | Claridad, storytelling |
| 7 – Despliegue y ética | Riesgos operativos, técnicos, legales y éticos, mitigación, sesgos, gobernanza, mantenimiento | Exhaustividad |
| 8 – Documentación | Documentación técnica del modelo (plantilla de la cátedra, **obligatoria para aprobar**), repo estructurado y reproducible, reporte ejecutivo | Reproducibilidad, coherencia |
| 9 – Defensa | Presentación oral, demo funcional (deseable) | Defender decisiones técnicas |

## 6. 🎯 Prioridad: el EDA y ~5 insights

El EDA tiene mucho peso en la evaluación y se pide obtener **alrededor de 5 insights** claros. Al trabajar en el EDA:

- Cada insight tiene que ser **accionable y estar conectado al problema de negocio** (qué significa para operaciones o planeamiento), no ser solo una descripción del gráfico.
- Formato sugerido: **título del insight + gráfico que lo sostiene + número concreto + implicancia para el negocio o para el modelo.**
- Usá el gráfico adecuado para cada caso: barras para tasas por categoría, líneas para series temporales u ordinales, boxplots o violines para distribuciones por clase, heatmaps para cruces de dos categóricas (por ejemplo, hora × día). Ejes y títulos en español, con unidades, y ordenados con sentido.
- Comparar siempre contra la **tasa base de retraso** (~21%).
- Cerrar el EDA con una sección **"Insights principales"** que los resuma.

Candidatos que ya aparecen en el borrador (consolidarlos, validarlos con números y elegir los ~5 más fuertes):
1. **Efecto acumulativo por hora:** la tasa de retraso sube a lo largo del día, con pico entre las 17 y las 21 h (las demoras se arrastran).
2. **Estacionalidad:** picos en meses de alta demanda (junio, julio, diciembre) y los viernes con la mayor tasa semanal. Además, el volumen por sí solo no explica la tasa.
3. **Aerolínea:** diferencias marcadas en la tasa de retraso entre aerolíneas.
4. **Aeropuerto + clima:** aeropuertos con más lluvia y viento (por ejemplo MDW, BWI, DAL) tienen tasas más altas, aunque la correlación es débil o moderada.
5. **Causas operativas > clima:** según las columnas de causa de demora, predominan las causas de aerolínea y de avión tardío frente a las climáticas.

Ideas para profundizar: heatmap hora × día de la semana, tasa por ruta, análisis del target por bloque de features (los bloques del canvas) y validación estadística de hipótesis (chi², tests de proporciones).

## 7. Datos y archivos

| Archivo | Descripción | En git |
|---|---|---|
| `flights_sample_3m.csv` | Dataset original de Kaggle (~600 MB) | ❌ (en `.gitignore`) |
| `flights_2022_es.csv` | Filtro 2022 con columnas en español (~140 MB) | ❌ (en `.gitignore`) |
| `clima_2022_por_aeropuerto.parquet` | Caché del clima diario de Open-Meteo por aeropuerto | ✅ |
| `dictionary.md` / `dictionary.html` | Diccionario de datos de las columnas del dataset de vuelos | ✅ |
| `tp.ipynb` | Notebook principal (primera versión) | ✅ |
| `generated/` | Carpeta para salidas generadas | — |

**Variables con leakage** (solo se conocen después del vuelo, **nunca** van al modelo, aunque se pueden usar en el EDA): `hora_salida_real`, `retraso_salida`, `tiempo_rodaje_salida`, `despegue`, `aterrizaje`, `tiempo_rodaje_llegada`, `hora_llegada_real`, `retraso_llegada`, `cancelado`, `codigo_cancelacion`, `desviado`, `duracion_real`, `tiempo_en_aire`, `retraso_por_*`.

## 8. Reglas de trabajo para el agente

**Idioma y estilo**
- Todo en **español** (rioplatense, registro académico claro): markdown, comentarios, nombres de variables y títulos de gráficos. Seguí la convención actual (`snake_case` en español: `df_2022`, `tasa_retraso`, `abt_final`).
- Las celdas de markdown tienen que **explicar el porqué** de cada decisión. La consigna evalúa la justificación técnica, no solo el código.

**Datos**
- **No modificar ni sobrescribir** `flights_sample_3m.csv`, `flights_2022_es.csv` ni el parquet de clima. Las salidas nuevas van a `generated/`.
- No cargues el CSV de 3M filas si no hace falta: es lento y pesado. Trabajá sobre `flights_2022_es.csv` y, para pruebas rápidas, sobre una muestra.
- No hagas llamadas a APIs externas si el resultado ya está cacheado.
- Mantené `random_state=42` en todo lo que tenga aleatoriedad.

**Rigor**
- **Cero leakage:** todo lo que use el target (encodings, imputaciones con estadísticos, escalado, selección de variables) se ajusta **solo con train**, idealmente dentro de un `Pipeline`.
- Cada número que aparezca en un markdown tiene que salir de una celda ejecutada. **No inventes resultados**: si no corriste el código, decilo.
- Si cambiás algo que altera los resultados (muestra, features, split), actualizá también los textos e interpretaciones que dependan de ellos.
- Respetá el canvas aprobado: los KPIs técnicos son Recall (prioritario), Precision y ROC-AUC.

**Forma de trabajar**
- Hacé cambios **chicos y revisables**: una sección o un problema por vez. Preferí editar celdas puntuales antes que reescribir el notebook entero.
- Antes de reestructurar mucho (mover código a `src/`, cambiar el orden de las secciones), proponé el plan y esperá confirmación.
- Ante una duda de criterio de negocio o de la consigna, **preguntá** en lugar de suponer.
- La autora tiene que poder **defender oralmente** cada decisión: explicá brevemente lo que hiciste y por qué.

## 9. Entorno

- Python 3 (local: 3.13), Jupyter Notebook, en Windows.
- Librerías: `pandas`, `numpy`, `matplotlib`, `seaborn`, `pyarrow`, `requests`, `scikit-learn`, `xgboost`.
- Dependencias con versiones fijas en `requirements.txt`. Todos los imports y constantes (`RANDOM_STATE`, `UMBRAL_RETRASO`, nombres de archivos) están en la sección "1. Configuración" del notebook: usalos en lugar de repetir valores.
- Pendiente: estructura de repo reproducible (FASE 8).
