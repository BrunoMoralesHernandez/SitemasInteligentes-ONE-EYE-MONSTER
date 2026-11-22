# Spec 004 — Modelado e inferencia

**Estado:** Borrador para revisión  
**Referencias:** [PLAN.md](../PLAN.md), [CONSTITUTION.md](../CONSTITUTION.md), [Spec 001](001-arquitectura-y-contratos.md), [Spec 003](003-etiquetas-y-features.md)

## Guía rápida (en 1 minuto)

Este documento explica **cómo se entrenan, evalúan y guardan los dos modelos de Machine Learning** y cómo los ejecuta `TrafficPredictor`:

- **Dos modelos independientes:** un clasificador para el estado (`Fluido`, `Moderado`, `Denso`, `Retención`) y un regresor para la demora (`delay_minutes_per_km`). Ninguno usa la salida del otro como entrada.
- **Reglas del juego limpias:**
  1. Primero medimos un modelo tonto de referencia (*baseline*) en `validation`.
  2. En clasificación compensamos que hay pocos atascos graves (`Retención`) aplicando **pesos por clase calculados solo en `train`**.
  3. Elegimos el mejor modelo e hiperparámetros mirando **solo `validation`**.
  4. **`test` se ejecuta una sola vez al final** para comprobar si alcanzamos los objetivos del proyecto (**Macro F1 > 0,75** y **MAE < 5 min/km**). Si no se alcanzan, se reporta con honestidad; no se hace trampa reentrenando contra `test`.
- **Artefacto todo-en-uno:** guardamos en `models/<model_version>/` los dos pipelines, la lista de sensores soportados, las métricas y las versiones de librerías.
- **Inferencia pura (`TrafficPredictor`):** recibe la tabla de features ya lista y devuelve el estado, las 4 probabilidades (que suman 1) y la demora truncada a $\ge 0$. No toca la red, ni la caché, ni la TUI.

---

## 1. Objetivo y alcance

Definir el protocolo de entrenamiento, el tratamiento del desbalanceo, la selección en `validation`, la evaluación única en `test`, el formato del artefacto serializado y el comportamiento de `TrafficPredictor`.

Componentes incluidos:
- `CongestionTrainer`, `DelayTrainer` (`models/training.py`)
- `ModelArtifact`, `ModelMetadata`, `EvaluationReport` (`models/artifact.py`)
- `TrafficPredictor` (`models/inference.py`)

## 2. Fuera de alcance

- Ingesta y cruce meteorológico ([Spec 002](002-datos-madrid-y-meteorologia.md)) o creación de etiquetas, P90 y calendario ([Spec 003](003-etiquetas-y-features.md)).
- Orquestación con Open-Meteo en vivo, caché o pantallas de la TUI ([Spec 005](005-servicio-tui-e-integracion.md)).
- Librerías externas no aprobadas (`imbalanced-learn`, `optuna`, `mlflow`, servidores web).

## 3. Entradas y salidas

### 3.1 En entrenamiento
- **Entrada:** dataset horario con particiones `train/validation/test`, features permitidas y etiquetas del Spec 003. Cada entrenador filtra solo las filas donde su etiqueta es válida.
- **Sensores soportados (`supported_sensor_ids`):** intersección de los sensores que tienen datos válidos tanto para el clasificador como para el regresor (incluyendo P90 válido en `train`).
- **Salida en disco (`models/<model_version>/`):**
  - Pipelines serializados con `joblib` (clasificación y regresión).
  - `metadata.json` (`ModelMetadata`) y `evaluation.json` (`EvaluationReport`).

