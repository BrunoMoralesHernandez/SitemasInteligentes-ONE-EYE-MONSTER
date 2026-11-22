# Spec 005 — Servicio, TUI e integración

**Estado:** Borrador para revisión  
**Referencias:** [PLAN.md](../PLAN.md), [CONSTITUTION.md](../CONSTITUTION.md), [Spec 001](001-arquitectura-y-contratos.md), [Spec 002](002-datos-madrid-y-meteorologia.md), [Spec 003](003-etiquetas-y-features.md), [Spec 004](004-modelado-e-inferencia.md)

## Guía rápida (en 1 minuto)

Este documento describe **la experiencia completa de consulta** cuando el usuario abre la aplicación en su terminal:

1. **Qué hace `main.py`:** al arrancar, carga el modelo entrenado y el catálogo de sensores **una sola vez**, construye `TrafficPredictionService` (o `MockPredictionService` en modo demo/test) y abre la TUI pasándole el servicio y la lista de sensores disponibles `(sensor_id, sensor_name)`.
2. **Qué hace `TrafficPredictionService`:** cuando el usuario pide un sensor y un rango de horas futuras `[start_at, end_at)`, valida las fechas, pide el pronóstico a Open-Meteo (pasando siempre por la caché en disco), calcula el calendario, transforma las features con el pipeline del modelo, ejecuta `TrafficPredictor` y empaqueta el `ForecastResult`.
3. **Qué pasa si falta el clima:**
   - Si falta el pronóstico completo o alguna hora entera del rango → lanza `WeatherUnavailableError` (nunca inventa horas ni usa un pronóstico caducado en silencio).
   - Si la hora existe pero falta un dato puntual que el pipeline sabe imputar (ej. `visibility_m`) → predice, deja ese campo como `None` en el resumen meteorológico y añade un código de aviso en `warnings`.
4. **Prioridad *core-first* y rendimiento:** la navegación básica y el mock pueden adelantarse, pero el acabado visual completo de la TUI se hace **después de validar los modelos (Hito H4)**. Una consulta de 24 horas con clima ya en caché debe responder en **menos de 1 segundo (p95)** y usando **menos de 4 GB de RAM**.

---

## 1. Objetivo y alcance

Especificar la orquestación de `TrafficPredictionService`, el comportamiento determinista de `MockPredictionService`, la interfaz de consola `OneEyeMonsterApp`, la composición en `main.py`, las pruebas de integración/E2E y la auditoría de rendimiento e instalación limpia (fases 5, 6 y 7 del plan).

Componentes incluidos:
- `TrafficPredictionService` (`models/service.py`)
- `MockPredictionService` (`tui/mocks.py`)
- `OneEyeMonsterApp` y widgets (`tui/app.py`)
- Punto de entrada y composición (`main.py`)

## 2. Fuera de alcance

- Reentrenar modelos o procesar históricos crudos durante el arranque o durante una consulta.
- Añadir paneles visuales complejos antes de superar el hito H4.
- Servidores web, llamadas reales a Internet dentro de los tests automatizados (`pytest`) o rutas absolutas cableadas a una máquina concreta.

## 3. Entradas y salidas

- **Al arrancar `OneEyeMonsterApp`:** recibe una instancia de `PredictionService` y una tupla inmutable de opciones `tuple[tuple[int, str], ...]` con los `(sensor_id, sensor_name)` que están simultáneamente en el catálogo M30 y en `supported_sensor_ids` del modelo.
- **Al consultar:** la TUI construye un `ForecastRequest(sensor_id, start_at, end_at)` y llama a `service.predict(request)`.
- **Resultado:** un `ForecastResult` (definido en el [Spec 001](001-arquitectura-y-contratos.md)) o una de las 5 excepciones de `PredictionError`.

## 4. Flujo del servicio, avisos y TUI

### 4.1 Paso a paso de `TrafficPredictionService.predict`

