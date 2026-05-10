# Bitácora Metodológica

Este documento registra **cada decisión técnica del proyecto y su justificación**. Es la fuente única de verdad para el reporte de la Etapa 2 — cuando llegue el momento de redactar, copiamos de aquí.

> ⚠ La rúbrica penaliza −5 por cada **decisión clave sin justificación técnica** (hasta −15). Toda decisión va aquí.

---

## 1. Selección del Track

**Decisión:** Track A — ML Clásico Tabular.

**Justificación:**
- N pequeño (29 sujetos `Included`) hace inadecuado el DL como aproximación inicial.
- Muse 2 aporta solo 4 canales → ingeniería de features es más rentable que aprendizaje de representaciones.
- Interpretabilidad: la rúbrica pesa XAI (7%), y SHAP sobre modelos tabulares es estándar y robusto.
- Sin GPU requerida → ejecución estable en Colab gratis.

---

## 2. Población de sujetos

**Decisión inicial:** trabajar con los 29 sujetos del subconjunto `Included`.

**Justificación:**
- El subconjunto `Excluded` (8 sujetos) fue marcado por los autores del dataset por problemas de calidad documentados.
- Usar `Included` mantiene consistencia con el preprocesamiento del paper original (Hayes & Magne, 2024).

**Sensitivity analysis pendiente (Notebook 08):**
- Repetir el pipeline incluyendo `Excluded` para reportar el impacto.

**Outliers conductuales identificados en Etapa 1:** MU11, MU20, MU27 (accuracy < grupo).
**Outliers de señal identificados en Etapa 1:** mu03, mu09, mu13, mu25, mu27, mu32.

**Política operativa:** se conservan en el dataset principal y se hace ablation con/sin ellos en Notebook 08 para cuantificar su impacto.

---

## 3. Unidad de análisis

**Decisión:** un trial = una epoch EEG alineada al onset del target word.

**Justificación:**
- El efecto N400 está time-locked al target, no al prime (Kutas & Federmeier, 2011).
- Esta es la unidad estándar en literatura ERP y la que usa el paper original.
- Las epochs son comparables entre sujetos (mismo evento, misma ventana).

---

## 4. Ventana de epoching

**Decisión:** epoch de **−100 ms a +900 ms** relativo al onset del target.

**Justificación:**
- Pre-stimulus de 100 ms suficiente para corrección de baseline.
- Post-stimulus de 900 ms cubre toda la componente N400 (típicamente 250–600 ms) más margen para tardíos.
- Coincide con el procesamiento del paper origen del dataset.

---

## 5. Baseline correction

**Decisión:** sustracción del promedio de **−100 a 0 ms** a cada epoch, por canal.

**Justificación:**
- Estándar en análisis ERP.
- Reduce la varianza inter-trial atribuible a deriva de baseline.
- Necesario antes de comparar amplitudes entre condiciones.

---

## 6. Filtros aplicados

**Decisión:**
- Notch 60 Hz (red eléctrica USA, donde se grabó el dataset).
- Bandpass 0.1–30 Hz.

**Justificación:**
- ERPs cognitivos como N400 viven en frecuencias bajas (< 30 Hz); arriba de eso predomina ruido (EMG, electrónica).
- 0.1 Hz como corte inferior preserva potenciales lentos sin distorsionar la N400.
- Notch obligatorio dado que el dataset reporta `PowerLineFrequency = 60 Hz`.

**Nota:** si los `.set` ya vienen filtrados desde el preprocesamiento del paper, se documentará en Notebook 01 y se omitirán los filtros para no doble-procesar.

---

## 7. Rechazo de artefactos

**Decisión preliminar:** umbral peak-to-peak de **±150 µV** por epoch.

**Justificación:**
- Muse 2 tiene más artefactos que EEG clínico; ±75 µV (clínico estándar) eliminaría demasiadas epochs.
- En `sub-mu01` se observó peak-to-peak P99 ≈ 336 µV → un umbral en ese orden conserva señal pero rechaza outliers extremos.
- Probaremos también ±100, ±200 y ±250 µV en ablation (Notebook 08).

**Pendiente:** documentar el % de epochs rechazadas por sujeto.

---

## 8. Filtrado por respuesta correcta

**Decisión:** conservar solo trials donde el participante respondió correctamente.

**Justificación:**
- En trials incorrectos, la condición psicológica del sujeto puede no corresponder a la condición experimental (procesó mal el par de palabras).
- Esto introduce ruido de etiqueta y compromete la validez del N400.
- Estándar en literatura ERP semántica.

