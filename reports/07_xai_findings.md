# Hallazgos XAI — Notebook 07

Análisis de explicabilidad con SHAP (TreeExplainer) sobre el modelo final
(LightGBM tuneado), calculado sobre el conjunto holdout.

## 1. Importancia global de features

Las 5 features de mayor importancia (|SHAP| promedio) fueron:

| Rank | Feature | |SHAP|̄ |
|------|---------|--------|
| 1 | freq_ratio_theta_alpha_AF8_full | 0.0689 |
| 2 | temp_var_AF7_full | 0.0660 |
| 3 | temp_ptp_AF7_full | 0.0590 |
| 4 | temp_slope_AF7_N400 | 0.0327 |
| 5 | cross_diff_beta_N400 | 0.0310 |

El modelo se apoya en medidas de variabilidad temporal (varianza, peak-to-peak,
pendiente) y en ratios/potencias espectrales, más que en la amplitud media del
N400. La feature teóricamente central, `temp_mean_AF8_N400`, quedó en rank 12 con
una importancia casi cuatro veces menor que la primera.

## 2. Agregación por categoría

- **Familia:** temporal (0.370) > frecuencial (0.173) > inter-canal (0.108)
- **Canal:** AF7 (0.295) > AF8 (0.248) > cross (0.108)
- **Ventana:** epoch completa (0.446) > ventana N400 (0.205)

La epoch completa concentra más del doble de la importancia que la ventana N400.
El modelo no discrimina a partir del componente N400 clásico, sino de patrones de
variabilidad eléctrica frontal distribuidos a lo largo de toda la epoch.

## 3. Convergencia con el análisis univariado

Solo 4 de las 15 features más importantes según SHAP aparecen también entre las 15
más discriminativas del test de Welch (Notebook 02). La coincidencia es parcial:
parte de la señal es estadísticamente consistente, pero el modelo además explota
combinaciones multivariadas que el univariado no detecta de forma aislada.

## 4. Hallazgo central

La información discriminativa a nivel single-trial reside en la variabilidad de
banda ancha de la señal frontal completa, no en la amplitud del N400 en su ventana
canónica. Esto es coherente con el bajo SNR del EEG single-trial con dispositivos
de consumo: un efecto robusto a nivel de promedio (grand-average) puede no ser la
fuente principal de discriminación trial a trial.

## 5. Limitaciones

- SHAP atribuye importancia a features (resúmenes estadísticos), no a procesos
  neurales; la importancia no implica causalidad fisiológica.
- Los valores se calcularon sobre el holdout de 5 sujetos; los rankings agregados
  pueden variar con muestras mayores.