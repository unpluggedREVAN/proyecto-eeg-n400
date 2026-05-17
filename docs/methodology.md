# Bitácora Metodológica

Este documento registra **cada decisión técnica del proyecto y su justificación**. Es la fuente única de verdad para el reporte de la Etapa 2 — cuando llegue el momento de redactar, copiamos de aquí.

> ⚠ La rúbrica penaliza −5 por cada **decisión clave sin justificación técnica** (hasta −15). Toda decisión va aquí.

---

## 1. Selección del Track

**Decisión:** Track A — ML Clásico Tabular.

**Justificación:**
- N pequeño (29 sujetos `Included`) hace inadecuado DL como aproximación inicial.
- Muse 2 aporta solo 2 canales en los datos procesados → ingeniería de features es más rentable que aprendizaje de representaciones.
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
- Repetir excluyendo outliers conductuales (MU11, MU20, MU27) y los de baja-mismatch (mu07, mu30, mu34).

---

## 3. Canales utilizados

**Decisión:** AF7 y AF8 (solo los dos disponibles).

**Justificación:**
- En la inspección del Notebook 00 detectamos que los archivos `_allepochs.set` solo conservan AF7 y AF8 (no TP9/TP10).
- El paper Hayes & Magne 2024 reporta que el cluster N400 significativo está en **AF8** entre 250–328 ms; los temporales no son necesarios para capturar el efecto.
- Resultado: el dataset entrega exactamente los canales relevantes para la N400 frontal.

---

## 4. Frecuencia de muestreo

**Decisión:** `sfreq = 256 Hz`.

**Justificación:**
- El Notebook 00 leyó directamente del archivo `.set` y reportó 256 Hz, no 250 Hz como decía la documentación.
- Se actualiza el config para reflejar el valor real medido.

---

## 5. Unidad de análisis

**Decisión:** un trial = una epoch EEG alineada al onset del target word.

**Justificación:**
- El efecto N400 está time-locked al target (Kutas & Federmeier, 2011).
- Esta es la unidad estándar en literatura ERP y la que usa el paper original.

---

## 6. Ventana de epoching

**Decisión:** ventana **−100 a +900 ms** (≈ −101.5 a +894.5 ms según el muestreo a 256 Hz).

**Justificación:**
- Pre-stimulus de 100 ms suficiente para corrección de baseline.
- Post-stimulus de 900 ms cubre toda la componente N400 (250–600 ms) más margen para tardíos.
- Los `.set` procesados ya vienen con esta ventana; no se re-epocha.

---

## 7. Filtros aplicados

**Decisión:** NO se aplican filtros adicionales (notch ni bandpass).

**Justificación:**
- Los `.set` procesados del paper YA están filtrados.
- Re-filtrar introduciría distorsión sin beneficio.
- En la `methodology` del paper Hayes & Magne 2025 se describe el pipeline EEGLAB usado.

---

## 8. Baseline correction

**Decisión:** aplicar baseline correction sustrayendo el promedio de **−100 a 0 ms** a cada epoch, por canal.

**Justificación:**
- En el Notebook 00 detectamos que los `.set` vienen **sin** baseline correction (`epochs.baseline = None`).
- Sin ella, las amplitudes inter-trial son inestables y los promedios no son comparables.
- Estándar absoluto en análisis ERP.

---

## 9. Filtrado a respuestas correctas

**Decisión:** usar solo `matchTarget/Correct` y `mismatchTarget/Correct`.

**Justificación:**
- En trials incorrectos, el sujeto procesó mal el par → la "condición" psicológica no corresponde a la condición experimental → ruido de etiqueta.
- Los archivos `.set` ya marcan correcto/incorrecto en el `event_id` (sintaxis `evento/Correct`), por lo que MNE filtra directamente sin necesidad de cruzar con los CSV conductuales.

**Efecto cuantitativo:**
- Total epochs antes del filtro: ~3000 (datos del Notebook 00).
- Total epochs `Correct` usables: ~1503.
- Distribución resultante: 57.5% match, 42.5% mismatch.
- Trade-off aceptado: menos datos a cambio de etiquetas de mejor calidad.

---

## 10. Sujetos con pocos `mismatchTarget/Correct`

**Hallazgo:** los sujetos mu07 (3), mu30 (7), mu34 (7) tienen muy pocos mismatch correctos.

**Decisión:** conservarlos en el dataset principal; reportar sensitivity analysis en Notebook 08.

