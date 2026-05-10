# Clasificación de ensayos EEG (N400) — Track A: ML Clásico

**Curso:** Inteligencia Artificial · TEC · Sede Cartago
**Profesor:** Kenneth Obando Rodríguez
**Autores:**
- Jose Pablo Agüero Mora — 2021126372
- Roilin Navarro Vargas — 2023213657

## ¿De qué trata el proyecto?

Construimos un pipeline de Machine Learning clásico para clasificar ensayos EEG capturados con un dispositivo Muse 2 durante una tarea de juicio de relación semántica. Cada ensayo se clasifica como **match** (palabras relacionadas, p. ej. `doctor → nurse`) o **mismatch** (palabras no relacionadas, p. ej. `doctor → banana`).

El objetivo es evaluar si un dispositivo EEG de bajo costo (4 canales) puede usarse para clasificación single-trial de procesamiento semántico, integrando rigor metodológico, prevención de data leakage, baselines, explicabilidad (SHAP) y un agente inteligente que consuma el modelo (Etapa 3).

**Dataset base:** [Hayes & Magne (2025) — 37-subject EEG recordings using Muse 2](https://www.sciencedirect.com/science/article/pii/S2352340925001222).

## Estado del proyecto

- [x] **Etapa 1** — Análisis del problema, EDA, diseño del pipeline.
- [ ] **Etapa 2** — Modelado, entrenamiento, XAI y evaluación rigurosa.
- [ ] **Etapa 3** — Integración en agente inteligente + artículo científico.

## Estructura del repositorio

```
proyecto-eeg-n400/
├── README.md                         ← este archivo
├── requirements.txt                  ← versiones de dependencias
├── LICENSE
├── .gitignore
├── configs/
│   └── main_config.yaml              ← parámetros globales del pipeline
├── notebooks/                        ← se ejecutan en orden 00 → 08
│   ├── 00_setup_and_verify.ipynb
│   ├── 01_preprocessing_and_epoching.ipynb     (pendiente)
│   ├── 02_feature_extraction.ipynb             (pendiente)
│   ├── 03_baselines.ipynb                      (pendiente)
│   ├── 04_main_models.ipynb                    (pendiente)
│   ├── 05_hyperparameter_tuning.ipynb          (pendiente)
│   ├── 06_evaluation_and_errors.ipynb          (pendiente)
│   ├── 07_xai_shap.ipynb                       (pendiente)
│   └── 08_ablations.ipynb                      (pendiente)
├── src/
│   └── utils/                        ← funciones helper compartidas
├── reports/                          ← salidas JSON/CSV de cada notebook
├── figures/                          ← figuras generadas
├── features/                         ← matriz de features tabular final
├── models/                           ← modelos entrenados (.pkl)
├── logs/                             ← logs de Optuna y experimentos
└── docs/
    └── methodology.md                ← bitácora metodológica
```

## ¿Dónde están los datos?

Los datos **NO se versionan en este repo** (son varios GB y el dataset es público). Se descargan por separado y se montan desde Google Drive en cada notebook.

Estructura esperada del dataset:

```
dataset/
├── Behavioral Data/
│   ├── MU01_SRJT_Resp.csv ... MU37_SRJT_Resp.csv
│   └── Muse_Demographics.xlsx
├── EEG Data/
│   ├── Processed EEG/
│   │   ├── Included/   ← 29 sujetos
│   │   └── Excluded/   ← 8 sujetos
│   └── Raw EEG/
│       ├── BIDS/       ← estructura BIDS estándar
│       ├── set/        ← .set crudos
│       └── xdf/        ← .xdf crudos
├── Experimental Task/
└── Scripts/
```

## Cómo ejecutar el pipeline (Google Colab)

1. **Clonar este repo en tu Google Drive:**
   ```bash
   # En tu máquina local
   git clone https://github.com/<TU_USUARIO>/proyecto-eeg-n400.git
   # Luego sube la carpeta a Google Drive en MyDrive/IA_Proyecto_EEG/
   ```

2. **Descargar el dataset y colocarlo** en `MyDrive/IA_Proyecto_EEG/dataset/`.

3. **Estructura final esperada en Drive:**
   ```
   MyDrive/IA_Proyecto_EEG/
   ├── dataset/                ← datos del paper Hayes & Magne 2025
   ├── proyecto-eeg-n400/      ← este repo clonado
   ├── reports/, figures/, features/, models/, logs/   ← se crean automáticamente
   ```

4. **Abrir el primer notebook** (`notebooks/00_setup_and_verify.ipynb`) en Colab.

5. **Ejecutar en orden** del 00 al 08. Cada notebook lee la salida del anterior desde Drive.

## Reproducibilidad

- Semilla global en todos los notebooks: **42**.
- Todas las versiones de dependencias fijadas en `requirements.txt`.
- Parámetros centralizados en `configs/main_config.yaml`.
- Cada notebook detecta si su salida ya existe y puede reanudarse sin reprocesar.
- Optuna usa SQLite storage en Drive para tuning resumible.

## Resumen del enfoque metodológico

- **Unidad de análisis:** una época EEG (−100 a 900 ms relativo al onset del target) por trial.
- **Clases:** `matchTarget` vs `mismatchTarget` (binario, balanceado).
- **Prevención de data leakage:** partición estricta por sujeto (ningún sujeto en dos splits).
- **Validación:** `GroupKFold(k=5)` con `subject_id` como grupo + test holdout de 5 sujetos no vistos. LOSO como evaluación complementaria.
- **Baselines obligatorios:**
  1. `DummyClassifier` (stratified) — piso de azar.
  2. Heurístico ERP — regresión logística con solo amplitudes medias en la ventana 250–600 ms.
- **Modelos candidatos:** Logistic Regression, SVM (RBF), Random Forest, XGBoost, LightGBM.
- **Tuning:** Optuna con 100–200 trials, optimizando `macro F1`.
- **XAI:** SHAP global + local; validación de que las features importantes correspondan a la ventana N400 y al canal AF8 (cluster reportado en Hayes & Magne 2024).

## Métricas

Reportadas con bootstrap CI 95%:
- Accuracy
- Macro F1-score (primaria)
- Precision / Recall por clase
- AUC-ROC
- Matriz de confusión

## Licencia

MIT — ver `LICENSE`.

## Referencias

1. Hayes, H. B., & Magne, C. (2025). *Dataset of 37-subject EEG recordings using a low-cost mobile EEG headset during a semantic relatedness judgment task.* Data in Brief, 59, 111390.
2. Hayes, H. B., & Magne, C. (2024). *Exploring the Utility of the Muse Headset for Capturing the N400: Dependability and Single-Trial Analysis.* Sensors, 24(24), 7961.
3. Trammel, T. et al. (2023). *Decoding semantic relatedness and prediction from EEG: A classification method comparison.* NeuroImage.
4. Sabio, J. et al. (2024). *A scoping review on the use of consumer-grade EEG devices for research.* PLOS ONE.

(Ver bibliografía completa en `docs/methodology.md` y en el reporte de la Etapa 1.)
