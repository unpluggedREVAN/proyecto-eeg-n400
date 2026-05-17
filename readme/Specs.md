# Resumen Científico del Pipeline Computacional

A continuación se describe el flujo metodológico del proyecto, organizado en nueve notebooks secuenciales. Cada notebook constituye una etapa funcional discreta del pipeline, con entradas, procesamiento y salidas claramente especificadas. La cadena completa implementa la metodología descrita en `docs/methodology.md` y produce los entregables requeridos por la Etapa 2 del proyecto.

---

## Notebook 00 — Inspección estructural y caracterización del dataset

**Objetivo:** establecer el inventario del dataset, validar su consistencia respecto a lo documentado en la Etapa 1, y caracterizar el formato real de los archivos EEG procesados (`.set` en formato EEGLAB).

**Procedimiento:**

1. Provisión del entorno computacional (Python 3.12 sobre Colab) e instalación de dependencias con versiones fijadas: `MNE-Python 1.7.1`, `scikit-learn 1.4.2`, `XGBoost 2.0.3`, `LightGBM 4.3.0`, `Optuna 3.6.1`, `SHAP 0.45.0`.
2. Montaje del sistema de archivos persistente (Google Drive) y carga de la configuración centralizada (`main_config.yaml`).
3. Verificación de integridad estructural: conteo de sujetos en BIDS (n=37), en `Included` (n=29), en `Excluded` (n=8), y de archivos conductuales (n=37).
4. Caracterización del primer archivo `.set` representativo para determinar si los datos están preprocesados como `mne.Epochs` (escenario A) o como `mne.io.Raw` (escenario B).
5. Extracción de metadata de adquisición: frecuencia de muestreo, canales, ventana temporal de epoching, estado de corrección de baseline, taxonomía de eventos.
6. Cuantificación de epochs por sujeto y condición sobre el subconjunto `Included`.

**Salidas:**

- `reports/00_dataset_inventory.json` — inventario estructural completo.
- `reports/00_epochs_per_subject.csv` — distribución de epochs por sujeto y condición.
- `figures/00_erp_sanity_<subject>.png` — visualización ERP de un sujeto representativo (sanity check).

**Hallazgos relevantes:** Escenario A confirmado; sfreq=256 Hz; canales disponibles AF7 y AF8 (no los cuatro originales); eventos jerárquicamente codificados con sufijos `/Correct` e `/Incorrect`; baseline correction no aplicada en origen.

---

## Notebook 01 — Preprocesamiento y construcción del conjunto de epochs

**Objetivo:** transformar los archivos `.set` brutos en un conjunto de epochs limpias y etiquetadas, listas para extracción de características, garantizando trazabilidad y aplicación uniforme del preprocesamiento.

**Procedimiento:**

1. Lectura de archivos `.set` mediante `mne.io.read_epochs_eeglab` para cada uno de los 29 sujetos en `Included`.
2. Selección y reordenación de canales conforme a la configuración (AF7, AF8).
3. Aplicación de **corrección de línea base** mediante sustracción del promedio en la ventana pre-estímulo [−100, 0] ms por canal y trial. Este paso es estándar en análisis ERP y reduce la varianza inter-trial atribuible a deriva basal.
4. **Filtrado por condición experimental válida**: selección exclusiva de epochs con etiquetas `matchTarget/Correct` y `mismatchTarget/Correct`, excluyendo trials con respuesta incorrecta donde la condición psicológica del sujeto no corresponde a la condición experimental asignada.
5. **Rechazo de epochs por amplitud peak-to-peak** con umbral operativo de ±150 µV. Se incluye un análisis comparativo de umbrales (75, 100, 150, 200, 250, 300 µV) para justificación cuantitativa.
6. Serialización de epochs limpias por sujeto en formato `.fif` (FIFF — Functional Imaging File Format).
7. Construcción del **CSV maestro** con metadata por epoch: `subject_id`, `epoch_idx`, `condition`, `label` (codificación binaria 0=match, 1=mismatch).
8. Cómputo del **grand-average ERP** (promedio de promedios individuales por sujeto) para validación de la presencia del componente N400 a nivel agregado.
9. Cuantificación de la diferencia de amplitud media (mismatch − match) en la ventana 250–600 ms por canal.

