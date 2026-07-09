## Dataset: Flight Take-Off Data — JFK Airport

**28,820 vuelos** desde el aeropuerto JFK (Nueva York) hacia **65 destinos**, operados por **9 aerolíneas**. Los datos combinan información operativa del vuelo con condiciones meteorológicas al momento del despegue.

### Variables clave

| Grupo | Columnas | Qué representan |
|---|---|---|
| **Temporal** | `MONTH`, `DAY_OF_MONTH`, `DAY_OF_WEEK` | Cuándo sale el vuelo (meses 1-12, con concentración en nov-dic) |
| **Vuelo** | `OP_UNIQUE_CARRIER`, `TAIL_NUM`, `DEST` | Aerolínea (B6/JetBlue domina con ~33%), avión específico, destino (LAX es el top) |
| **Horarios** | `CRS_DEP_M`, `DEP_TIME_M`, `CRS_ARR_M`, `sch_dep`, `sch_arr` | Horarios programados y reales en minutos desde medianoche |
| **Delay** | `DEP_DELAY` | **Variable objetivo probable** — demora en minutos (negativo = salió antes, media ~6 min, max 1276 min!) |
| **Ruta** | `DISTANCE`, `CRS_ELAPSED_TIME` | Distancia en millas y duración programada |
| **Clima** | `Temperature`, `Dew Point`, `Humidity`, `Wind`, `Wind Speed`, `Wind Gust`, `Pressure`, `Condition` | Condiciones meteorológicas al despegue (25 tipos de clima distintos) |
| **Taxi** | `TAXI_OUT` | Tiempo de rodaje en pista (media ~21 min) |

### Observaciones rápidas

- **Casi sin nulos** (solo 2 en `Wind`) — dataset muy limpio
- `Dew Point` es `object` en vez de numérico — hay que limpiarlo
- `DEP_DELAY` tiene outliers extremos (hasta 1276 min = ~21 horas de demora)
- Los meses están concentrados en enero, noviembre y diciembre (percentil 25 = 1, mediana = 11)

### Cómo analizarlo — EDA sugerido

1. **Distribuciones**: histogramas de `DEP_DELAY`, `TAXI_OUT`, `Temperature`, `Wind Speed`
2. **Delay por categoría**: boxplots de delay por aerolínea, por condición climática, por día de la semana
3. **Correlaciones**: heatmap numérico (delay vs. clima, distancia, hora)
4. **Temporal**: delay promedio por mes, por hora del día
5. **Top destinos**: ¿cuáles tienen más delay?
6. **Clima**: ¿cómo impactan lluvia/nieve/niebla en los delays?
