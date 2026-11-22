# Spec 003 — Etiquetas y features

**Estado:** Borrador para revisión  
**Referencias:** [PLAN.md](../PLAN.md), [CONSTITUTION.md](../CONSTITUTION.md), [Spec 001](001-arquitectura-y-contratos.md), [Spec 002](002-datos-madrid-y-meteorologia.md)

## Guía rápida (en 1 minuto)

Este documento define **qué tiene que aprender el modelo y con qué variables de entrada**, asegurando que no hagamos trampas usando información del futuro (*data leakage*):

1. **Primero se divide por fechas (`train`, `validation`, `test`):** el corte es cronológico y global para todos los sensores. Nunca se barajan horas al azar.
2. **Etiqueta 1 — Estado del tráfico (`TrafficStateLabeler`):** convierte la `road_load` (`carga`) horaria en 4 clases fijas: `0` Fluido (`<25`), `1` Moderado (`25–49`), `2` Denso (`50–74`) y `3` Retención (`>=75`).
3. **Etiqueta 2 — Demora por kilómetro (`DelayLabeler`):** compara el tiempo en recorrer **1 km** a la velocidad actual (`avg_speed_kmh`) frente al tiempo en flujo libre del sensor (su **percentil 90** de velocidad calculado **solo con `train`**). Sale en `min/km` y nunca es negativa.
4. **Solo pistas disponibles en el futuro:** para predecir una hora futura solo usamos `sensor_id`, hora/día cíclicos, calendario de festivos de Madrid y meteorología. **Quedan prohibidas** la intensidad, ocupación, carga o velocidad de esa misma hora (y tampoco usamos lags en el MVP).

---

## 1. Objetivo y alcance

Especificar la partición temporal, la generación de las dos etiquetas objetivo (`traffic_state` y `delay_minutes_per_km`) y el pipeline de transformación de variables (`TrafficFeaturePipeline`) que comparten el entrenamiento y la inferencia futura.

Componentes incluidos:
- `TrafficStateLabeler`, `DelayLabeler` (`features/targets.py`)
- `CyclicalFeatures`, `MadridCalendarFeatures` (`features/temporal.py`)
- `TrafficFeaturePipeline` (`features/pipeline.py`)

## 2. Fuera de alcance

- Limpieza de archivos crudos, agregación de 15 minutos a 1 hora y descarga/caché del clima (ver [Spec 002](002-datos-madrid-y-meteorologia.md)).
- Elección de algoritmos de ML, balanceo de clases durante el entrenamiento y evaluación de Macro F1 / MAE (ver [Spec 004](004-modelado-e-inferencia.md)).
- Variables con rezagos (*lags*), medias móviles de tráfico o predicción de rutas completas.

## 3. Entradas y salidas

- **Entrada en entrenamiento:** el `DataFrame` horario integrado del Spec 002 (unido por `sensor_id` y `hour_utc`).
- **Entrada en una consulta futura:** `sensor_id`, ventana `hour_utc`, calendario local para esas fechas y previsión meteorológica horaria (sin ninguna medida de tráfico contemporánea).
- **Salidas:**
  - Asignación de partición (`train`, `validation` o `test`) por fila.
  - Columna objetivo `traffic_state` (`0`, `1`, `2`, `3` o nulo si `road_load` no es válido).
  - Columna objetivo `delay_minutes_per_km` (`float >= 0.0` o nulo si `avg_speed_kmh <= 0`, es nula o el sensor no tiene referencia P90 en `train`).
  - Diccionario de velocidades de referencia $v^{\text{ff}}_s$ (P90 en km/h) por `sensor_id`.
  - Matriz de features transformada con columnas y orden fijos (`feature_names`).

## 4. Reglas de negocio, fórmulas y casos de error

### 4.1 División cronológica global (antes de cualquier `fit`)

Tras inventariar el dataset integrado se fijan cuatro fechas UTC en punto ($T_0 < T_1 < T_2 < T_3$) comunes para todos los sensores:

| Partición | Regla sobre `hour_utc` | Uso permitido |
|---|---|---|
| `train` | $T_0 \le \text{hour\_utc} < T_1$ | Único conjunto donde se hace `fit` (P90, imputadores, codificadores, escaladores y pesos). |
| `validation` | $T_1 \le \text{hour\_utc} < T_2$ | Solo `transform`; sirve para comparar modelos y ajustar hiperparámetros. |
| `test` | $T_2 \le \text{hour\_utc} < T_3$ | Solo `transform`; sellado para una única evaluación final en el Spec 004. |

