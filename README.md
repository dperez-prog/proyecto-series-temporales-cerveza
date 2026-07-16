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

1. **Análisis exploratorio (EDA):** visualización de la serie, comparación con
   insumos relacionados (malta, cereales, lúpulo) como contexto.
2. **Test de estacionariedad:** Dickey-Fuller aumentado (ADF) sobre la serie
   original y sobre la serie diferenciada.
3. **Descomposición:** separación en tendencia, estacionalidad y residuo
   (modelo aditivo, período=12).
4. **Identificación de órdenes SARIMA:** análisis de ACF/PACF sobre la serie
   diferenciada (estacional + regular).
5. **División train/test:** últimos 12 meses (2019) como conjunto de prueba,
   respetando el orden temporal.
6. **Ingeniería de atributos (XGBoost):** variables de calendario (mes,
   trimestre) y variables de rezago (lags), construidas sin fuga de datos.
7. **Evaluación:** métricas RMSE, MAE y MAPE sobre el conjunto de prueba,
   además de un análisis de error por horizonte de pronóstico (h=1 a h=12).
8. **Optimización de hiperparámetros de diseño:** comparación de ventanas de
   entrenamiento (Cuestión 1) y configuraciones de lags (Cuestión 2).

## 4. Modelos implementados

### Modelo 1: SARIMA(0,1,1)(0,1,1)₁₂

Identificado a partir del análisis de ACF/PACF (patrón característico del
"modelo Airline" de Box-Jenkins). Órdenes:
- **d=1, D=1:** diferenciación regular y estacional, confirmadas por el test ADF.
- **q=1, Q=1:** un único término de media móvil en cada nivel, sin componente AR.

### Modelo 2: XGBoost (Machine Learning)

Modelo de gradient boosting entrenado sobre atributos de calendario (mes,
trimestre) y rezagos de la propia serie. Configuración final optimizada:
lags 1, 2 y 3 (ver Cuestión 2).

## 5. Resultados y métricas

| Modelo | RMSE (barriles) | MAE (barriles) | MAPE |
|---|---|---|---|
| **SARIMA(0,1,1)(0,1,1)₁₂** | **583,913** | **430,181** | **2.82%** |
| XGBoost (lags 1-3, óptimo) | 694,718 | 585,715 | 4.00% |

**SARIMA superó a XGBoost en las tres métricas de evaluación**, incluso luego
de optimizar la configuración de atributos del modelo de ML.

### Análisis de error por horizonte

El error absoluto no crece de forma monótona con el horizonte de pronóstico
(h=1 a h=12). Se observa un pico de error compartido por ambos modelos en
h=8 (agosto 2019), lo que sugiere un mes particularmente difícil de anticipar
para ambas metodologías, más que una pérdida de precisión asociada a la
distancia temporal del pronóstico.

## 6. Respuestas a las cuestiones metodológicas

### Cuestión 1: ¿Ventana deslizante o todos los datos?

Se probaron ventanas de entrenamiento de 24, 36, 48, 60, 72, 96 meses y el
histórico completo (132 meses), evaluando el MAPE resultante sobre el mismo
conjunto de prueba.

**Resultado:** el MAPE disminuye a medida que aumenta el tamaño de la ventana,
alcanzando su mínimo (2.82%) al usar **todo el histórico disponible**. Se
observó una excepción puntual en la ventana de 36 meses (MAPE=7.12%), que se
interpreta como un tramo particularmente inestable de la serie más que un
patrón general de las ventanas cortas.

**Conclusión:** para esta serie, no conviene usar ventana deslizante; es
preferible entrenar con todos los datos disponibles.

### Cuestión 2: ¿Qué intervalo del pasado es óptimo como input?

Se probaron distintas configuraciones de lags para XGBoost: solo lag_1,
lags 1-3, lags 1-6, lags 1-3 y 12, y lags 1-12.

**Resultado:** la configuración de **lags 1 a 3** obtuvo el menor MAPE (4.00%),
superando incluso a configuraciones que incluían el lag_12 (que buscaba
capturar explícitamente la estacionalidad anual).

**Conclusión:** un intervalo corto del pasado (3 meses) es suficiente y
preferible, dado que las variables de calendario (mes, trimestre) ya aportan
la información estacional necesaria; agregar lags más lejanos introduce
ruido en lugar de ayuda.

## 7. Visualizaciones incluidas

- Serie temporal original y comparación con insumos relacionados (malta,
  cereales, lúpulo).
- Descomposición en tendencia, estacionalidad y residuo.
- ACF y PACF de la serie diferenciada.
- División train/test.
- Pronóstico SARIMA vs valores reales (con intervalo de confianza 95%).
- Pronóstico XGBoost vs valores reales.
- Diagnóstico de residuales del modelo SARIMA (4 gráficos estándar).
- Error absoluto por horizonte de pronóstico (h=1 a h=12).
- Comparación de MAPE por ventana de entrenamiento.
- Comparación de MAPE por configuración de lags.
- Tabla y gráfico comparativo final (SARIMA vs XGBoost).

## 8. Limitaciones conocidas

- **Outlier puntual (2012-2013):** los residuales del modelo SARIMA presentan
  colas más pesadas que una distribución normal (Jarque-Bera=411.27,
  Kurtosis=12.28), atribuible a un valor atípico específico en ese período.
  No afecta la validez general del modelo (el correlograma de residuales no
  muestra autocorrelación significativa).
- **Quiebre estructural en `brewing_materials.csv`:** documentado como
  motivo del cambio de dataset (ver sección 2), no utilizado en el
  modelado final.

## 9. Conclusiones

Para la serie de producción mensual de cerveza en EE.UU., un modelo
estadístico clásico (SARIMA) superó a un modelo de Machine Learning
(XGBoost) en las tres métricas de evaluación, incluso tras optimizar su
configuración de atributos. Esto confirma que, en series con estacionalidad
fuerte, tendencia suave y volumen de datos moderado, la sofisticación de un
modelo no garantiza mejor desempeño frente a métodos clásicos bien
especificados. El modelo SARIMA entrenado con todo el histórico disponible
ofrece un pronóstico confiable (error promedio ~2.8%), útil para la
planificación de insumos, logística y capacidad productiva de la industria
cervecera.

## Estructura del repositorio

```
proyecto_ts/
│
├── data/
│   └── beer_taxed.csv
│
├── notebooks/
│   └── forecasting_produccion_cerveza.ipynb
│
├── results/
│   ├── metricas.csv
│   └── graficos.png
│
├── README.md
└── requirements.txt
```

## Cómo ejecutar el notebook

1. Descargar el dataset desde [Kaggle](https://www.kaggle.com/datasets/jessemostipak/beer-production).
2. Abrir el notebook en Google Colab.
3. Ejecutar la celda de carga de archivo (`from google.colab import files; files.upload()`)
   y subir manualmente `beer_taxed.csv` cuando se solicite.
4. Ejecutar el resto de las celdas en orden.