**Salidas:**

- `features/epochs_clean/<subject>-epo.fif` — epochs preprocesadas por sujeto.
- `features/01_epochs_master.csv` — registro maestro de epochs etiquetadas.
- `reports/01_preprocessing_summary.csv` — estadísticas de retención por sujeto.
- `reports/01_n400_amplitude_check.csv` — diferencias de amplitud en ventana N400.
- `reports/01_artifact_threshold_comparison.csv` — comparativa de umbrales de rechazo.
- `figures/01_grand_average_ERP.png` — figura agregada por condición.

---

## Notebook 02 — Extracción de características

**Objetivo:** transformar cada epoch (representada como una matriz de dimensiones canales × muestras temporales) en un vector de características numéricas de baja dimensionalidad, apto para algoritmos de ML clásico.

**Procedimiento:**

1. Carga del conjunto de epochs limpias serializadas en el Notebook 01.
2. Extracción de **descriptores en el dominio temporal** por canal, sobre dos ventanas: epoch completa y ventana N400 [250–600] ms. Incluye: media, desviación estándar, varianza, raíz cuadrática media (RMS), amplitud peak-to-peak, asimetría (skewness), curtosis, área bajo la curva, pendiente de regresión lineal.
3. Extracción de **descriptores en el dominio frecuencial** por canal mediante estimación de densidad espectral de potencia (PSD) por método de Welch. Cuantificación de potencia integrada en bandas canónicas: delta (1–4 Hz), theta (4–8 Hz), alpha (8–13 Hz), beta (13–30 Hz). Cómputo de ratios espectrales theta/alpha y beta/alpha.
4. Extracción de **descriptores inter-canal**: asimetría frontal (AF7 − AF8) en amplitud media y en potencias por banda.
5. Cálculo de la **amplitud media específica en la ventana N400** por canal — característica teóricamente prioritaria por su correspondencia directa con el componente ERP investigado.
6. Ensamblaje de la matriz de características X ∈ ℝ^(n_epochs × n_features), conservando los identificadores `subject_id` y `label` como columnas auxiliares para la validación posterior.

**Salidas:**

- `features/02_features_matrix.csv` — matriz tabular completa de características.
- `reports/02_feature_dictionary.json` — diccionario que asocia cada feature con su definición operativa y origen teórico.

---

## Notebook 03 — Modelos de referencia (baselines)

**Objetivo:** establecer modelos de referencia contra los cuales se evalúa el desempeño del modelo principal. La rúbrica del curso exige un mínimo de dos baselines y penaliza con −10 puntos su ausencia.

**Procedimiento:**

1. **Baseline 1 — Trivial estocástico:** `DummyClassifier` con estrategia `stratified` que predice clases respetando la distribución observada en entrenamiento. Establece el límite inferior de desempeño (azar estructurado).
2. **Baseline 2 — Heurístico ERP:** regresión logística con regularización L2 sobre un subconjunto reducido de características (amplitud media en ventana N400 por canal, n=2 features). Representa el "análisis ERP tradicional" cuantificado y operacionaliza la hipótesis nula constructiva: si el modelo principal no supera este baseline, la ingeniería de características adicional no aporta valor discriminativo.
3. Evaluación de ambos baselines bajo el **mismo protocolo** que se aplicará al modelo principal: validación cruzada `GroupKFold(k=5)` con `subject_id` como variable de agrupamiento.
4. Reporte de métricas con intervalos de confianza por bootstrap.

**Salidas:**

- `reports/03_baselines_results.csv` — métricas comparativas con intervalos de confianza al 95%.
- `figures/03_baselines_confusion_matrices.png` — matrices de confusión normalizadas.

---

## Notebook 04 — Comparación de familias de modelos

**Objetivo:** evaluar comparativamente cinco familias de algoritmos supervisados con hiperparámetros por defecto razonables, antes de invertir recursos computacionales en optimización fina. La selección de la familia ganadora se basa en el F1 macro obtenido en validación cruzada.

**Modelos comparados:**