1. **Validar petición:** comprueba las reglas temporales del Spec 001 (`[start_at, end_at)` aware, horas en punto, futuras respecto al reloj inyectado y sin horas locales DST inexistentes o ambiguas). Si falla → `InvalidForecastRequestError`.
2. **Validar sensor:** comprueba que `sensor_id` está soportado por el artefacto cargado y tiene coordenadas en el catálogo. Si falla → `UnknownSensorError`.
3. **Generar rejilla horaria UTC:** crea la secuencia de horas `[start_utc, ..., end_utc - 1h]`.
4. **Obtener pronóstico meteorológico en lote:** pide todo el rango de una vez a `OpenMeteoClient` (que consulta `FileWeatherCache`). Si falta cobertura para alguna de las horas solicitadas, el pronóstico está fuera de horizonte o la caché ha caducado y no hay red → `WeatherUnavailableError`.
5. **Construir features e inferir:** añade variables cíclicas y calendario de Madrid (`Europe/Madrid`), aplica `TrafficFeaturePipeline.transform` (sin hacer nunca `fit`) e invoca `TrafficPredictor`.
6. **Empaquetar `ForecastResult`:** convierte la salida a `ForecastPoint` por cada hora, añade los códigos de `warnings` si algún campo meteorológico puntual venía nulo y fue imputado, y sella `generated_at` (UTC) y `model_version`.

### 4.2 Catálogo de códigos de aviso (`ForecastPoint.warnings`)

Cuando una hora tiene previsión meteorológica válida pero le falta una variable concreta que el pipeline entrenado puede imputar, el campo se deja en `None` dentro de `WeatherSummary` y se añade su código estable para que la TUI lo muestre al usuario:

| Código en `warnings` | Campo ausente en `WeatherSummary` | Mensaje en la TUI |
|---|---|---|
| `weather_temperature_missing` | `temperature_c` | Temperatura no publicada; se usó el valor imputado del modelo. |
| `weather_precipitation_missing` | `precipitation_mm` | Precipitación no publicada; se usó el valor imputado del modelo. |
| `weather_rain_missing` | `rain_mm` | Lluvia no publicada; se usó el valor imputado del modelo. |
| `weather_visibility_missing` | `visibility_m` | Visibilidad no publicada; se usó el valor imputado del modelo. |
| `weather_wind_speed_missing` | `wind_speed_kmh` | Viento no publicado; se usó el valor imputado del modelo. |
| `weather_code_missing` | `weather_code` | Código meteorológico no publicado; se usó la categoría de ausente. |

### 4.3 `MockPredictionService`

- Vive en `tui/mocks.py` y sirve para desarrollar la navegación básica (tarea 5.1) y ejecutar tests de interfaz sin red ni modelos pesados.
- Aplica **exactamente el mismo validador de `ForecastRequest`** que el servicio real.
- Es 100 % determinista (usa un reloj fijo/inyectado y datos en memoria), devuelve `model_version="mock-v1"` y permite simular bajo demanda las excepciones `UnknownSensorError`, `WeatherUnavailableError` y `ModelUnavailableError`.
- El servicio real **nunca** hace *fallback* silencioso al mock si ocurre un error.

### 4.4 Interfaz `OneEyeMonsterApp` (Textual + Rich)

La TUI muestra una pantalla clara con:
1. **Selector de sensor M30** (por ID o nombre) y **campos de fecha/hora de inicio y fin** en `Europe/Madrid` (indicando que el fin está excluido).
2. **Cabecera de respuesta:** nombre del sensor, `model_version` y hora de generación.
3. **Tabla horaria:**
   - Hora local en `Europe/Madrid` (mostrando el offset `+01:00` / `+02:00` para distinguir el cambio de hora de otoño).
   - Estado previsto (`Fluido`, `Moderado`, `Denso`, `Retención`).
   - Las 4 probabilidades.
   - Demora estimada en `min/km`.
   - Resumen meteorológico y avisos de esa hora.
4. **Panel de error claro:** si el servicio lanza `PredictionError`, muestra la causa sin borrar lo que el usuario había escrito en el formulario para que pueda corregirlo fácilmente.

## 5. Dependencias y empaquetado limpio

- **Sin dependencias nuevas:** usa únicamente el stack aprobado en `AGENTS.md` (`textual`, `rich`, `pandas`, `numpy`, `scikit-learn`, `xgboost`, `pytest`, `ruff` y biblioteca estándar).
- **Instalación limpia:** `python -m src.one_eye_monster.main` resuelve todas las rutas con `pathlib` de forma relativa a la raíz del proyecto o mediante configuración/variables de entorno. Quedan prohibidas las rutas absolutas de la máquina local (`/home/...`) y cualquier secreto en el repositorio.

