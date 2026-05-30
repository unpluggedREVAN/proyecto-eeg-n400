# proyecto-eeg-n400 (Etapa 2)

Este repositorio implementa y evalúa un pipeline de Machine Learning para clasificar ensayos de EEG (single-trial) en una tarea de juicio de relación semántica: match vs mismatch. La meta en Etapa 2 es comparar baselines y modelos, ajustar hiperparámetros, evaluar generalización por sujeto (holdout/LOSO) y documentar qué señales/variables está usando el modelo SHAP más ablaciones.

## Qué se está haciendo 
Estamos probando si, con EEG de bajo costo y pocos canales, se puede separar match vs mismatch a nivel de ensayo individual usando features temporales y frecuenciales, con evaluación estricta por sujeto.

## Qué se está probando
- baselines (dummy y heurísticos tipo ERP)
- comparación de modelos sin tuning (LogReg, SVM, RF, XGBoost, LightGBM)
- tuning con Optuna (LogReg y LightGBM)
- evaluación final en sujetos no vistos (holdout por sujeto)
- evaluación por sujeto con Leave-One-Subject-Out (LOSO)
- explicabilidad con SHAP (importancia global y casos locales)
- ablations (familia de features, canal, ventana temporal, y exclusión de grupos de sujetos)

## Dataset
Se usa el dataset “Muse Validation for Language ERPs”, publicado en OSF y descrito en un Data in Brief (2025). El dataset contiene EEG y datos conductuales de una tarea de semantic relatedness, con archivos en formatos como .xdf/.set y una versión BIDS, además de datos procesados (Included/Excluded).

- OSF: https://osf.io/u6y9g/
- DOI dataset: 10.17605/OSF.IO/U6Y9G
- Paper: https://doi.org/10.1016/j.dib.2025.111390

Algo a destacar es que en el pipeline de Etapa 2 se trabaja con el subconjunto procesado Included, que son los sujetos que pasan criterios de calidad, y con los eventos de ensayos correctos.

## Requisitos y ambiente
Las dependencias están listadas en `requirements.txt`.