### 3.2 En inferencia (`TrafficPredictor`)
- **Entrada:** `DataFrame` con las columnas `feature_names` en el orden exacto del artefacto.
- **Salida por fila:**
  - `traffic_state`: `TrafficState` con mayor probabilidad (empates resueltos a favor del código menor).
  - `state_probabilities`: `dict[TrafficState, float]` con las 4 clases (`0..3`), valores en $[0, 1]$ y suma $1.0$.
  - `delay_minutes_per_km`: `float` finito $\ge 0.0$ (si el regresor predice un valor ligeramente negativo, se trunca con $\max(0.0, \hat{y})$).

## 4. Reglas de entrenamiento, evaluación y artefacto

### 4.1 Baselines y modelos candidatos

| Tarea | Baseline (referencia en `validation`) | Candidatos permitidos (`scikit-learn` / `xgboost`) | Criterio de selección en `validation` |
|---|---|---|---|
| **Clasificación** (`CongestionTrainer`) | `DummyClassifier(strategy="prior")` | `LogisticRegression`, `RandomForestClassifier`, `HistGradientBoostingClassifier`, `XGBClassifier(objective="multi:softprob")` | Maximizar **Macro F1** (desempate por F1 de `Retención`). |
| **Regresión** (`DelayTrainer`) | `DummyRegressor(strategy="median")` | `Ridge`, `RandomForestRegressor`, `HistGradientBoostingRegressor`, `XGBRegressor` | Minimizar **MAE** en `min/km` (desempate por RMSE). |

- Toda búsqueda de hiperparámetros usa semillas fijas (`random_state=42` por defecto) y se evalúa en `validation` (o con `TimeSeriesSplit` interno dentro de `train`).
- Los baselines **nunca** se evalúan en `test`; solo sirven para comprobar en `validation` que el modelo elegido aporta valor real.

### 4.2 Balanceo de clases sin fuga

Como la clase `3` (`Retención`) es minoritaria, el clasificador debe entrenarse con **pesos inversamente proporcionales a la frecuencia de cada clase en `train`**:

$$
w_c = \frac{N_{\text{train}}}{K_{\text{train}} \cdot n_c}
$$

- Se aplica mediante `class_weight="balanced"` o `sample_weight` al hacer `fit` en `train` (nunca ambos a la vez).
- `validation` y `test` **no se rebalancean ni se remuestrean**: deben reflejar la distribución real del tráfico.

### 4.3 Métricas y evaluación única de `test`

Una vez congelados los dos modelos ganadores de `validation`, se ejecuta `test` **una sola vez**:

1. **Clasificación (sobre las 4 clases `0, 1, 2, 3`):**
   - Métrica principal: **Macro F1** (objetivo del proyecto: `> 0.75`).
   - Obligatorio reportar: precisión, *recall*, F1 y soporte **por cada una de las 4 clases** y la **matriz de confusión $4 \times 4$**.
2. **Regresión (en `min/km`, aplicando el mismo truncado $\max(0, \hat{y})$ que en inferencia):**
   - Métrica principal: **MAE** (objetivo del proyecto: `< 5.0 min/km`).
   - Obligatorio reportar: RMSE, mediana del error absoluto y número de muestras evaluadas.

> **Regla de honestidad:** `Macro F1 > 0.75` y `MAE < 5 min/km` son objetivos a verificar, no resultados garantizados. Si en `test` no se alcanzan, el informe registra `no_cumple` y se abre revisión en el checkpoint H4; está prohibido reajustar hiperparámetros usando `test`.

### 4.4 Contenido obligatorio de `ModelMetadata`

```yaml
model_version: string
supported_sensor_ids: list[int]
training_time_range: datetime_range
validation_time_range: datetime_range
test_time_range: datetime_range
timezone: Europe/Madrid
traffic_temporal_resolution: 1h
source_temporal_resolution: 15min
reference_length_km: 1.0
classification_thresholds:
  Fluido: "< 25"
  Moderado: "25-49"
  Denso: "50-74"
  Retencion: ">= 75"
feature_names: list[str]
dataset_fingerprint: string
weather_sources:
  - madrid_municipal
  - open_meteo
dependency_versions: map[str, str]
classification_metrics: map[str, float]
regression_metrics: map[str, float]
```

