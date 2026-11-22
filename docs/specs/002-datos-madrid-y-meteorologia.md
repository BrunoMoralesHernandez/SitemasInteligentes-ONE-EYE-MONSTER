# Spec 002 — Datos de Madrid y meteorología

**Estado:** Borrador para revisión  
**Referencias:** [PLAN.md](../PLAN.md), [CONSTITUTION.md](../CONSTITUTION.md), [Spec 001](001-arquitectura-y-contratos.md)

## Guía rápida (en 1 minuto)

Este documento explica cómo pasamos de los ficheros brutos a una **tabla limpia por sensor y hora**:

- **Los datos originales no se tocan:** se guardan tal cual en `data/` con su huella SHA-256. El resultado limpio se escribe aparte en Parquet particionado por año y mes.
- **Filtro temprano de tráfico:** leemos los CSV mensuales por bloques (*chunks*) para no saturar la RAM. Solo nos quedamos con sensores `M30` y filas sin fallo (`error == 'N'`). Cualquier número negativo se convierte en `NaN`.
- **De 15 minutos a 1 hora:** cada hora se resume calculando la **media** de sus cuartos de hora válidos (nunca la suma, porque `intensidad` ya viene en vehículos/hora) y anotando cuántos cuartos válidos hubo (`valid_15min_observations`).
- **Clima híbrido (desde 2019):** para entrenar usamos primero las estaciones meteorológicas del Ayuntamiento de Madrid más cercanas al sensor, y rellenamos con Open-Meteo Archive lo que el Ayuntamiento no mide (`rain`, `visibility`, `weather_code`) o los huecos puntuales. Para predecir el futuro usamos siempre Open-Meteo Forecast.
- **Caché obligatoria:** toda petición a Open-Meteo pasa por disco. El histórico guardado no caduca; las previsiones futuras caducan y nunca se reutilizan en silencio si están obsoletas (*stale*).

---

## 1. Objetivo y alcance

Definir la ingesta, limpieza, agregación horaria, cruce geográfico/temporal con meteorología y caché local para construir el dataset horario integrado de la M30 sin superar el presupuesto de memoria local.

Componentes incluidos:
- `MadridTrafficLoader`, `MadridSchemaNormalizer` (`data/madrid_traffic.py`)
- `MadridSensorCatalogLoader` (`data/sensor_catalog.py`)
- `HourlyTrafficAggregator` (`data/aggregation.py`)
- `MadridWeatherLoader`, `MadridWeatherStationCatalog`, `NearestWeatherStationResolver` (`data/madrid_weather.py`)
- `WeatherCache`, `FileWeatherCache` (`data/weather_cache.py`)
- `OpenMeteoClient` (`data/open_meteo.py`)
- `MadridDatasetBuilder` (`data/dataset_builder.py`)

## 2. Fuera de alcance

- Cálculo de etiquetas (`TrafficState`, demora P90), división `train/validation/test` y features de calendario (ver [Spec 003](003-etiquetas-y-features.md)).
- Integración de años anteriores a **enero de 2019** (fecha de inicio del histórico meteorológico municipal).
- Imputar o inventar valores de tráfico (`road_load`, `avg_speed_kmh`) o deducir `visibility` / `weather_code` a partir de datos municipales no equivalentes.

## 3. Fuentes oficiales y esquemas de datos

### 3.1 Fuentes de datos

