# Hallazgos de Ablations — Notebook 08

Cada componente del pipeline se evaluó reentrenando el modelo final (LightGBM
tuneado) bajo el mismo protocolo (GroupKFold por sujeto + bootstrap), variando un
solo componente a la vez respecto a la línea base.

**Línea base:** 72 features, 24 sujetos → Macro F1 = 0.558 [0.531, 0.587]

## Resultados

| Ablation | Macro F1 | Δ vs base |
|----------|---------|-----------|
| Línea base | 0.558 | — |
| Solo epoch completa | 0.558 | +0.000 |
| Solo AF7 | 0.547 | −0.011 |
| Solo temporales | 0.530 | −0.028 |
| Sin baja-mismatch | 0.517 | −0.041 |
| Sin ambos grupos | 0.517 | −0.041 |
| Solo frecuenciales | 0.514 | −0.045 |
| Solo AF8 | 0.511 | −0.047 |
| Sin outliers conductuales | 0.504 | −0.054 |
| Solo inter-canal | 0.492 | −0.066 |
| Solo ventana N400 | 0.491 | −0.067 |

## Interpretación

**Familia de features.** Ninguna familia por separado alcanza la línea base
(temporal 0.530, frecuencial 0.514, inter-canal 0.492). El modelo se beneficia de
la combinación de las tres.

**Canal.** AF7 solo (0.547) supera a AF8 solo (0.511), consistente con SHAP. Ambos
quedan por debajo de la base: aportan información complementaria y reducir a un solo
canal degradaría el desempeño.

**Ventana temporal.** Es el resultado más informativo: usar solo la ventana N400
rinde 0.491 (la peor ablation, Δ = −0.067), mientras que usar solo la epoch completa
iguala la línea base (0.558). Confirma cuantitativamente el hallazgo de SHAP: la
señal discriminativa no se concentra en la ventana N400 canónica.

**Población de sujetos.** Excluir 2 de 24 sujetos reduce el Macro F1 en 0.054, una
caída mayor que eliminar familias enteras de features. Esto no indica robustez sino
sensibilidad a la composición de la muestra: con un número moderado de sujetos, la
variabilidad inter-individual domina, coherente con la desviación del LOSO (±0.091).
Se conservan todos los sujetos porque excluirlos no mejora el desempeño y reduciría
los datos disponibles.

## Conclusión

Ninguna decisión de diseño del pipeline fue accesoria: cada componente contribuye,
la información es distribuida (entre familias, canales y a lo largo de la epoch), y
el desempeño está limitado principalmente por la variabilidad inter-sujeto y el
tamaño muestral, no por la elección de features.