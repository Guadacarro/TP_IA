# Diccionario de datos — Dataset de vuelos
| Columna | Columna (español) | Columna original | Tipo de dato | Descripción (español) |
|---|---|---|---|---|
| FL_DATE | Fecha de vuelo | FlightDate | object | Fecha del vuelo (aaaammdd) |
| AIRLINE_CODE | Código de aerolínea | Reporting_Airline | object | Código único de aerolínea. Cuando el mismo código fue utilizado por múltiples aerolíneas, se agrega un sufijo numérico para las anteriores, por ejemplo, PA, PA(1), PA(2). Usar este campo para análisis a lo largo de varios años. |
| DOT_CODE | Código DOT | DOT_ID_Reporting_Airline | int64 | Número de identificación asignado por el DOT de EE.UU. para identificar una aerolínea única. Se define como aquella que opera y reporta bajo el mismo certificado DOT, sin importar su código, nombre o empresa matriz. |
| FL_NUMBER | Número de vuelo | Flight_Number_Reporting_Airline | int64 | Número de vuelo |
| ORIGIN | Origen | Origin | object | Aeropuerto de origen |
| ORIGIN_CITY | Ciudad de origen | OriginCityName | object | Aeropuerto de origen, nombre de la ciudad |
| DEST | Destino | Dest | object | Aeropuerto de destino |
| DEST_CITY | Ciudad de destino | DestCityName | object | Aeropuerto de destino, nombre de la ciudad |
| CRS_DEP_TIME | Hora programada de salida | CRSDepTime | int64 | Hora de salida programada según CRS (hora local: hhmm) |
| DEP_TIME | Hora real de salida | DepTime | float64 | Hora de salida real (hora local: hhmm) |
| DEP_DELAY | Demora en salida | DepDelay | float64 | Diferencia en minutos entre la hora de salida programada y la real. Las salidas anticipadas se muestran con números negativos. |
| TAXI_OUT | Tiempo de rodaje (salida) | TaxiOut | float64 | Tiempo de rodaje en pista antes del despegue, en minutos |
| WHEELS_OFF | Despegue | WheelsOff | float64 | Hora en que las ruedas dejan el suelo (hora local: hhmm) |
| WHEELS_ON | Aterrizaje | WheelsOn | float64 | Hora en que las ruedas tocan el suelo (hora local: hhmm) |
| TAXI_IN | Tiempo de rodaje (llegada) | TaxiIn | float64 | Tiempo de rodaje en pista después del aterrizaje, en minutos |
| CRS_ARR_TIME | Hora programada de llegada | CRSArrTime | int64 | Hora de llegada programada según CRS (hora local: hhmm) |
| ARR_TIME | Hora real de llegada | ArrTime | float64 | Hora de llegada real (hora local: hhmm) |
| ARR_DELAY | Demora en llegada | ArrDelay | float64 | Diferencia en minutos entre la hora de llegada programada y la real. Las llegadas anticipadas se muestran con números negativos. |
| CANCELLED | Cancelado | Cancelled | float64 | Indicador de vuelo cancelado (1=Sí) |
| CANCELLATION_CODE | Código de cancelación | CancellationCode | object | Especifica el motivo de la cancelación |
| DIVERTED | Desviado | Diverted | float64 | Indicador de vuelo desviado (1=Sí) |
| CRS_ELAPSED_TIME | Duración programada | CRSElapsedTime | float64 | Duración programada del vuelo según CRS, en minutos |
| ELAPSED_TIME | Duración real | ActualElapsedTime | float64 | Duración real del vuelo, en minutos |
| AIR_TIME | Tiempo en el aire | AirTime | float64 | Tiempo de vuelo en el aire, en minutos |
| DISTANCE | Distancia | Distance | float64 | Distancia entre aeropuertos (millas) |
| DELAY_DUE_CARRIER | Demora por aerolínea | CarrierDelay | float64 | Demora atribuida a la aerolínea, en minutos |
| DELAY_DUE_WEATHER | Demora por clima | WeatherDelay | float64 | Demora atribuida al clima, en minutos |
| DELAY_DUE_NAS | Demora por sistema aéreo | NASDelay | float64 | Demora atribuida al Sistema Aéreo Nacional, en minutos |
| DELAY_DUE_SECURITY | Demora por seguridad | SecurityDelay | float64 | Demora atribuida a seguridad, en minutos |
| DELAY_DUE_LATE_AIRCRAFT | Demora por aeronave tardía | LateAircraftDelay | float64 | Demora atribuida a la llegada tardía de la aeronave, en minutos |