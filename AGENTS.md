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

**Diferencias entre el canvas y la implementación actual** (hay que actualizarlas en el canvas del informe final, no cambiarlas en el código):
- **Horizonte de predicción: 5 días**, no 7. Todas las features que dependen del resultado de otros vuelos usan datos hasta D−6.
- **La ABT tiene 41 features** (38 después de RFECV), no 31, organizadas en tres bloques: programación del vuelo, historial de retrasos y clima.
- **El target encoding se reemplazó** por tasas históricas dinámicas calculadas solo con vuelos de train (ver sección 4).
- **Acciones del semáforo:** la reubicación preventiva no quedó en ningún nivel, porque según la simulación de impacto no se paga sola.

## 3. Datos operativos que NO tenemos: se eligieron datos sintéticos

La solución del canvas incluye acciones como **reubicar pasajeros, reprogramar vuelos y reasignar tripulación**, y KPIs de costos de compensación, reubicación y operación. **El dataset no tiene esos datos**: no hay pasajeros, conexiones, tripulaciones, costos ni NPS.

La profesora habilitó dos alternativas (solución teórica o datos sintéticos) y **la autora eligió datos sintéticos**. Están implementados en `simulacion_impacto.ipynb` (FASE 5, ver sección 4.2).

**Reglas para los datos sintéticos:**
- Tienen que estar **claramente separados y etiquetados**: carpeta `generated/sintetico/`, prefijo `sintetico_` y semilla fija (`RANDOM_STATE = 42`). Nunca se mezclan con los datos reales.
- Cada supuesto tiene que estar en la celda de configuración del notebook de simulación, con su fuente. Si no tiene fuente directa, se marca como **supuesto** y se cubre con análisis de sensibilidad.
- Los resultados de la simulación se presentan como **estimaciones del impacto posible, no como resultados observados**.
- No cambies los supuestos de costo o efectividad para favorecer una política. Si hace falta cambiarlos, consultalo con la autora.

## 4. Estado actual

Los dos notebooks están completos en su primera versión y se siguen iterando. Todavía no hay código en módulos `.py`. Moverlo a `src/` queda pendiente para la FASE 8 y requiere confirmación antes de hacerlo.

### 4.1 `tp.ipynb` (notebook principal)

1. **Configuración:** imports y constantes (`RANDOM_STATE`, `UMBRAL_RETRASO`, `PROPORCION_TEST`, `SUAVIZADO_TASAS`, `UMBRAL_ALERTA`, nombres de archivos).
2. **Datos:**
   - Se carga `flights_2022_es.csv`. Si no existe, se genera desde el CSV de Kaggle.
   - Se toma una muestra del 25% estratificada por mes y se filtran los 92 aeropuertos de origen que concentran el 90% de los vuelos.
   - El clima viene de Open-Meteo: coordenadas de OurAirports (o su mirror en GitHub) y caché en parquet.
   - Target: `retrasado = retraso_salida > 15`. Los cancelados y desviados se analizan aparte y se excluyen del target y del modelo. Se conservan solo para contar la congestión programada.
3. **Calidad:** tipos de variables, separación de variables con leakage y outliers. Los outliers no se tratan, con justificación por grupo de variables.
4. **EDA:**
   - target, univariado, causas de la demora, bivariado (hora, día, mes, hora × día, aerolínea, aeropuerto, clima) y resumen con WOE, IV y KS;
   - multivariado: correlaciones, hora × modelo de negocio, aerolínea × aeropuerto y lluvia × franja;
   - cierra con **5 insights principales** (4.5).
5. **ABT:**
   - **Horizonte de 5 días.** El split train/test (aleatorio, estratificado, 75/25, columna `es_test`) se define **antes** de construir las features.
   - Features de calendario y congestión programada, más las interacciones `lowcost_x_hora` y `lluvia_x_hora`.
   - **6 tasas históricas dinámicas**, calculadas **solo con vuelos de train hasta D−6** y con suavizado m=20:
     - 4 acumuladas: origen, destino, ruta y aerolínea × origen;
     - 2 recientes, de 7 días: aerolínea y origen.
   - Control de leakage de las tasas (5.5): recálculo a mano de una muestra e inversión del target de test.
   - One-hot de aerolínea y EDA de las features construidas, solo con train (5.7).
   - Resultado: 41 features.