- **Regresión logística** con regularización L2 — modelo lineal de referencia, interpretable.
- **SVM con kernel RBF** — modelo no lineal basado en kernel, históricamente robusto en EEG.
- **Random Forest** — ensemble de árboles con bagging; robusto a outliers, interpretable vía importancia de features.
- **XGBoost** — gradient boosting; estado del arte en datos tabulares.
- **LightGBM** — gradient boosting optimizado; complementario a XGBoost.

**Procedimiento:**

1. Definición de pipelines de sklearn con `StandardScaler` (ajustado dentro de cada fold para prevenir leakage) seguido del estimador.
2. Validación cruzada con `GroupKFold(k=5)` sobre los sujetos no asignados al holdout.
3. Registro de F1 macro, accuracy, precision/recall por clase y AUC-ROC para cada fold y cada modelo.
4. Selección del modelo de mejor desempeño promedio para tuning en el Notebook 05.

**Salidas:**

- `reports/04_models_cv_comparison.csv` — tabla comparativa con métricas por fold y promedios.
- `figures/04_models_cv_boxplot.png` — distribución de métricas por modelo (boxplots).
- `reports/04_best_model_selected.json` — identificador del modelo seleccionado para tuning.

---

## Notebook 05 — Optimización de hiperparámetros

**Objetivo:** identificar la configuración de hiperparámetros que maximiza el F1 macro del modelo seleccionado, mediante búsqueda bayesiana eficiente.

**Procedimiento:**

1. Definición del espacio de búsqueda de hiperparámetros específico al modelo seleccionado en el Notebook 04, basado en rangos recomendados por la documentación del algoritmo y por la literatura aplicada.
2. Configuración del estudio Optuna:
    - Sampler: TPE (Tree-structured Parzen Estimator), más eficiente que búsqueda aleatoria o exhaustiva.
    - Pruner: MedianPruner, que termina anticipadamente los trials con desempeño inferior a la mediana.
    - Storage: SQLite persistente en Drive (`logs/optuna_study.db`), permitiendo reanudación tras desconexiones.
3. Ejecución de 200 trials, cada uno evaluado mediante `GroupKFold(k=5)` con `subject_id` como grupo. La métrica objetivo es F1 macro promedio entre folds.
4. Análisis del estudio: convergencia, importancia relativa de hiperparámetros, distribución de trials.
5. Reentrenamiento del modelo final con los hiperparámetros óptimos sobre la totalidad del conjunto train + validation.

**Salidas:**

- `models/best_model.pkl` — modelo final serializado.
- `reports/05_optuna_history.csv` — registro de todos los trials con sus parámetros y métricas.
- `figures/05_optuna_optimization_history.png` — curva de optimización.
- `figures/05_optuna_param_importances.png` — importancia relativa de hiperparámetros.
- `reports/05_best_hyperparameters.json` — configuración óptima.

---

## Notebook 06 — Evaluación final y diagnóstico

**Objetivo:** evaluar el modelo final sobre el conjunto de test holdout (5 sujetos completos no utilizados en ninguna etapa previa) y producir el diagnóstico completo de desempeño requerido por la rúbrica.

**Procedimiento:**

1. Carga del modelo final entrenado y predicción sobre el test holdout.
2. Cómputo de métricas con **intervalos de confianza al 95% por bootstrap** (1000 iteraciones): accuracy, F1 macro, precision/recall por clase, AUC-ROC.
3. Construcción de la **matriz de confusión** normalizada.
4. Generación de **curvas de aprendizaje** (desempeño vs tamaño de entrenamiento) para diagnosticar régimen de sesgo/varianza (overfitting/underfitting).
5. **Análisis de errores** estructurado:
    - Tasa de FP/FN por sujeto.
    - Correlación de errores con tiempo de reacción del sujeto.
    - Distribución de errores por condición y características del trial.
6. **Evaluación complementaria por LOSO** (Leave-One-Subject-Out): 29 evaluaciones independientes, una por sujeto. Reporte de F1 macro promedio ± desviación estándar entre sujetos. Permite cuantificar la variabilidad inter-sujeto.
7. Comparación numérica final: modelo principal vs baseline 1 vs baseline 2, con tests estadísticos de significancia (Wilcoxon signed-rank entre folds).

