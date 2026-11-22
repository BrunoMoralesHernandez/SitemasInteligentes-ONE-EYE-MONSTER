# AGENTS.md — O.N.E. E.Y.E. M.O.N.S.T.E.R.

## 1. Visión del Proyecto

Aplicación modular en Python para predecir atascos de tráfico y estimar tiempos de demora cruzando registros históricos de tráfico de Madrid con meteorología municipal y Open-Meteo. La interfaz principal es una aplicación de consola (TUI).

## 2. Stack Tecnológico

- **Lenguaje:** Python 3.10+ (ejecutar siempre dentro del entorno virtual activo).
- **Datos y Machine Learning:** `pandas`, `numpy`, `scikit-learn`, `xgboost`.
- **Interfaz de Consola (TUI):** `rich` y `textual`.
- **Fuentes de Datos:** histórico oficial de tráfico y meteorología del Ayuntamiento de Madrid, API REST de Open-Meteo y datasets locales (CSV/Parquet).
- **Testing y Calidad:** `pytest` para tests, `ruff` (o `flake8`/`black`) para formateo y linting.

## 3. Arquitectura del Repositorio

Respeta estrictamente esta estructura. No crees carpetas nuevas en la raíz sin autorización:

```text
one-eye-monster/
├── data/               # Datasets crudos y procesados (ignorado en git salvo muestras)
├── docs/               # Documentación técnica y arquitectura
├── models/             # Modelos entrenados serializados (.joblib / .json)
├── src/
│   └── one_eye_monster/
│       ├── data/       # Ingesta, limpieza y clientes de API (Open-Meteo)
│       ├── features/   # Ingeniería de características (feature engineering)
│       ├── models/     # Entrenamiento, evaluación e inferencia (Sklearn/XGBoost)
│       ├── tui/        # Vistas, widgets y lógica de interfaz (Rich/Textual)
│       └── main.py     # Punto de entrada de la aplicación
└── tests/              # Tests unitarios y de integración (espejo de src/)
```

## 4. Especificaciones de Datos y Modelado (Madrid)

- **Filtrado inicial:** Descartar siempre registros con `error != 'N'`. Priorizar sensores con `tipo_elem == 'M30'` para disponer de `vmed` fiable además de `intensidad`, `ocupacion` y `carga`.
- **Meteorología híbrida:** observaciones municipales de Madrid como fuente histórica principal desde 2019; Open-Meteo (`temperature_2m`, `precipitation`, `rain`, `visibility`, `wind_speed_10m`, `weather_code`) como complemento histórico y como única previsión futura, siempre mediante caché.
- **Etiquetas de Clasificación (desde `carga`):**
  - `0` — **Fluido**: `< 25`
  - `1` — **Moderado**: `25 – 49`
  - `2` — **Denso**: `50 – 74`
  - `3` — **Retención**: `>= 75`
- **Variable de Regresión (`delay_minutes_per_km`):** Calculada comparando el tiempo de recorrido a velocidad actual (`vmed`) frente al tiempo en flujo libre (percentil 90 del sensor, solo entrenamiento) sobre un tramo de referencia de 1 km. La salida se expresa en `min/km`.

## 5. Convenciones de Código

- **Desacoplamiento:** Nunca mezcles lógica de negocio, ingesta de datos o inferencia ML dentro de los archivos de `tui/`. La TUI solo consume resultados.
- **Tipado:** Usa type hints obligatorios en todas las firmas de funciones y métodos.
- **Documentación:** Usa docstrings en formato Google Style en todos los módulos, clases y funciones públicas.
- **Estilo:** Nombres de archivos y funciones en `snake_case`; clases en `PascalCase`; constantes en `UPPER_SNAKE_CASE`. Aplica principios SOLID (especialmente responsabilidad única por módulo).

## 6. Comandos Clave

- **Ejecutar aplicación:** `python -m src.one_eye_monster.main`
- **Ejecutar tests:** `pytest`
- **Verificar estilo/linting:** `ruff check .`

## 7. Flujo de Trabajo Obligatorio

1. **Desarrollo por Fases:** Implementa únicamente la fase o tarea solicitada. No te adelantes a fases futuras.
2. **Verificación Previa:** Antes de dar por terminada una fase, asegúrate de que el código no tiene errores de sintaxis, cumple con el tipado y pasa los tests correspondientes.
3. **Punto de Control (Checkpoint):** Al finalizar cada fase, detente, resume brevemente los archivos creados/modificados y espera la verificación explícita del usuario antes de continuar con la siguiente fase.
