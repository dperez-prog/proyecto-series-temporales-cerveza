# Forecasting de Producción Mensual de Cerveza en EE.UU. (Barriles)

**Materia:** Series Temporales — Maestría en Ciencias de la Inteligencia Artificial
**Universidad:** Universidad Nacional de Asunción (FIUNA)
**Profesores:** Dr. Diego Stalder — Msc. Enrique Paiva

**Integrantes:**
- Daira Pérez (Ingeniera de Alimentos)
- Malena Russo (Ingeniera Química)

**Fecha:** Julio 2026

---

## 1. Descripción del problema

La industria cervecera de EE.UU. reporta mensualmente su producción total ante la
*Alcohol and Tobacco Tax and Trade Bureau (TTB)*. En este proyecto se busca predecir
la **producción mensual de cerveza, medida en barriles**, a nivel nacional. Contar con
una predicción confiable permite anticipar necesidades de insumos (malta, lúpulo,
cereales), planificar logística de distribución y dimensionar la capacidad productiva
del sector.

## 2. Dataset

- **Fuente:** [Beer Production — Kaggle (jessemostipak)](https://www.kaggle.com/datasets/jessemostipak/beer-production)
- **Origen original:** Alcohol and Tobacco Tax and Trade Bureau (TTB), vía TidyTuesday (2020-03-31)
- **Licencia:** CC0 Public Domain
- **Archivo utilizado:** `beer_taxed.csv`
- **Frecuencia:** Mensual, 2008–2019 (144 observaciones)
- **Variable objetivo:** Barriles producidos por mes (`type='Production'`, `tax_status='Totals'`)

### Nota metodológica sobre selección de dataset

Inicialmente se evaluó la serie de consumo de insumos (`brewing_materials.csv`,
malta y derivados), pero se detectó un **quiebre estructural no explicado** a partir
de 2016 (caída simultánea de más del 90% en todas las categorías de insumos por
igual, entre diciembre 2015 y enero 2016), atribuible a un cambio en la metodología
de reporte de la fuente (TTB). Este patrón se confirmó también en `beer_states.csv`
(salto similar en la serie anual de Alaska), reforzando la hipótesis de un cambio
metodológico general de la fuente y no un error puntual del dataset.

Por este motivo se optó por la serie de producción total (`beer_taxed.csv`), que no
presenta esta discontinuidad y ofrece un rango temporal más extenso y consistente
(144 vs. 96 observaciones utilizables).

## 3. Metodología aplicada

1. **Análisis exploratorio (EDA):** visualización de la serie temporal y análisis
   de su distribución, tendencia y estacionalidad.
2. **Test de estacionariedad:** Dickey-Fuller aumentado (ADF) sobre la serie
   original y sobre la serie diferenciada.
3. **Descomposición:** separación en tendencia, estacionalidad y residuo
   (modelo aditivo, período=12).
4. **Identificación de órdenes SARIMA:** análisis de ACF/PACF sobre la serie
   diferenciada (estacional + regular).
5. **División train/test:** últimos 12 meses (2019) como conjunto de prueba,
   respetando el orden temporal.
6. **Ingeniería de atributos (XGBoost):** variables de calendario (mes, trimestre)
   y variables de rezago (lags 1, 2, 3 y 12), construidas exclusivamente sobre el
   conjunto de entrenamiento, sin acceder a datos de 2019.
7. **Pronóstico recursivo (XGBoost):** el modelo predice mes a mes utilizando sus
   propias predicciones anteriores como input, sin acceder a valores reales de 2019.
8. **Evaluación:** métricas RMSE, MAE y MAPE sobre el conjunto de prueba, además
   de un análisis de error por horizonte de pronóstico (h=1 a h=12).

## 4. Modelos implementados

### Modelo 1: SARIMA(0,1,1)(0,1,1)₁₂

Identificado a partir del análisis de ACF/PACF (patrón característico del
"modelo Airline" de Box-Jenkins). Órdenes:
- **d=1, D=1:** diferenciación regular y estacional, confirmadas por el test ADF.
- **q=1, Q=1:** un único término de media móvil en cada nivel, sin componente AR.

### Modelo 2: XGBoost recursivo

Modelo de gradient boosting entrenado sobre atributos de calendario (mes, trimestre)
y rezagos de la propia serie (lags 1, 2, 3 y 12). El pronóstico para 2019 se genera
de forma recursiva: cada predicción mensual se incorpora al historial para construir
los lags del mes siguiente, sin utilizar en ningún momento los valores reales de 2019.

## 5. Resultados y métricas

| Modelo | RMSE (barriles) | MAE (barriles) | MAPE |
|---|---:|---:|---:|
| **SARIMA(0,1,1)(0,1,1)₁₂** | **583.913** | **430.181** | **2,82%** |
| XGBoost recursivo (lags 1-3 y 12) | 725.634 | 640.817 | 4,29% |

**SARIMA superó a XGBoost en las tres métricas de evaluación.** La comparación
es justa: ambos modelos fueron entrenados exclusivamente con datos hasta diciembre
de 2018 y pronosticaron los 12 meses de 2019 sin utilizar sus valores reales.

### Análisis de error por horizonte

El error absoluto no crece de forma monótona con el horizonte de pronóstico
(h=1 a h=12). Se observa un pico de error compartido por ambos modelos en
h=8 (agosto 2019), lo que sugiere un mes particularmente difícil de anticipar
para ambas metodologías. SARIMA presentó menor error que XGBoost en 10 de los
12 meses evaluados.

## 6. Conclusiones

1. **SARIMA obtuvo el mejor desempeño** en las tres métricas (MAPE 2,82% vs 4,29%).
2. **La comparación se realizó sin fuga de datos:** ambos modelos conocen únicamente
   información hasta diciembre de 2018 al momento de pronosticar.
3. **Un modelo más complejo no garantiza mayor precisión:** para una serie con
   estacionalidad marcada y volumen de datos moderado, el modelo estadístico clásico
   superó al modelo de Machine Learning.
4. **SARIMA presentó menor error en 10 de los 12 meses evaluados.** Ambos modelos
   registraron su mayor error en agosto de 2019.

## 7. Visualizaciones incluidas

- Serie temporal original (2008–2019).
- Descomposición en tendencia, estacionalidad y residuo.
- ACF y PACF de la serie diferenciada.
- División train/test.
- Pronóstico SARIMA vs valores reales (con intervalo de confianza 95%).
- Pronóstico XGBoost recursivo vs valores reales.
- Diagnóstico de residuales del modelo SARIMA (4 gráficos estándar).
- Error absoluto por horizonte de pronóstico (h=1 a h=12).
- Comparación de métricas finales (SARIMA vs XGBoost recursivo).

## 8. Limitaciones conocidas

- **Outlier puntual (2012-2013):** los residuales del modelo SARIMA presentan
  colas más pesadas que una distribución normal (Jarque-Bera=411.27,
  Kurtosis=12.28), atribuible a un valor atípico específico en ese período.
  No afecta la validez general del modelo (el correlograma de residuales no
  muestra autocorrelación significativa).
- **Quiebre estructural en `brewing_materials.csv`:** documentado como
  motivo del cambio de dataset (ver sección 2), no utilizado en el modelado final.
- **XGBoost con hiperparámetros fijos:** no se realizó búsqueda de hiperparámetros
  para evitar utilizar el conjunto de prueba durante la selección del modelo.
- **Propagación de errores en pronóstico recursivo:** al encadenar predicciones,
  los errores de XGBoost pueden acumularse de un mes al siguiente.
- **Ventana de evaluación única:** la evaluación final se realizó sobre un único
  período de 12 meses (2019).

## 9. Estructura del repositorio

```
proyecto-series-temporales-cerveza/
│
├── data/
│   └── beer_taxed.csv
│
├── notebooks/
│   └── Pérez_Russo_ST v1.ipynb
│
├── results/
│   ├── metricas.csv
│   ├── errores_por_horizonte.csv
│   ├── serie_temporal_original.png
│   ├── predicciones_vs_reales.png
│   ├── diagnostico_residuales_sarima.png
│   ├── error_por_horizonte.png
│   └── comparacion_modelos.png
│
├── README.md
└── requirements.txt
```

## 10. Cómo ejecutar el notebook

1. Abrir el notebook en Google Colab desde el repositorio.
2. El dataset se carga automáticamente desde la URL del repositorio en el Bloque 1,
   sin necesidad de subir ningún archivo manualmente.
3. Ejecutar todas las celdas en orden.
4. Al finalizar, el Bloque 19 exporta los resultados y genera un ZIP descargable
   con todas las métricas y visualizaciones del proyecto.