| Fuente | Uso | Enlace oficial |
|---|---|---|
| Histórico de tráfico (15 min) | Medidas de `intensidad`, `ocupacion`, `carga`, `vmed`, `error` | [datos.madrid.es — Histórico puntos de medida](https://datos.madrid.es/dataset/208627-0-transporte-ptomedida-historico) |
| Catálogo de puntos de medida | Nombre y coordenadas de sensores M30 | [datos.madrid.es — Intensidad de tráfico](https://datos.madrid.es/dataset/202468-0-intensidad-trafico) |
| Meteorología municipal horaria | Observaciones locales desde 2019 | [datos.madrid.es — Meteorológicos horarios](https://datos.madrid.es/dataset/300352-0-meteorologicos-horarios) |
| Estaciones meteorológicas | Ubicación de estaciones municipales | [datos.madrid.es — Estaciones meteorológicas](https://datos.madrid.es/dataset/300360-0-meteorologicos-estaciones) |
| Open-Meteo Archive / Forecast | Complemento histórico y previsión futura | [open-meteo.com/en/docs](https://open-meteo.com/en/docs) |

Cada archivo descargado se registra en un manifiesto local con su URL exacta, fecha de descarga y hash `SHA-256`.

### 3.2 Tablas de salida (`DataFrame`)

Los nombres de columna, tipos y unidades siguen las tablas de `PLAN.md` (secciones 4.1 a 4.3):

1. **Tráfico normalizado (15 min):** `sensor_id` (`int32`), `observed_at` (`datetime64[ns, Europe/Madrid]`), `intensity_vph`, `occupancy_pct`, `road_load`, `avg_speed_kmh` (`float32`), `quality_status` (`category`, solo `'N'`) e `integration_samples` (`Int16`).
2. **Catálogo de sensores:** `sensor_id` (`int32`), `central_code` (`string`), `sensor_name` (`string`), `utm_x`, `utm_y`, `latitude`, `longitude` (`float64`).
3. **Dataset horario integrado** (clave única `sensor_id` + `hour_utc`):
   - **Tráfico:** `local_datetime`, medias horarias de las 4 variables y `valid_15min_observations` (`Int8`, de 1 a 4).
   - **Clima:** `temperature_c`, `precipitation_mm`, `rain_mm`, `visibility_m`, `wind_speed_kmh` (`float32`) y `weather_code` (`Int16`).
   - **Trazabilidad:** `weather_source` (`madrid_municipal` u `open_meteo`), `source_location_id`, `weather_match_distance_minutes` (`Int16`) y `station_distance_km` (`float32`).

## 4. Reglas de procesamiento y casos de error

### 4.1 Limpieza y agregación del tráfico

1. **Lectura por bloques:** se procesa archivo a archivo y por *chunks* sin cargar años enteros en memoria. Se conserva el borde entre bloques para no partir una hora a la mitad.
2. **Variantes de columnas:** `MadridSchemaNormalizer` unifica `id`/`idelem` → `sensor_id`. Si un fichero no permite identificar sensores `M30` o carece de columna obligatoria, se rechaza con error explícito.
3. **Filtro de calidad:** se descartan de inmediato filas no `M30` o con `error != 'N'`. En las 4 medidas numéricas, cualquier valor `< 0` pasa a `NaN` (el valor `0` sí se conserva en esta fase).
4. **Duplicados `(sensor_id, instante_utc)`:** si son copias idénticas, se deja una; si tienen valores distintos en conflicto, se descartan ambas y se anota en el informe de calidad.
5. **Conversión a UTC y agregación:**
   - La fecha local (`Europe/Madrid`) se convierte a UTC antes de agrupar por hora. Las horas locales inexistentes o ambiguas que no puedan desambiguarse se ponen en cuarentena.
   - *Nota previa a H2:* al perfilar las muestras reales se confirmará si la marca de 15 minutos indica inicio o fin del cuarto de hora y se fijará en configuración.
   - Para cada `(sensor_id, hour_utc)` se calcula la **media aritmética** de los valores válidos de cada magnitud y se cuenta `valid_15min_observations`.
   - Solo son elegibles las horas con `valid_15min_observations >= min_valid_15min_observations` (umbral configurable de 1 a 4, fijado tras el análisis de calidad inicial).

### 4.2 Cruce meteorológico híbrido

1. **Asignación geográfica:** cada sensor M30 se asocia a su estación municipal más cercana con coordenadas válidas (`NearestWeatherStationResolver`), guardando la distancia en `station_distance_km`.
2. **Prioridad de fuentes en histórico:**
   - Se usan primero las variables medidas por la estación municipal (`temperature_c`, `precipitation_mm`, `wind_speed_kmh`).
   - Open-Meteo Archive rellena las variables que el Ayuntamiento no ofrece (`rain_mm`, `visibility_m`, `weather_code`) y los huecos puntuales.
   - **Prohibido mezclar conceptos:** la precipitación municipal nunca se copia en `rain_mm`, ni se inventan `visibility_m` o `weather_code`.
   - Si una fila usa al menos un dato municipal, `weather_source` es `madrid_municipal` (y se guarda qué campo vino de qué fuente en el informe/linaje del dataset); si todo procede de Open-Meteo, es `open_meteo`.
3. **Tolerancia temporal:** el cruce por `hour_utc` exige una diferencia máxima de **30 minutos** (`weather_match_distance_minutes <= 30`). En caso de empate, se prefiere el instante anterior.
4. **Cobertura mínima (Hito H2):** al menos el **95 %** de las horas de tráfico elegibles deben quedar con su contexto meteorológico completo. Si es menor, H2 se detiene hasta revisar y aprobar una excepción documentada.

### 4.3 Caché de Open-Meteo (`FileWeatherCache`)

- **Clave única (SHA-256):** combina tipo (`archive` vs `forecast`), coordenadas, rango UTC, lista de variables, unidades y versión de esquema.
- **Escritura atómica:** se escribe en un fichero temporal y se renombra para evitar JSON corruptos a mitad de escritura.
- **Caducidad:**
  - `archive` (histórico cerrado): no caduca mientras coincidan clave y checksum.
  - `forecast` (previsión futura): caduca según `forecast_ttl_minutes` (por defecto 15 min). Si una previsión ha caducado y falla la red al renovarla, se lanza error de clima no disponible; **nunca se devuelve una previsión caducada como si fuera actual**.

### 4.4 Resumen de errores y cuarentena

| Problema detectado | Qué hace el sistema |
|---|---|
| Hash SHA-256 distinto al del manifiesto o columna obligatoria ausente | Aborta la lectura de ese archivo. |
| Registro no `M30` o `error != 'N'` | Se descarta antes de acumular en memoria. |
| Medida de tráfico negativa (`< 0`) | Se convierte a `NaN` en esa columna. |
| Duplicado con valores contradictorios o fecha DST ambigua sin resolver | Se excluye esa clave horaria y se registra en el informe de calidad. |
| Cobertura meteorológica `< 95 %` sobre horas elegibles | Bloquea el cierre del hito H2. |
| Caché corrupta o forecast caducado sin red disponible | Invalida la entrada y falla con error explícito (`WeatherUnavailableError` en consulta). |

## 5. Dependencias

- Stack aprobado: Python 3.10+, `pandas`, `numpy`, biblioteca estándar (`hashlib`, `json`, `zoneinfo`, `pathlib`, `urllib`).
- Cualquier librería adicional para conversión de coordenadas UTM a latitud/longitud (como `pyproj`) o motor Parquet no presente en el entorno requerirá autorización explícita del usuario antes de instalarse.

## 6. Ejemplo práctico: de 15 minutos a 1 hora

Cuatro lecturas de un sensor M30 el `2024-01-15` entre las `08:00` y las `08:45` (`Europe/Madrid`, que corresponde a `07:00 UTC`), todas con `error == 'N'`:

| Hora local | `intensity_vph` | `occupancy_pct` | `road_load` | `avg_speed_kmh` |
|---|---:|---:|---:|---:|
| 08:00 | 400 | 10 | 20 | 50 |
| 08:15 | 800 | 20 | 40 | 40 |
| 08:30 | 1200 | 30 | 60 | 30 |
| 08:45 | 600 | 15 | 30 | -1 *(ausente)* |

Resultado agregado para `hour_utc = 2024-01-15 07:00Z`:
- `intensity_vph` = `(400 + 800 + 1200 + 600) / 4` = **750.0** (media, no 3000).
- `occupancy_pct` = **18.75**
- `road_load` = **37.5**
- `avg_speed_kmh` = `(50 + 40 + 30) / 3` = **40.0** (el `-1` se convirtió en `NaN` y no promedia como número).
- `valid_15min_observations` = **4**.

## 7. Tests obligatorios

1. **Filtrado y negativos:** descarta filas con `tipo_elem != 'M30'` o `error != 'N'`, convierte negativos a `NaN` y mantiene los ceros.
2. **Variantes y lectura por chunks:** unifica `id`/`idelem` y produce exactamente el mismo agregado horario leyendo un archivo de golpe o partido en chunks que cortan una hora por la mitad.
3. **Duplicados:** elimina duplicados exactos sin inflar `valid_15min_observations` y envía a cuarentena los duplicados con valores contradictorios.
4. **Agregación por medias:** verifica con el ejemplo de la sección 6 que `intensity_vph` se promedia y que un `NaN` en `avg_speed_kmh` solo reduce el denominador de esa columna.
5. **Zonas horarias y DST:** convierte `Europe/Madrid` a `hour_utc` manteniendo separadas las dos horas UTC del cambio de otoño y rechazando horas locales inexistentes de primavera.
6. **Cruce meteorológico y separación de variables:** asigna la estación más cercana, respeta la tolerancia de $\le 30$ minutos, no copia precipitación municipal en `rain_mm` y deja `visibility_m`/`weather_code` en `NaN` si Open-Meteo no los aporta.
7. **Cálculo de cobertura ($\ge 95\\%$):** calcula el porcentaje sobre las horas de tráfico elegibles sin eliminar del denominador las horas en las que falle el clima.
8. **Caché Open-Meteo:** comprueba con reloj y red simulados que un *hit* vigente no hace peticiones HTTP, que `archive` y `forecast` tienen claves/caducidad distintas, que una escritura a medias no corrompe el archivo anterior y que un `forecast` caducado no se devuelve como vigente.

## 8. Criterios de aceptación

- [ ] **CA-002-1:** Existe un manifiesto con URL, fecha y SHA-256 de los archivos crudos utilizados, y los originales en `data/` permanecen intactos.
- [ ] **CA-002-2:** La ingesta procesa los datos por meses/chunks y genera el dataset horario en Parquet particionado cumpliendo los esquemas y tipos de la sección 3.2.
- [ ] **CA-002-3:** Queda documentada con evidencia de los datos reales la convención de marca temporal de 15 minutos y el valor elegido para `min_valid_15min_observations`.
- [ ] **CA-002-4:** El informe de calidad del hito H2 acredita una cobertura meteorológica $\ge 95\\%$ sobre registros elegibles (o cuenta con una excepción documentada y aprobada por el usuario).
- [ ] **CA-002-5:** Todos los tests de la sección 7 pasan en local sin realizar llamadas reales a Internet.