**Orden obligatorio:** 1º ordenar por `(hour_utc, sensor_id)` sin barajar → 2º cortar en `train/validation/test` → 3º ajustar `DelayLabeler` y `TrafficFeaturePipeline` exclusivamente con `train` → 4º aplicar `transform` a los tres conjuntos.

### 4.2 Etiquetas de aprendizaje

#### A) Clasificación: `TrafficStateLabeler` (desde `road_load` horario)

Se evalúa sobre valores finitos en el rango físico $[0, 100]$ sin redondear previamente:

| Condición sobre `road_load` | Clase (`TrafficState`) | Código |
|---|---|---:|
| $0 \le \text{road\_load} < 25$ | `FLUIDO` | `0` |
| $25 \le \text{road\_load} < 50$ | `MODERADO` | `1` |
| $50 \le \text{road\_load} < 75$ | `DENSO` | `2` |
| $75 \le \text{road\_load} \le 100$ | `RETENCION` | `3` |
| `NaN`, $< 0$ o $> 100$ | Sin etiqueta (se excluye de clasificación) | `NA` |

#### B) Regresión: `DelayLabeler` (demora en `min/km`)

1. **Velocidad de flujo libre ($v^{\text{ff}}_s$) solo con `train`:** para cada sensor $s$, se toman sus velocidades válidas de entrenamiento ($\mathcal{V}^{\text{train}}_s = \{v_{\text{med}, i} \mid i \in \text{train},\; s_i = s,\; v_{\text{med}, i} > 0\}$). Si tiene al menos `min_train_speed_samples` observaciones (umbral fijado tras perfilar `train`), se calcula su percentil 90:

$$
v^{\text{ff}}_{s} = Q_{0.90}\!\left(\mathcal{V}^{\text{train}}_{s}\right) \quad [\text{km/h}]
$$

2. **Cálculo de la demora sobre $L = 1\,\text{km}$:** para cualquier fila con $v_{\text{med}} > 0$ y sensor con $v^{\text{ff}}_s$ válido:

$$
T_{\text{obs}} = \frac{60\,L}{v_{\text{med}}}, \qquad T_{\text{ff}, s} = \frac{60\,L}{v^{\text{ff}}_{s}}
$$

$$
\text{delay\_minutes\_per\_km} = \max\!\left(0,\; T_{\text{obs}} - T_{\text{ff}, s}\right) \quad [\text{min/km}]
$$

- **Sin fallback global:** si un sensor no alcanza el mínimo de muestras en `train`, no se le inventa un P90 global ni de otro sensor; queda excluido de regresión y se documenta.
- **Sin imputación de etiquetas:** las filas sin `road_load` o sin `avg_speed_kmh` válida no se rellenan con ceros ni medias; simplemente no entrenan la tarea donde les falta etiqueta.

### 4.3 Variables predictoras (Allowlist del MVP)

| Grupo | Columnas generadas / permitidas | Regla |
|---|---|---|
| Identidad | `sensor_id` | Codificado como categoría ajustada en `train`. Si llega un sensor desconocido en inferencia, se rechaza. |
| Ciclos temporales (`CyclicalFeatures`) | `hour_sin`, `hour_cos`, `weekday_sin`, `weekday_cos` | Calculadas sobre la hora local (`Europe/Madrid`): $\sin(2\pi h/24)$, $\cos(2\pi h/24)$ para $h \in [0,23]$ y $\sin(2\pi w/7)$, $\cos(2\pi w/7)$ para $w \in [0,6]$ (lunes=0). |
| Calendario (`MadridCalendarFeatures`) | `is_weekend`, `is_holiday` | Sábado/domingo y festivos nacionales, autonómicos y locales de Madrid (incluidos móviles). Si una fecha cae fuera de los años cubiertos por el calendario, lanza error en lugar de asumir laborable (`False`). |
| Clima numérico | `temperature_c`, `precipitation_mm`, `rain_mm`, `visibility_m`, `wind_speed_kmh` | Si falta un campo puntual admitido, se imputa con la **mediana aprendida en `train`**. |
| Clima categórico | `weather_code` | Código WMO tratado como categoría (con categoría explícita para ausentes/desconocidos), nunca como número continuo. |
| Coordenadas (opcional) | `utm_x`, `utm_y` / `latitude`, `longitude` | **Desactivadas por defecto.** Solo se incluyen si un experimento en `train/validation` demuestra mejora real antes de tocar `test`. |

**Variables estrictamente prohibidas en `TrafficFeaturePipeline`:** `intensity_vph`, `occupancy_pct`, `road_load`, `avg_speed_kmh`, las propias etiquetas, el valor $v^{\text{ff}}_s$ y cualquier lag o media móvil de tráfico.

