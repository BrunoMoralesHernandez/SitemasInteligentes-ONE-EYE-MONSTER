# CONSTITUTION.md — O.N.E. E.Y.E. M.O.N.S.T.E.R.

Este documento establece los principios arquitectónicos, científicos y operativos **inviolables** del proyecto. Ninguna solicitud puntual ni prototipo rápido justifica romper estas reglas.

---

## 1. Filosofía del Sistema y Ejecución Local
- **Autosuficiencia y Ligereza:** La aplicación debe ejecutarse 100 % en local desde la terminal (TUI), priorizando el bajo consumo de memoria y respuestas analíticas instantáneas. Queda estrictamente prohibido introducir servidores web pesados (FastAPI, Flask, Django), bases de datos externas o dependencias de navegador.
- **Prioridad Core-First:** La interfaz TUI se construye siempre de forma incremental sobre un núcleo de datos e IA ya funcional y validado. Nunca se desarrollará complejidad visual antes de cerrar los hitos de modelado.

## 2. Arquitectura y Separación de Responsabilidades
- **Desacoplamiento Estricto:** La capa de interfaz (`tui/`) jamás contendrá lógica de negocio, transformaciones de datos, llamadas directas a APIs ni código de inferencia. La TUI únicamente consume resultados de los servicios de dominio.
- **Modularidad:** Cada módulo debe responder a una única responsabilidad (ingesta, ingeniería de características, entrenamiento/inferencia, visualización).

## 3. Integridad de Datos y Prevención de Riesgos
- **Inmutabilidad del Dataset Crudo:** Los datos históricos originales descargados nunca se sobrescriben ni modifican directamente. Toda limpieza debe generar un artefacto procesado y congelado localmente para garantizar reproducibilidad.
- **Alineación Temporal Estricta:** El cruce entre aforos de tráfico y datos meteorológicos debe pasar siempre por una función explícita de redondeo o interpolación temporal por ventana horaria. Nunca se asume sincronización exacta por defecto.
- **Caché Obligatorio en APIs:** Toda consulta a la API de Open-Meteo debe pasar por una capa de caché local. Está prohibido realizar peticiones repetidas en ráfaga por los mismos rangos horarios/geográficos.
- **Calendario Dinámico:** El tratamiento de días no laborables no puede depender exclusivamente de reglas fijas; debe contemplar festivos móviles/locales mediante librerías de calendario (`holidays`) o mapeos configurables.

## 4. Rigor Científico en Machine Learning
- **Cero *Data Leakage*:** El escalado de variables numéricas, la codificación cíclica (seno/coseno para horas y días) y cualquier técnica de balanceo deben ajustarse (`fit`) **exclusivamente** sobre el conjunto de entrenamiento antes de aplicarse a validación o test.
- **Tratamiento Obligatorio de Desbalanceo:** Dado el bajo porcentaje natural de eventos de "Retención", ningún modelo de clasificación multiclase (`Fluido`, `Moderado`, `Denso`, `Retención`) se considerará válido sin aplicar estrategias explícitas de balanceo (reponderación de clases en XGBoost, submuestreo controlado o SMOTE).
- **Doble Salida Evaluada:** El sistema debe mantener la separación clara entre el problema de clasificación (estado de la vía) y el de regresión (minutos adicionales de demora), registrando siempre métricas de evaluación antes de serializar cualquier modelo.

## 5. Seguridad, Gobernanza y Control Humano
- **Cero Secretos ni Rutas Hardcodeadas:** Prohibido escribir rutas absolutas de máquina local, credenciales o tokens en el código fuente. Usa siempre rutas relativas al proyecto (`pathlib`) y variables de entorno si fueran necesarias.
- **Control de Dependencias:** No se instalará ninguna librería externa fuera del stack aprobado sin autorización explícita del usuario.
- **Flujo por Fases Verificadas:** El agente tiene prohibido avanzar a una nueva fase de desarrollo sin haber recibido confirmación explícita del usuario tras revisar la fase anterior.