## 6. Ejemplo de salida en la TUI

Consulta para un sensor entre las `08:00` y las `10:00` locales (`+01:00`):

```text
Sensor: 1001 — M-30 PK 12.4 | Modelo: m30-xgb-v1 | Generado: 2027-01-10 11:00Z

Hora local (Madrid)        Estado    Fluido  Moderado  Denso  Retención  Demora (min/km)  Clima / Avisos
2027-01-11 08:00 (+01:00)  Fluido      70%       20%     8%         2%             0.4   7.5 °C, 0.0 mm, vis. 12000 m
2027-01-11 09:00 (+01:00)  Moderado    25%       55%    15%         5%             2.1   7.8 °C, 0.0 mm, vis. N/D [!]
[!] 09:00 — Visibilidad no publicada; se usó el valor imputado del modelo.
```

## 7. Protocolo de rendimiento (Hito H6)

Para verificar los objetivos de rendimiento sin mezclarlos con la velocidad de Internet ni con el entrenamiento:

- **Escenario:** modelo cargado una sola vez en memoria; consulta de **1 sensor y 24 horas futuras consecutivas** con el pronóstico ya presente y vigente en `FileWeatherCache` (y red bloqueada mediante stub).
- **Latencia:** se ejecutan **30 repeticiones** seguidas de `service.predict(request)` sin cachear el `ForecastResult`. El **percentil 95 (p95)** de las 30 mediciones debe ser **$< 1.0\text{ s}$**.
- **Memoria:** el pico de memoria RSS del proceso durante la carga del artefacto y las 30 consultas debe ser **$< 4\text{ GB}$** (`4000 MB`).
- Se registran en un informe el hardware (CPU, RAM), versiones de librerías, tiempos obtenidos y pico de memoria.

## 8. Tests obligatorios

1. **Orquestación del servicio real (con stubs):** verifica el flujo completo (validación → caché/clima → calendario → pipeline → predictor → `ForecastResult`) y comprueba que un campo nulo imputable genera el código de `warnings` correspondiente dejando `None` en `WeatherSummary`.
2. **Errores de horizonte y caché caducada:** comprueba que si al stub de Open-Meteo le falta una hora del rango, o si la caché está *stale* y la red simulada falla, se lanza `WeatherUnavailableError` y nunca se devuelven puntos parciales ni datos caducados.
3. **Paridad `TrafficPredictionService` vs `MockPredictionService`:** ambos servicios pasan la misma batería de contrato y errores, y el mock devuelve siempre exactamente la misma salida para el mismo input y reloj.
4. **Flujo de la TUI (con utilidades de test de Textual):** selecciona sensor, envía una franja de 2 horas, comprueba que la tabla muestra 2 filas con sus 4 probabilidades, demora en `min/km` y avisos, y verifica que ante un `InvalidForecastRequestError` o `WeatherUnavailableError` se muestra el aviso sin romper la aplicación ni perder los datos del formulario.
5. **Integración E2E offline:** construye la aplicación desde la función de composición de `main.py` usando un artefacto pequeño de prueba y caché local con la red bloqueada, ejecutando una consulta de 24 horas de principio a fin.

## 9. Criterios de aceptación

- [ ] **CA-005-1:** `TrafficPredictionService` y `MockPredictionService` cumplen el contrato del Spec 001 y superan todas las pruebas unitarias, de caché/stubs y de errores sin realizar llamadas reales a Internet.
- [ ] **CA-005-2:** `OneEyeMonsterApp` permite consultar cualquier sensor soportado y franja futura mostrando estado, 4 probabilidades, demora en `min/km`, clima y avisos, sin contener lógica de datos ni de ML en `tui/`.
- [ ] **CA-005-3:** El informe de rendimiento del hito H6 acredita **p95 < 1 s** (en 30 repeticiones de una consulta de 24 h con caché) y **pico RSS < 4 GB**.
- [ ] **CA-005-4:** La aplicación arranca limpiamente con `python -m src.one_eye_monster.main` en un entorno limpio sin rutas absolutas cableadas ni secretos.
