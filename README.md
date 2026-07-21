# Modelo de Riesgo Corporativo de Quiebra

## Resumen ejecutivo
Modelo predictivo que estima la probabilidad de quiebra de una empresa a partir de sus estados financieros, usando datos reales de empresas polacas (UCI Machine Learning Repository). El proyecto cubre el ciclo completo de un analista de riesgos: diagnóstico y limpieza de datos, selección/interpretación de variables, modelado comparativo, y un dashboard de decisión en Power BI.

## El problema de negocio
Cuando un banco evalúa una solicitud de crédito empresarial, necesita estimar qué tan probable es que esa empresa incumpla. A diferencia del riesgo de crédito al consumo (comportamiento de una persona), el riesgo corporativo se lee en los estados financieros: liquidez, apalancamiento, rentabilidad. Este proyecto responde la pregunta central de originación de crédito PYME/corporativo: **¿qué tan sano está este negocio, y qué tan probable es que quiebre?**

## Los datos
- **Fuente:** UCI Machine Learning Repository — Polish Companies Bankruptcy Data (Zięba, Tomczak & Tomczak, 2016)
- **Archivo:** `5year.arff` — datos financieros del año más cercano a la quiebra, prediciendo incumplimiento a 1 año
- **Tamaño:** 5,910 empresas, 64 razones financieras (liquidez, apalancamiento, rentabilidad, eficiencia)
- **Desbalance:** 410 empresas en quiebra (6.9%) vs. 5,500 sanas

## Metodología

### 1. Diagnóstico y limpieza de datos
El dataset tiene valores faltantes significativos en varias columnas. Se probaron dos estrategias y se compararon **empíricamente**, no solo en teoría:

| Estrategia | Descripción | AUC-ROC (prueba rápida) |
|---|---|---|
| 1 — Básica | Imputar todos los nulos con la mediana, sin eliminar columnas | 0.9271 |
| 2 — Híbrida | Eliminar columnas con >40% de nulos (Attr37), luego imputar el resto con la mediana | **0.9290** |

**Se eligió la Estrategia 2**: eliminar Attr37 (43% de datos faltantes) dio mejor desempeño que "inventar" casi la mitad de sus valores — la calidad de los datos importó más que conservar todas las columnas.

### 2. Selección e interpretación de variables
Con 64 variables, la interpretación directa no es práctica. Se usó la importancia de variables de Random Forest para priorizar. Las 3 variables más influyentes fueron:

- **Margen Operativo** (Attr39) — Margen de beneficio sobre ventas
- **ROA Operativo** (Attr35) — Retorno sobre activos operativo
- **Cobertura de Gastos Financieros** (Attr27)

Esto reduce el riesgo corporativo a tres preguntas de negocio: ¿el negocio principal es rentable?, ¿son eficientes con lo que tienen?, ¿pueden pagar los intereses de su deuda?

*Nota de alcance: el modelo final se entrenó con las 63 variables restantes (no un subconjunto reducido), ya que Random Forest maneja bien alta dimensionalidad. El análisis de importancia se usó para interpretación de negocio, no para reducción de features.*

### 3. Modelado
- **Baseline:** Regresión Logística (escalado con `StandardScaler`, `class_weight='balanced'`)
- **Ensamble:** Random Forest (`class_weight='balanced'`, sin necesidad de escalado)
- División estratificada 80/20 para preservar la proporción de quiebras (6.9%) en ambos conjuntos

### 4. Resultados (Random Forest, conjunto de prueba, n=1,182)

| Métrica | Valor |
|---|---|
| AUC-ROC (Regresión Logística) | 0.8187 |
| AUC-ROC (Random Forest) | 0.9143 |
| Coeficiente Gini (Random Forest) | 0.8286 |
| Estadístico KS (Random Forest) | 0.7067 |
| Exactitud (Accuracy) | 91.7% |
| Precisión (clase quiebra) | 41.5% |
| Sensibilidad / Recall (clase quiebra) | 47.6% |
| Especificidad | 95.0% |

**Matriz de confusión:**

| | Predicho: Sana | Predicho: Quiebra |
|---|---|---|
| **Real: Sana** | 1,045 (VN) | 55 (FP) |
| **Real: Quiebra** | 43 (FN) | 39 (VP) |

**Lectura de negocio:** el modelo aprobó correctamente a 1,045 empresas sanas y detectó 39 de las 82 quiebras reales del conjunto de prueba. Los 43 falsos negativos (empresas que el modelo marcó como seguras y terminaron quebrando) son el error más costoso para un banco — representan pérdida crediticia directa — mientras que los 55 falsos positivos son solo costo de oportunidad (negar crédito a una empresa sana). Este desbalance en el costo de los errores es la base para decidir, en un caso real, si conviene mover el umbral de decisión por debajo de 0.5 para capturar más verdaderos positivos a cambio de más falsos positivos.

## Dashboard (Power BI)
El modelo se aplicó a las 5,910 empresas para generar un semáforo de riesgo:

| Categoría | Empresas | % del total |
|---|---|---|
| Bajo Riesgo (Seguro) | 4,610 | 78.0% |
| Riesgo Medio (Monitorear) | 758 | 12.8% |
| Alto Riesgo (Alerta de Quiebra) | 542 | 9.2% |

El panel (`Riesgo_Corporativo.pbix`) incluye la probabilidad de quiebra y el semáforo de riesgo por empresa, junto con las 3 variables financieras clave ya renombradas para lectura de negocio.

*Nota: a diferencia de Tableau Public, Power BI no tiene una versión gratuita de publicación web abierta sin una cuenta de Power BI Service. Para que cualquiera pueda ver el dashboard sin instalar nada, exporta 2-3 capturas de pantalla del panel como PNG e inclúyelas en este README (sección de abajo), y sube el `.pbix` al repo para quien sí quiera abrirlo en Power BI Desktop.*



## Cómo correr este proyecto
```
1. Descargar 5year.arff de UCI ML Repository y colocarlo en la carpeta datos/
2. pip install pandas scikit-learn scipy matplotlib seaborn --break-system-packages
3. Correr notebook_riesgo_corporativo.ipynb de principio a fin
4. Abrir Riesgo_Corporativo.pbix en Power BI Desktop para explorar el dashboard
```

## Qué le aporta a un rol de riesgos bancarios
Este proyecto demuestra el mismo ciclo de trabajo que pide un área de riesgos: diagnóstico de calidad de datos, decisiones justificadas con evidencia (no solo intuición), modelado comparativo, traducción de resultados técnicos a lenguaje de negocio, y pensamiento explícito sobre el costo asimétrico de los errores — aplicado aquí a riesgo corporativo en vez de crédito al consumo.