6. **Modelado:**
   - **Selección de variables con RFECV** (XGBoost, ROC-AUC): conserva 38 de las 41 variables.
   - Modelos comparados: regresión logística (con y sin balanceo), XGBoost, LightGBM, MLP y un voting classifier (este último no entra en la elección). Todos usan `RandomizedSearchCV` con recall y la misma CV de 5 folds.
   - **Modelo elegido: XGBoost (RFECV)**, elegido con la CV y no con el test. En test: ROC-AUC 0,710, KS 0,304 y lift de 2,44 en el decil 1.
   - También incluye: aporte del feature engineering por bloques, deciles, ganancia, lift, KS y control de overfitting con curva de aprendizaje.
   - **Semáforo de 4 niveles**, con cortes elegidos en validación:
     - verde: menos de 0,40;
     - amarillo: 0,40 a 0,60, con aviso;
     - naranja: 0,60 a 0,80, con aviso y monitoreo prioritario;
     - rojo: 0,80 o más, con aviso y reasignación de recursos.
     
     El corte de 0,80 sale de la simulación. Con alerta desde amarillo, el recall en test es 0,783.
   - Exporta `generated/predicciones_test.csv` y `generated/predicciones_validacion.csv`, que usa la simulación.
   - **SHAP:** importancia global, dirección de los efectos y `factores_vuelo` (los 3 factores de cada alerta, para el dashboard).
   - **Validación fuera de la ventana con enero a agosto de 2023:**
     - Usa las funciones `preparar_vuelos` y `construir_abt`, verificadas contra la ABT de 2022.
     - Evalúa sin reentrenar.
     - Calcula el PSI del score y de las features.
7. **Conclusión** con limitaciones y próximos pasos.

### 4.2 `simulacion_impacto.ipynb` (FASE 5, datos sintéticos)

Estima el ahorro del semáforo frente a gestionar las demoras recién cuando ocurren.

- **Datos:**
  - Usa las predicciones reales del modelo.
  - Genera pasajeros por vuelo (binomial, 81,5% de ocupación) y pasajeros con conexión (20,5%).
  - `retraso_salida` y las causas de la demora se usan solo para **medir el costo** de cada demora, nunca para predecir.
- **Costos:**
  - operativo: USD 100,76 por minuto;
  - compensación: solo en demoras controlables, según los compromisos de cada aerolínea vigentes en 2024-2025;
  - reubicación reactiva.
  - Las acciones del semáforo se pagan también en las falsas alarmas.
- **Secciones 4 a 7, diseño inicial de 3 niveles** (aviso desde 0,40; reasignación y reubicación preventiva desde 0,60):
  - El escenario base **pierde dinero**.
  - Calcula los KPIs estratégicos del canvas, las horas de espera evitadas y el ingreso futuro en riesgo. Estos dos últimos son indicadores aparte y no se suman al ahorro.
  - Incluye sensibilidad a la efectividad de anticiparse y al costo de las acciones.
- **Sección 8, comparación de políticas:**
  - Las políticas se comparan **con las predicciones de validación**. Gana la de 4 niveles con reasignación desde 0,80 y sin reubicación preventiva.
  - Esa política se mide **una sola vez** en test.
  - De esta comparación sale el semáforo final de `tp.ipynb`.
- **Sección 9:** las conclusiones están pendientes de redactar; las limitaciones ya están escritas.

### 4.3 Pendientes y mejoras conocidas
- **Simulación:**
  - Redactar las conclusiones (sección 9).
  - En 2023 la precisión del rojo bajó a 56%, y la reasignación se pagaba sola con 61%. Falta evaluar si sigue siendo rentable.
- **Dashboard (FASE 6):** agregar los factores de `factores_vuelo` como columna en las predicciones exportadas y armar el dashboard en Looker (u otra herramienta validada).
- **Mejoras del modelo, documentadas pero sin aplicar:** no se aplican porque se decidirían mirando el test o 2023.
  - entrenar con varios años;
  - tasas acumuladas con ventana móvil de 365 días;
  - pronóstico del clima en lugar de clima observado;
  - calibrar el score, que hoy ordena el riesgo pero no es una probabilidad calibrada;
  - quitar `es_low_cost`, que tiene SHAP 0.
- **Fases 7 a 9:** riesgos y ética, monitoreo (PSI mensual, recall, ROC-AUC y tasa del nivel rojo), documentación con la plantilla de la cátedra, repo reproducible y reporte ejecutivo.
- **Informe final:** actualizar el canvas (ver las diferencias en la sección 2).

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