**Trade-off:** se pierden ~3% de los trials (accuracy promedio reportada en Etapa 1 = 0.9735).

---

## 9. Prevención de data leakage

**Decisión:** partición **estricta por sujeto** en todos los niveles.

**Mecanismos:**
1. Test holdout: 5 sujetos completos nunca usados durante tuning ni feature engineering.
2. CV: `GroupKFold(n_splits=5)` con `subject_id` como grupo.
3. `StandardScaler` ajustado solo dentro del fold de entrenamiento (vía `Pipeline` de sklearn).
4. Script de verificación (`leakage_check.py`) que falla si cualquier sujeto aparece en dos splits.
5. LOSO como evaluación secundaria.

**Justificación:**
- Cada sujeto tiene 56+56 = 112 epochs altamente correlacionadas (misma anatomía, mismo estado).
- Si epochs del mismo sujeto están en train y test, el modelo memoriza al sujeto, no aprende la tarea.
- Penalización por leakage en la rúbrica: −15 puntos.

---

## 10. Métricas

**Decisión:** métrica primaria = **macro F1-score**.

**Justificación:**
- Dataset balanceado en estructura, pero pueden surgir desbalances residuales tras el rechazo de artefactos.
- Macro F1 trata ambas clases por igual (vs. accuracy que se sesga con desbalance).
- Reportamos accuracy, precision/recall por clase, AUC-ROC y matriz de confusión como complementarias.
- Bootstrap CI 95% sobre todas las métricas para reportar incertidumbre.

---

## 11. Baselines

**Decisión:** dos baselines obligatorios.

**Baseline 1 — Dummy:** `DummyClassifier(strategy="stratified")`.
- Justificación: piso de azar dada la distribución observada de clases.

**Baseline 2 — Heurístico ERP:** regresión logística con 4 features (amplitud media de cada canal en la ventana N400).
- Justificación: representa el "análisis ERP clásico" cuantificado. Si nuestro modelo principal no supera esto, el feature engineering completo no aporta valor.

---

## 12. Modelos candidatos (Notebook 04)

**Decisión:** comparar 5 familias con hiperparámetros por defecto razonables:
- Logistic Regression (L2)
- SVM (RBF kernel)
- Random Forest
- XGBoost
- LightGBM

**Justificación:**
- Cubre familias lineales (LogReg), basadas en kernel (SVM), basadas en árboles ensemble (RF, XGB, LGBM).
- Para datasets tabulares de tamaño moderado, alguna de estas suele ser óptima.
- Las 5 corren rápido en CPU sin tuning.

Solo la mejor pasa a tuning con Optuna (Notebook 05).

---

## 13. Tuning de hiperparámetros (Notebook 05)

**Decisión:** Optuna con 200 trials, optimizando macro F1 en CV interno.

**Justificación:**
- Optuna usa TPE (Tree-structured Parzen Estimator) — más eficiente que GridSearch o Random Search ciegos.
- 200 trials es un compromiso razonable entre tiempo (~1-1.5 h en Colab) y exploración.
- `MedianPruner` corta trials malos temprano → reduce costo computacional.
- Storage SQLite en Drive permite reanudar si Colab se desconecta.

---

## 14. Explicabilidad (Notebook 07)

**Decisión:** SHAP (TreeExplainer si gana modelo de árboles; KernelExplainer si gana SVM; LinearExplainer si gana LogReg).

**Plan de análisis:**
1. Global: summary plot + ranking de features.
2. Local: 5 casos (TP alta confianza, TN alta confianza, FP, FN, ambiguo).
3. Validación teórica: ¿AF8 y la ventana 250–600 ms están entre las top features?

**Justificación:**
- SHAP es la técnica de XAI estándar en ML tabular.
- La validación teórica conecta los resultados del modelo con el efecto N400 reportado en literatura → cierra el loop científico.

---

## 15. Ablations (Notebook 08)

Para entender de dónde proviene el desempeño:

1. Solo features temporales (sin frecuenciales).
2. Solo features frecuenciales (sin temporales).
3. Solo canales frontales (AF7, AF8) vs solo temporales (TP9, TP10).
4. Sin baseline correction.
5. Solo ventana N400 vs epoch completa.
6. Con vs sin sujetos outliers (MU11, MU20, MU27).
7. Diferentes umbrales de artifact rejection (±100, ±150, ±200 µV).
8. Con vs sin sujetos del subconjunto `Excluded`.

---

## Cambios y revisiones

| Fecha | Cambio | Justificación |
|-------|--------|---------------|
| 2026-mm-dd | Versión inicial de la bitácora | Inicio Etapa 2 |