### 4.4 Casos de error

- **`hour_utc` sin zona o duplicados `(sensor_id, hour_utc)`:** error inmediato antes de dividir el dataset.
- **`DelayLabeler.transform` o `TrafficFeaturePipeline.transform` llamados antes de `fit`:** error de estado.
- **Intento de pasar columnas prohibidas:** el pipeline filtra por *allowlist* estricta y rechaza esquemas incompatibles.
- **Ausencia total de clima en una consulta:** no se rellena con medianas de `train`; el servicio la rechaza antes con `WeatherUnavailableError`.

## 5. Dependencias

- Python 3.10+ (`datetime`, `zoneinfo`, `math`), `pandas`, `numpy`, `scikit-learn`.
- Para festivos de Madrid: librería `holidays` (contemplada en `CONSTITUTION.md`, fijando versión y complementando festivos locales de Madrid capital si faltan) o tabla local versionada de festivos oficiales.

## 6. Ejemplos numéricos rápidos

1. **Etiquetas de estado (`road_load`):**
   - `24.99` → `0` (`FLUIDO`)
   - `25.00` → `1` (`MODERADO`)
   - `75.00` → `3` (`RETENCION`)
   - `-5.0` o `102.0` → `NA` (fuera de dominio, sin etiqueta).
2. **Etiqueta de demora (`L = 1 km`, sensor con $v^{\text{ff}}_s = 60\,\text{km/h}$ → $T_{\text{ff}} = 1.0\,\text{min}$):**
   - Si $v_{\text{med}} = 30\,\text{km/h}$ → $T_{\text{obs}} = 60/30 = 2.0\,\text{min}$ → demora = $\max(0, 2.0 - 1.0) = \mathbf{1.0\,\text{min/km}}$.
   - Si $v_{\text{med}} = 60\,\text{km/h}$ → $T_{\text{obs}} = 1.0\,\text{min}$ → demora = $\mathbf{0.0\,\text{min/km}}$.
   - Si $v_{\text{med}} = 80\,\text{km/h}$ (va más rápido que el P90) → $T_{\text{obs}} = 0.75\,\text{min}$ → demora = $\max(0, -0.25) = \mathbf{0.0\,\text{min/km}}$.
   - Si $v_{\text{med}} = 0$ o `NaN` → `NA` (sin etiqueta; nunca se divide por cero).

## 7. Tests obligatorios

1. **Fronteras de `TrafficStateLabeler`:** comprueba `0`, `24.999`, `25`, `49.999`, `50`, `74.999`, `75`, `100` y verifica que `< 0`, `> 100` y `NaN` quedan sin etiqueta.
2. **Fórmula y unidades de `DelayLabeler`:** reproduce el ejemplo de la sección 6 (`30`, `60`, `80 km/h` con P90 `60 km/h`), comprueba que `vmed <= 0` da `NA` y que sensores distintos no mezclan su P90.
3. **Antifuga en P90 e imputadores:** demuestra que añadir valores extremos en `validation` o `test` no cambia en absoluto el P90 de `DelayLabeler`, ni las medianas, categorías o escalas de `TrafficFeaturePipeline`.
4. **Partición cronológica global:** verifica que todas las filas de una misma `hour_utc` caen en el mismo split ($[T_0, T_1)$, $[T_1, T_2)$, $[T_2, T_3)$) sin barajar.
5. **Bloqueo de variables prohibidas:** comprueba que `feature_names` jamás contiene `intensity_vph`, `occupancy_pct`, `road_load`, `avg_speed_kmh` ni lags, y que el pipeline funciona en inferencia futura sin esas columnas.
6. **Ciclos y calendario en `Europe/Madrid`:** verifica seno/coseno en horas `0, 6, 12, 18`, cambios de hora DST, festivos móviles/locales de Madrid y fallo explícito si se consulta un año fuera del calendario configurado.

## 8. Criterios de aceptación

- [ ] **CA-003-1:** Las cuatro fechas de corte ($T_0, T_1, T_2, T_3$) están congeladas en UTC dentro de la configuración y de `ModelMetadata`.
- [ ] **CA-003-2:** `TrafficStateLabeler` y `DelayLabeler` generan etiquetas sin imputar valores ausentes y calculando el P90 por sensor exclusivamente sobre `train`.
- [ ] **CA-003-3:** `TrafficFeaturePipeline` se ajusta solo con `train`, expone exactamente las mismas columnas en entrenamiento e inferencia futura y no utiliza ninguna medida de tráfico contemporánea.
- [ ] **CA-003-4:** Todos los tests de la sección 7 pasan con `pytest` y el código cumple `ruff check .`.