Además, se guarda la tabla de velocidades P90 por sensor y los hashes SHA-256 de los archivos `.joblib`. Por seguridad, **solo se cargan artefactos locales generados por el propio proyecto**.

### 4.5 Casos de error en carga e inferencia

- **Artefacto incompleto, hash inválido o versión de esquema incompatible:** lanza `ModelUnavailableError` al cargar.
- **Columnas de entrada distintas a `feature_names` o sensor fuera de `supported_sensor_ids`:** `TrafficPredictor` rechaza la llamada sin inventar predicciones.
- **Probabilidades con `NaN`/`inf` o suma no positiva:** lanza `PredictionError`. Si durante el entrenamiento con un fixture faltó alguna de las 4 clases en `classes_`, `TrafficPredictor` rellena esa clase con probabilidad `0.0` para devolver siempre las 4 claves de `TrafficState`.

## 5. Dependencias

- Python 3.10+, `pandas`, `numpy`, `scikit-learn`, `xgboost`, `joblib` (incluido con `scikit-learn`), `pytest` y `ruff`.

## 6. Ejemplo de salida de `TrafficPredictor`

Para una fila de entrada válida, el predictor devuelve internamente:

```python
{
    "traffic_state": TrafficState.DENSO,
    "state_probabilities": {
        TrafficState.FLUIDO: 0.05,
        TrafficState.MODERADO: 0.20,
        TrafficState.DENSO: 0.60,
        TrafficState.RETENCION: 0.15,
    },
    "delay_minutes_per_km": 2.8,
}
```

## 7. Tests obligatorios

1. **Baselines deterministas:** comprueba que el baseline de clasificación (frecuencias de `train`) y el de regresión (mediana de `train`) se evalúan solo en `train/validation` y nunca en `test`.
2. **Pesos de balanceo solo en `train`:** verifica que los pesos por clase coinciden con la fórmula de la sección 4.2 usando solo recuentos de `train`, y que `validation`/`test` mantienen sus filas intactas.
3. **Antifuga en entrenamiento:** espías sobre ambos pipelines confirman que ningún `fit` recibe filas de `validation` ni de `test`.
4. **Cálculo de métricas e informe:** con predicciones sintéticas conocidas, verifica el cálculo de Macro F1, matriz $4 \times 4$, soportes por clase (incluido soporte 0) y MAE en `min/km`.
5. **Contrato de `TrafficPredictor`:** verifica que siempre devuelve las 4 claves de `TrafficState` sumando `1.0`, que elige el `argmax` determinista, que trunca demoras negativas a `0.0` y que lanza error ante `NaN` o columnas ausentes.
6. **Serialización *round-trip* y compatibilidad:** guardar y volver a cargar un `ModelArtifact` en un directorio temporal produce exactamente las mismas predicciones y falla con `ModelUnavailableError` si se altera el hash o falta un campo obligatorio de `ModelMetadata`.

## 8. Criterios de aceptación

- [ ] **CA-004-1:** `CongestionTrainer` y `DelayTrainer` entrenan sus pipelines por separado sobre `train`, aplican balanceo explícito en clasificación y seleccionan el modelo usando únicamente `validation`.
- [ ] **CA-004-2:** Existe un `ModelArtifact` versionado en `models/` con todos los metadatos de la sección 4.4 y el `EvaluationReport` completo (baselines, candidatos, matriz $4 \times 4$, Macro F1 y MAE en `min/km`).
- [ ] **CA-004-3:** La evaluación sobre `test` se ha ejecutado una única vez tras cerrar la selección, registrando de forma transparente si se cumplen o no los objetivos `Macro F1 > 0.75` y `MAE < 5 min/km`.
- [ ] **CA-004-4:** `TrafficPredictor` supera todos los tests de la sección 7 sin depender de red, caché ni TUI.