Insights consolidados en `tp.ipynb` (sección 4.5):
1. **El retraso se acumula a lo largo del día, sobre todo en las low cost:** la hora es la variable con más señal (IV 0,234), y en las low cost la tasa sube del 15,4% al 45,6% de la mañana a la noche.
2. **Parte del "efecto aeropuerto" es en realidad un efecto aerolínea:** BWI tiene 31,9% de retrasos, pero sin Southwest baja a 14,5%. Es una variable de confusión.
3. **Las demoras son operativas, no climáticas:** aerolínea y avión tardío explican el 78% de los minutos de demora; el clima extremo, el 5%.
4. **La lluvia sola pesa poco, pero combinada con la hora marca el mayor riesgo:** de noche con lluvia fuerte se demora el 46,4% de los vuelos.
5. **El calendario marca temporadas, pero no alcanza para predecir:** va del 15,9% en octubre al 25,7% en diciembre, con V de Cramér débil.

Si se modifica el EDA, mantené este formato y verificá que los números de los insights sigan coincidiendo con las celdas ejecutadas.

## 7. Datos y archivos

| Archivo | Descripción | En git |
|---|---|---|
| `flights_sample_3m.csv` | Dataset original de Kaggle (~600 MB) | ❌ (en `.gitignore`) |
| `flights_2022_es.csv` | Filtro 2022 con columnas en español (~140 MB) | ❌ (en `.gitignore`) |
| `clima_2022_por_aeropuerto.parquet` | Caché del clima diario de Open-Meteo por aeropuerto | ✅ |
| `dictionary.md` / `dictionary.html` | Diccionario de datos de las columnas del dataset de vuelos | ✅ |
| `tp.ipynb` | Notebook principal: datos, EDA, ABT, modelado, semáforo, SHAP y validación con 2023 | ✅ |
| `simulacion_impacto.ipynb` | Simulación de impacto con datos sintéticos (FASE 5) | ✅ |
| `README.md` | Descripción del repo y cómo correrlo | ✅ |
| `generated/flights_2023_es.csv` | Vuelos de 2023 para la validación fuera de la ventana (se regenera desde el CSV de Kaggle) | ❌ |
| `generated/clima_2023_por_aeropuerto.parquet` | Caché del clima de 2023 | ✅ |
| `generated/predicciones_test.csv` / `predicciones_validacion.csv` | Predicciones y nivel del semáforo que exporta `tp.ipynb` (6.11) y que usa la simulación | ❌ (se regeneran) |
| `generated/sintetico/sintetico_*.csv` | Pasajeros sintéticos por vuelo (test y validación) | ❌ (se regeneran) |

**Orden de ejecución:** primero `tp.ipynb`, que genera las predicciones; después `simulacion_impacto.ipynb`.

**Variables con leakage** (solo se conocen después del vuelo, **nunca** van al modelo, aunque se pueden usar en el EDA): `hora_salida_real`, `retraso_salida`, `tiempo_rodaje_salida`, `despegue`, `aterrizaje`, `tiempo_rodaje_llegada`, `hora_llegada_real`, `retraso_llegada`, `cancelado`, `codigo_cancelacion`, `desviado`, `duracion_real`, `tiempo_en_aire`, `retraso_por_*`.

## 8. Reglas de trabajo para el agente

**Idioma y estilo**
- Todo en **español** (rioplatense, registro académico claro): markdown, comentarios, nombres de variables y títulos de gráficos. Seguí la convención actual (`snake_case` en español: `df_2022`, `tasa_retraso`, `abt_final`).
- Las celdas de markdown tienen que **explicar el porqué** de cada decisión. La consigna evalúa la justificación técnica, no solo el código.

**Datos**
- **No modificar ni sobrescribir** `flights_sample_3m.csv`, `flights_2022_es.csv` ni los parquet de clima. Las salidas nuevas van a `generated/`, y las sintéticas, a `generated/sintetico/`.
- **Los datos de 2023 no se usan para decidir nada** (features, tuning, umbrales, modelo). Si se mira 2023 y se cambia algo, deja de ser una validación fuera de la ventana. Lo mismo vale para el test: las decisiones se toman con la CV o con las predicciones de validación.
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
- Librerías: `pandas`, `numpy`, `matplotlib`, `seaborn`, `pyarrow`, `requests`, `scikit-learn`, `scipy`, `xgboost`, `lightgbm`, `shap`.
- Dependencias con versiones fijas en `requirements.txt`. Los comentarios del archivo explican cómo crear el entorno y el kernel "Python (TP IA)".
- Todos los imports y constantes (`RANDOM_STATE`, `UMBRAL_RETRASO`, nombres de archivos) están en la sección "1. Configuración" de cada notebook: usalos en lugar de repetir valores.
- Pendiente: estructura de repo reproducible (FASE 8).