**Justificación:**
- Excluirlos a priori sería decisión sin evidencia.
- Su impacto se cuantifica en una ablation explícita.

---

## 11. Rechazo de artefactos

**Decisión preliminar:** umbral peak-to-peak de **±150 µV** por epoch.

**Justificación:**
- Muse 2 tiene más artefactos que EEG clínico; ±75 µV (clínico estándar) eliminaría demasiadas epochs.
- En `sub-mu01` (Etapa 1) el P99 peak-to-peak ≈ 336 µV.
- En el Notebook 01 se reporta una comparativa de umbrales (75, 100, 150, 200, 250, 300 µV) y se justifica el operativo.

**Pendiente:** ablation con umbrales alternativos en Notebook 08.

---

## 12. Prevención de data leakage

**Decisión:** partición **estricta por sujeto** en todos los niveles.

**Mecanismos:**
1. Test holdout: 5 sujetos completos nunca usados durante tuning ni feature engineering.
2. CV: `GroupKFold(n_splits=5)` con `subject_id` como grupo.
3. `StandardScaler` ajustado solo dentro del fold de entrenamiento (vía `Pipeline` de sklearn).
4. LOSO como evaluación secundaria.

**Justificación:**
- Cada sujeto tiene decenas de epochs altamente correlacionadas (misma anatomía, mismo estado).
- Si epochs del mismo sujeto están en train y test, el modelo memoriza al sujeto, no aprende la tarea.
- Penalización por leakage en la rúbrica: −15 puntos.

---

## 13. Métricas

**Decisión:** métrica primaria = **macro F1-score**.

**Justificación:**
- Dataset desbalanceado (57.5% / 42.5%) tras filtrado a Correct.
- Macro F1 trata ambas clases por igual (vs. accuracy que se sesga con desbalance).
- Reportamos accuracy, precision/recall por clase, AUC-ROC y matriz de confusión como complementarias.
- Bootstrap CI 95% sobre todas las métricas.

---

## 14. Baselines

**Baseline 1 — Dummy:** `DummyClassifier(strategy="stratified")`.
- Piso de azar dada la distribución observada de clases.

**Baseline 2 — Heurístico ERP:** regresión logística con 2 features (amplitud media de AF7 y AF8 en la ventana N400 250–600 ms).
- Representa el "análisis ERP clásico" cuantificado. Si nuestro modelo principal no supera esto, el feature engineering completo no aporta valor.

---

## 15. Modelos candidatos (Notebook 04)

Comparar 5 familias con hiperparámetros razonables por defecto:
- Logistic Regression (L2)
- SVM (RBF)
- Random Forest
- XGBoost
- LightGBM

Solo el mejor pasa a tuning con Optuna.

---

## 16. Tuning de hiperparámetros (Notebook 05)

**Decisión:** Optuna con 200 trials, optimizando macro F1 en CV interno.

**Justificación:**
- Optuna usa TPE — más eficiente que GridSearch.
- `MedianPruner` corta trials malos temprano.
- Storage SQLite en Drive permite reanudar si Colab se desconecta.

---

## 17. Explicabilidad (Notebook 07)

SHAP (TreeExplainer si gana modelo de árboles; otro según el caso).

**Plan de análisis:**
1. Global: summary plot + ranking de features.
2. Local: 5 casos (TP alta confianza, TN alta confianza, FP, FN, ambiguo).
3. **Validación teórica:** ¿AF8 y la ventana 250–600 ms están entre las top features?

---

## 18. Ablations (Notebook 08)

1. Solo features temporales (sin frecuenciales).
2. Solo features frecuenciales (sin temporales).
3. Solo AF7 vs solo AF8.
4. Sin baseline correction.
5. Solo ventana N400 vs epoch completa.
6. Con vs sin sujetos outliers conductuales (MU11, MU20, MU27).
7. Con vs sin sujetos de baja-mismatch (mu07, mu30, mu34).
8. Diferentes umbrales de rechazo (±100, ±150, ±200 µV).
9. Con vs sin sujetos `Excluded`.

---

## Cambios y revisiones

| Fecha | Cambio | Justificación |
|-------|--------|---------------|
| Inicio Etapa 2 | Versión inicial | — |
| Tras Notebook 00 | sfreq de 250→256, n_channels de 4→2, eventos con `/Correct` | Hallazgos de la inspección de los `.set` |