**Salidas:**

- `reports/06_final_evaluation.json` — métricas finales con CI.
- `figures/06_confusion_matrix.png`
- `figures/06_learning_curves.png`
- `figures/06_loso_results.png` — F1 macro por sujeto bajo LOSO.
- `reports/06_error_analysis.csv` — análisis estructurado de errores.

---

## Notebook 07 — Explicabilidad mediante SHAP

**Objetivo:** abrir la caja negra del modelo final mediante valores de Shapley (SHAP — SHapley Additive exPlanations), produciendo explicaciones a nivel global y local. Permite validar que el modelo aprovecha señal teóricamente fundamentada (la N400 frontal en AF8) y no artefactos espurios.

**Procedimiento:**

1. Instanciación del explainer apropiado según el modelo final:
    - `TreeExplainer` para Random Forest, XGBoost, LightGBM.
    - `LinearExplainer` para Logistic Regression.
    - `KernelExplainer` para SVM.
2. **Análisis global:**
    - Summary plot de los 20 features más importantes.
    - Ranking de feature importance promedio sobre el test set.
    - Dependence plots de las 5 features principales (relación feature ↔ predicción).
3. **Análisis local** sobre 5 casos seleccionados representativos: verdadero positivo de alta confianza, verdadero negativo de alta confianza, falso positivo, falso negativo, caso ambiguo (probabilidad ≈ 0.5).
4. **Validación teórica explícita:** evaluación cuantitativa de si las features asociadas al canal AF8 en la ventana N400 (250–600 ms) están entre las top contribuyentes. Esta es la prueba de que el modelo aprende señal cognitiva genuina.

**Salidas:**

- `figures/07_shap_summary_global.png`
- `figures/07_shap_dependence_<feature>.png`
- `figures/07_shap_local_<case>.png`
- `reports/07_xai_findings.md` — interpretación escrita de los hallazgos, lista para incorporación al reporte final.

---

## Notebook 08 — Estudios de ablación

**Objetivo:** cuantificar la contribución de cada componente del pipeline mediante ablaciones controladas. Las ablaciones son experimentos donde se modifica un único elemento del sistema para aislar su efecto sobre el desempeño final.

**Ablaciones implementadas:**

1. **Familia de features:** modelo con solo features temporales vs solo frecuenciales.
2. **Selección de canales:** modelo con solo AF7 vs solo AF8.
3. **Corrección de baseline:** con vs sin baseline correction.
4. **Ventana temporal:** features extraídas solo de [250, 600] ms vs epoch completa.
5. **Población de sujetos:**
    - Con vs sin outliers conductuales (MU11, MU20, MU27).
    - Con vs sin sujetos de baja-mismatch (mu07, mu30, mu34).
    - Con vs sin sujetos del subconjunto `Excluded`.
6. **Umbral de rechazo de artefactos:** ±100, ±150, ±200, ±250 µV.

Cada configuración se evalúa bajo el mismo protocolo (GroupKFold + test holdout) y se reporta su métrica primaria. Las diferencias se interpretan como evidencia de la importancia relativa del componente ablacionado.

**Salidas:**

- `reports/08_ablations.csv` — tabla maestra de ablaciones con F1 macro ± CI 95%.
- `figures/08_ablations_comparison.png` — visualización comparativa.

---

## Síntesis del flujo y trazabilidad

El pipeline es **estrictamente secuencial**: cada notebook consume artefactos del anterior y produce artefactos para el siguiente. La trazabilidad experimental se garantiza mediante:

- Semilla aleatoria global fijada (`seed=42`) propagada a `random`, `numpy`, y a todos los algoritmos estocásticos.
- Configuración centralizada en `configs/main_config.yaml`.
- Versiones de dependencias fijadas en `requirements.txt`.
- Logging persistente de cada experimento con tracking de configuraciones, métricas y artefactos.
- Bitácora metodológica viva (`docs/methodology.md`) que registra cada decisión con su justificación técnica.

El conjunto de artefactos producidos satisface los criterios de evaluación de la Etapa 2 (45% de la nota total) y constituye la base experimental sobre la cual se construirá el agente inteligente y el artículo científico final de la Etapa 3.