# Metodología

## Definición del problema

Clasificación binaria single-trial: predecir, a partir del EEG de un único ensayo,
si el target corresponde a una condición semánticamente relacionada (match) o no
relacionada (mismatch). Se eligió el nivel single-trial —y no el promedio por
sujeto— porque es el escenario relevante para aplicaciones de EEG en tiempo real y
el más exigente en términos de relación señal-ruido.

## Datos y unidad de análisis

La unidad de análisis es el ensayo (epoch), no la muestra de señal continua. Se
trabaja sobre el subconjunto procesado del dataset, ya epocheado y limpiado por los
autores originales, lo que prioriza calidad de señal validada sobre un
preprocesamiento propio. Se conservan únicamente los ensayos con respuesta correcta
para reducir ruido conductual, y los dos canales frontales disponibles (AF7, AF8),
coherentes con la localización del componente N400 reportada en la literatura.

## Representación: features tabulares

Cada epoch se resume en 72 características de tres familias —temporales,
frecuenciales e inter-canal— calculadas tanto sobre la ventana N400 (250–600 ms)
como sobre la epoch completa. No se asume a priori que el efecto esté confinado a
una sola ventana: se deja que el análisis posterior determine dónde reside la señal
discriminativa. El enfoque tabular con ML clásico se justifica por el tamaño
moderado del dataset y por la interpretabilidad que permite (relevante para la fase
de explicabilidad).

## Protocolo de validación

La partición es **por sujeto**, no por ensayo: epochs del mismo sujeto comparten
estructura y mezclarlas entre train y test inflaría artificialmente el desempeño.
Se reservan 5 sujetos como holdout, evaluados una única vez al final, y los 24
restantes se usan para desarrollo con GroupKFold (k=5). Toda transformación
(p. ej. escalado) se ajusta dentro del fold de entrenamiento, evitando fuga por
normalización.

Como validación complementaria y más estricta se aplica Leave-One-Subject-Out sobre
los 29 sujetos, que además aporta una medida directa de la variabilidad inter-sujeto
que el GroupKFold promedia.

## Comparación y selección de modelos

Toda comparación —baselines, familias de modelos, modelos tuneados— se realiza bajo
el mismo protocolo, con intervalos de confianza al 95% por bootstrap, para que las
diferencias sean atribuibles al componente evaluado y no al esquema de evaluación.
Se establecen baselines de azar e informados antes de cualquier modelo principal, de
modo que la mejora se mida contra un piso explícito y no contra el vacío. La
optimización de hiperparámetros (Optuna, búsqueda bayesiana) se restringe al
conjunto de desarrollo, sin tocar el holdout.

## Interpretabilidad y verificación

El modelo final se analiza con SHAP y con estudios de ablación, contrastados además
con un análisis univariado independiente. La convergencia (o divergencia) entre
estos enfoques sirve como verificación cruzada: una conclusión sostenida por tres
métodos distintos es más confiable que una derivada de uno solo.

## Reproducibilidad

Parámetros centralizados en un archivo de configuración, semilla fija, pipeline
ejecutable por notebooks numerados y dependencias versionadas. Cada etapa serializa
sus salidas, de modo que los resultados son trazables y regenerables.