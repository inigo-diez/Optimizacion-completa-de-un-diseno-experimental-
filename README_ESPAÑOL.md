# Optimización de las Condiciones de Experimentación GC-MS/SPME

Este repositorio recoge el análisis completo de un experimento de tipo Diseño Central Compuesto (CCD) orientado a optimizar las condiciones de extracción por microextracción en fase sólida (SPME) acoplada a cromatografía de gases-espectrometría de masas (GC-MS) para el estudio del metaboloma fecal. El objetivo principal fue identificar qué combinación de tiempos de extracción, desorción e incubación maximiza la detección e intensidad de metabolitos de interés clínico, con especial atención a los compuestos asociados al eje microbiota-cerebro.

**Datos no corresponden a pacientes o controles, son propios.**

---

## Contexto y motivación

El análisis metabolómico de muestras fecales mediante GC-MS requiere una etapa de extracción que es altamente sensible a las condiciones operativas. Pequeñas variaciones en el tiempo de incubación de la fibra, en la temperatura de desorción o en el tiempo de exposición al headspace pueden traducirse en diferencias sustanciales en la cobertura y la intensidad de los perfiles cromatográficos.

En lugar de optimizar las condiciones de forma empírica, se diseñó un experimento estructurado siguiendo la metodología de superficies de respuesta (RSM), complementado con modelos de aprendizaje automático supervisado. La idea fue doble: extraer el máximo de información con el mínimo número de experimentos y comparar si los modelos de ML son capaces de identificar tendencias que el RSM clásico no captura con N=17.

---

**Tabla 1. Arquitectura del notebook y función de cada bloque**

| Bloque | Pregunta que responde | Papel en el flujo |
|--------|----------------------|-------------------|
| 1. Validación preliminar | ¿Los datos de partida son consistentes? | Asegura que el diseño, la matriz de áreas y la homogeneidad de muestra permiten modelar con sentido. |
| 2. Codificación del DOE | ¿Cómo se llevan los factores a una escala común? | Convierte extracción, desorción e incubación a coordenadas codificadas comparables. |
| 3. Respuestas y D_global | ¿Qué significa que una corrida sea buena en términos generales? | Define métricas cuantitativas de rendimiento y una desirability global. |
| 4. D_biomarkers | ¿Qué condiciones favorecen específicamente a los biomarcadores prioritarios? | Introduce una función objetivo dirigida a indoles, p-cresol, methional y ácidos carboxílicos. |
| 5. Exploración DOE | ¿Qué tendencias básicas aparecen antes del modelado formal? | Distribuciones, relaciones simples, reproducibilidad e interacciones. |
| 6. RSM | ¿Puede un modelo cuadrático describir la superficie de respuesta? | Ajusta modelos clásicos DOE y propone óptimos teóricos. |
| 7. Machine Learning | ¿Qué modelo predice mejor con N=17? | Compara PLS, RandomForest, XGBoost, GaussianProcess y RSM. |
| 8. Interpretabilidad | ¿Qué factor domina el comportamiento del sistema? | Usa SHAP, permutation importance y PDP para jerarquizar efectos. |
| 9. Optimización final | ¿Cómo se traducen los óptimos a minutos reales? | Pasa del espacio codificado a condiciones operativas para el protocolo. |

---

## Diseño experimental

Se utilizó un Diseño Central Compuesto (CCD) con tres factores y 17 experimentos en total: 8 puntos factoriales (Experimentos 1–8), 6 puntos axiales (Experimentos 9, 10, 11, 12, 13 y 17) y 3 puntos centrales (Experimentos 14, 15 y 16).

**Tabla 2. Validación preliminar del diseño experimental**

| Elemento | Resultado | Para qué se comprueba |
|----------|----------|----------------------|
| Diseño experimental | 17 experimentos × 7 columnas | Base estructural del CCD |
| Matriz de áreas | 17 experimentos × 140 metabolitos | Matriz cuantitativa sobre la que se construyen las respuestas |
| Metabolitos de interés | 123 en lista; 26 detectados y utilizables | Panel diana inicial para análisis dirigido |
| Peso de muestra | 32.9 ± 0.6 mg; CV = 1.86% | Homogeneidad alta; el peso no introduce variabilidad relevante |
| Tipos de punto | 8 factoriales, 6 axiales, 3 centrales | Cobertura balanceada del espacio experimental |

El matching entre los 123 metabolitos diana y la matriz de áreas real identifica solo 26 compuestos utilizables. Este dato es metodológicamente importante: la optimización no persigue maximizar cualquier señal, sino una fracción concreta y clínicamente significativa del perfil metabolómico.

**Tabla 3. Parámetros de codificación de los factores experimentales**

| Variable codificada | Factor experimental | Mín real (min) | Máx real (min) | Rango |
|---------------------|---------------------|---------------|---------------|-------|
| $x_1$ | Tiempo de extracción en headspace (fibra SPME) | 0.6 | 120.0 | 119.4 |
| $x_2$ | Tiempo de desorción térmica en el inyector | 0.3 | 3.7 | 3.4 |
| $x_3$ | Tiempo de incubación previa de la muestra | 1.7 | 19.2 | 17.5 |

A lo largo del análisis, $x_1$, $x_2$ y $x_3$ se usan sistemáticamente para referirse a estos factores en escala codificada $[-1, +1]$. La transformación directa y su inversa son:

$$x_\text{cod} = \frac{2\,(x_\text{real} - x_\text{min})}{x_\text{max} - x_\text{min}} - 1$$

$$x_\text{real} = x_\text{min} + \frac{x_\text{cod} + 1}{2}\,(x_\text{max} - x_\text{min})$$

**Figura 1 — Validación preliminar**

![Figura 1](Figuras/01_validacion_preliminar.png)

Tres paneles con la distribución de tipos de punto en el CCD, la evolución del peso de muestra entre corridas y la cobertura espacial del diseño. El panel de peso confirma la excelente homogeneidad (CV = 1.86%); el scatter espacial muestra que el dominio se ha explorado sin sesgos visibles.

**Figura 2 — Diseño CCD codificado**

![Figura 2](Figuras/02_design_matrix_coded.png)

Representación del CCD en el espacio codificado para los tres pares de factores ($x_1$-$x_2$, $x_1$-$x_3$, $x_2$-$x_3$). Confirma que las esquinas factoriales, extremos axiales y réplicas centrales se sitúan en la geometría esperada del diseño compuesto central.

---

## Variables de respuesta y función D_global

A partir de los 140 metabolitos cuantificados por área normalizada, se construyeron cuatro respuestas complementarias que cubren distintas dimensiones del rendimiento analítico.

**Tabla 4. Estadísticos descriptivos de las respuestas construidas**

| Respuesta | Mínimo | Máximo | Media | SD |
|-----------|--------|--------|-------|-----|
| Y_detected_count (n picos totales) | 78 | 140 | 132.82 | 14.63 |
| Y_interest_intensity (suma áreas diana) | 2.35 × 10⁷ | 1.77 × 10⁹ | 1.21 × 10⁹ | 4.80 × 10⁸ |
| Y_precision_score | 0.00236 | 0.00391 | 0.00252 | 0.00037 |
| Y_interest_detected (n picos diana) | 21 | 37 | 34.76 | 3.72 |

A partir de estas cuatro componentes se construye D_global (desirability compuesta con pesos por escenario). En el escenario base, D_global alcanza una media de 0.311 y un máximo de 0.449.

**Tabla 5. Cinco mejores corridas según D_global (escenario base)**

| Experimento | Tipo | D_global | Y_detected | Y_interest_detected |
|-------------|------|----------|-----------|---------------------|
| Exp. 10 | Axial | 0.4490 | 135 | 35 |
| Exp. 8 | Factorial | 0.4421 | 136 | 35 |
| Exp. 11 | Axial | 0.4298 | 135 | 36 |
| Exp. 4 | Factorial | 0.4031 | 129 | 33 |
| Exp. 2 | Factorial | 0.3900 | 129 | 34 |

**Figura 3 — D_global por escenario de ponderación**

![Figura 3](Figuras/03_desirability_escenarios.png)

Los tres escenarios producen distintas medias pero mantienen el mismo ranking en cabeza (experimentos 10, 8, 11). El experimento 9 aparece sistemáticamente como el peor en todos los escenarios. La estabilidad del ranking valida que el índice no es artefacto de una elección concreta de pesos.

---

## D_biomarkers: función objetivo para familias prioritarias

En paralelo a D_global, se construyó una segunda función objetivo centrada exclusivamente en cuatro familias de biomarcadores con relevancia clínica para el eje microbiota-intestino-cerebro.

**Tabla 6. Familias de metabolitos integradas en D_biomarkers**

| Familia | Compuestos agregados | Relevancia |
|---------|---------------------|-----------|
| Indoles | 4 | Biomarcadores del metabolismo del triptófano por la microbiota; relevantes en el eje intestino-cerebro |
| p-Cresol | 2 | Indicador de actividad del metabolismo proteico bacteriano |
| Methional | 2 | Compuesto azufrado volátil; familia más heterogénea y penalizadora del índice |
| Ácidos carboxílicos | 6 | Ácidos grasos de cadena corta y ramificada; señal del metabolismo fermentativo colónico |

Cada familia se normaliza a $[0, 1]$ según el formalismo de Derringer y Suich (1980). Para respuestas de tipo "cuanto mayor, mejor", la deseabilidad individual $d_i$ se define como:

$$d_i = \begin{cases} 0 & \text{si } y_i \leq L \\ \left(\dfrac{y_i - L}{U - L}\right)^s & \text{si } L < y_i < U \\ 1 & \text{si } y_i \geq U \end{cases}$$

donde $L$ es el límite inferior aceptable, $U$ es el valor objetivo y $s$ es un parámetro de forma ($s = 1$ equivale a interpolación lineal). En la implementación adoptada, siguiendo la adaptación de Jové et al. (2021), se tomó $L = 0$, $U = \max(y_i)$ y $s = 1$, de modo que $d_i = y_i / \max(y_i)$.

La deseabilidad compuesta se calcula como la media geométrica de las cuatro componentes:

$$D_{\text{biomarkers}} = \left(d_{\text{Indoles}} \cdot d_{\text{p-Cresol}} \cdot d_{\text{Methional}} \cdot d_{\text{Ácidos}}\right)^{1/4}$$

La propiedad clave de la media geométrica es que si cualquier $d_i = 0$, el índice compuesto colapsa a cero, garantizando que la solución óptima no sacrifique ninguna familia de biomarcadores.

**Tabla 7. Resumen cuantitativo del índice D_biomarkers**

| Indicador | Resultado |
|-----------|----------|
| Rango D_biomarkers | 0.000 – 0.839 |
| Media ± SD | 0.493 ± 0.284 |
| CV% | 57.7% |
| Top 1: Exp. 5 (Factorial) | D = 0.839 — mejor observado global |
| Top 2: Exp. 12 (Axial) | D = 0.775 |
| Top 3: Exp. 16 (Central) | D = 0.762 — mejor punto central |
| Bottom 1: Exp. 9 (Axial) | D = 0.000 — fallo total en todas las familias |
| Bottom 2: Exp. 2 (Factorial) | D = 0.136 |
| Bottom 3: Exp. 3 (Factorial) | D = 0.148 |
| Media Centrales (n=3) | 0.741 ± 0.021 |
| Media Axiales (n=6) | 0.540 ± 0.304 |
| Media Factoriales (n=8) | 0.364 ± 0.268 |

La diferencia central-factorial ($\Delta = 0.377$) es estadísticamente relevante dado el rango del índice. Los puntos centrales son la región del diseño más favorable y más repetible para la extracción conjunta de biomarcadores.

**Figura 4 — Distribuciones de desirabilidad individual por familia**

![Figura 4](Figuras/06_01_desirability_distributions.png)

Histogramas de $d_\text{indoles}$, $d_\text{p-cresol}$, $d_\text{methional}$ y $d_\text{carbox}$. p-Cresol e indoles presentan medias relativamente altas (>0.6) y distribuciones más simétricas. Methional muestra la mayor asimetría y el mayor número de experimentos en zona baja ($d < 0.3$ en experimentos 1, 2, 3, 4, 6, 7, 9 y 17), siendo la familia que más penaliza la media geométrica.

**Figura 5 — Distribución de D_biomarkers**

![Figura 5](Figuras/06_02_desirability_global.png)

El histograma cubre prácticamente todo el rango [0, 0.84] con media ≈ 0.49 y amplia dispersión. La ausencia de colapso en una franja estrecha indica que el índice discrimina bien entre condiciones experimentales, condición necesaria para que el modelado posterior tenga señal que aprender.

**Figura 6 — D_biomarkers por tipo de diseño**

![Figura 6](Figuras/06_03_desirability_by_classification.png)

Los tres tipos de diseño muestran comportamientos claramente distintos. Los centrales se sitúan en zona alta (media 0.74) con dispersión mínima (std = 0.021). Los axiales son los más variables e incluyen el peor caso (exp. 9, D = 0). Los factoriales tienen la media más baja y alta dispersión, coherente con la exploración de esquinas del dominio.

**Figura 7 — D_biomarkers por experimento**

![Figura 7](Figuras/06_04_desirability_experiments.png)

Perfil experimento a experimento. Exp. 5 como máximo absoluto y exp. 9 como mínimo. No hay tendencia monótona por número de experimento, lo que confirma que el rendimiento biomarcador depende de la combinación concreta de ($x_1$, $x_2$, $x_3$) y no de factores de orden de corrida.

**Figura 8 — Matriz de correlación entre desirabilidades**

![Figura 8](Figuras/06_05_correlation_matrix.png)

Las correlaciones con D_biomarkers son: Methional $r = 0.910$, Indoles $r = 0.902$, p-Cresol $r = 0.858$, Ácidos carboxílicos $r = 0.700$. El hecho de que mejorar cualquier familia tiende a arrastrar al resto en la misma dirección justifica la media geométrica como índice conjunto: hay sinergia real entre familias.

---

## Exploración DOE y análisis factorial

**Tabla 8. Repetibilidad en los tres puntos centrales (Exp. 14, 15, 16)**

| Respuesta | Exp. 14 | Exp. 15 | Exp. 16 | Media | CV% | Evaluación |
|-----------|---------|---------|---------|-------|-----|-----------|
| Y_detected_count | 140 | 139 | 139 | 139.33 | 0.4% | Excelente |
| Y_interest_detected | 37 | 37 | 36 | 36.67 | 1.6% | Excelente |
| Y_interest_intensity | 1.229×10⁹ | 1.617×10⁹ | 1.471×10⁹ | 1.439×10⁹ | 13.6% | Aceptable |
| Y_precision_score | 0.0024 | 0.0024 | 0.0024 | 0.0024 | 0.0% | Excelente |
| D_global | 0.005 | 0.312 | 0.299 | 0.205 | 84.6% | Alta variabilidad |

Varias respuestas muestran excelente repetibilidad en los puntos centrales, pero D_global presenta variabilidad alta (CV = 84.6%). Esto advierte que la función combinada es muy sensible a pequeñas diferencias entre respuestas parciales, especialmente por la componente de precisión.

Resultados adicionales del análisis exploratorio:

- Correlación de Pearson entre $x_1$ e Y_interest_intensity: **r = 0.825** (la más alta del dataset).
- Efecto principal más fuerte: $x_1$ sobre Y_interest_detected, con salto positivo claro entre nivel bajo (−1) y nivel alto (+1).
- Interacción más relevante: $x_2 \times x_3$ (desorción × incubación) sobre Y_interest_detected.
- El efecto de $x_3$ (incubación) es ligeramente negativo al pasar de −1 a +1, lo que sugiere que tiempos de incubación muy largos pueden favorecer procesos de degradación térmica o competencia por la fibra SPME.

**Figura 9 — Distribución de respuestas**

![Figura 9](Figuras/04_01_response_distributions.png)

Histogramas de las cuatro respuestas. Las respuestas de conteo muestran sesgo hacia valores altos; la intensidad es la más dispersa; el score de precisión ocupa una banda estrecha y baja en todos los experimentos.

**Figura 10 — Factores vs Y_interest_detected**

![Figura 10](Figuras/04_02_factors_vs_response.png)

El panel de $x_1$ muestra una nube con pendiente positiva visible. Los paneles de $x_2$ y $x_3$ no muestran tendencias claras, confirmando que el efecto lineal simple de desorción e incubación sobre detectabilidad es pequeño frente al de extracción.

**Figura 11 — Espacio experimental 3D**

![Figura 11](Figuras/04_02b_3d_design_space.png)

Visualización en 3D de los 17 experimentos en el dominio ($x_1$, $x_2$, $x_3$), con colores por tipo de punto y tamaño escalado con la respuesta. Permite verificar que no hay acumulación excesiva en ninguna región.

**Figura 12 — Heatmap de correlación factores-respuestas**

![Figura 12](Figuras/04_03_correlation_matrix.png)

$x_1$ correlaciona positiva y significativamente con Y_detected, Y_intensity y D_biomarkers. Presenta correlación negativa con el score de precisión, lo que explica por qué D_global no crece en la misma proporción que Y_intensity al aumentar $x_1$.

**Figura 13 — Boxplots por clasificación DOE**

![Figura 13](Figuras/04_05_boxplots_classification.png)

Los puntos centrales presentan la menor dispersión en todas las respuestas. Los axiales concentran los casos extremos. Los factoriales muestran dispersión amplia, especialmente en intensidad.

**Figura 14 — Efectos principales**

![Figura 14](Figuras/04_06_main_effects.png)

El salto de $x_1$ de nivel bajo a nivel alto es claramente el más grande. $x_2$ muestra un cambio mínimo. $x_3$ presenta un efecto negativo moderado al pasar de −1 a +1.

**Figura 15 — Gráficos de interacción**

![Figura 15](Figuras/04_07_interaction_plots.png)

Las líneas de interacción $x_1 \times x_2$ y $x_1 \times x_3$ son aproximadamente paralelas, indicando que el efecto de $x_1$ no depende fuertemente del nivel de $x_2$ o $x_3$. La interacción $x_2 \times x_3$ sí muestra líneas divergentes, lo que justifica incluir el término de interacción $x_2 \cdot x_3$ en el modelo RSM.

---

## Modelado RSM (Response Surface Methodology)

Se ajustaron tres modelos cuadráticos de segundo orden, uno por target. La forma general del modelo es:

$$\hat{y} = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \beta_3 x_3 + \beta_{11} x_1^2 + \beta_{22} x_2^2 + \beta_{33} x_3^2 + \beta_{12} x_1 x_2 + \beta_{13} x_1 x_3 + \beta_{23} x_2 x_3$$

La validación se realizó exclusivamente con LOOCV dado que N = 17.

**Tabla 9. Métricas de ajuste y validación cruzada de los modelos RSM**

| Modelo | Target | R²_train | R²_LOOCV | RMSE_train | RMSE_LOOCV | Diagnóstico |
|--------|--------|----------|----------|-----------|-----------|------------|
| RSM_1 | Y_interest_detected | 0.813 | −0.613 | 1.56 | 4.58 | Sobreajuste alto |
| RSM_2 | Y_interest_intensity | 0.919 | +0.330 | 1.33×10⁸ | 3.81×10⁸ | Moderado |
| RSM_3 | D_biomarkers | 0.891 | −0.083 | 0.091 | 0.287 | Sobreajuste alto |

Los tres modelos ajustan bien en entrenamiento ($R^2$ entre 0.81 y 0.92), pero solo RSM_2 generaliza con capacidad moderada. RSM_1 y RSM_3 tienen $R^2_\text{LOOCV}$ negativos: predicen peor que usar simplemente la media. El diagnóstico es estructural: un modelo de segundo orden completo con 10 términos aplicado a 17 puntos deja muy poco grado de libertad para validación cruzada.

En los tres modelos, $x_1$ y $x_1^2$ son los términos de mayor importancia (medida como $|\beta| \times -\log_{10}(p)$). Para RSM_1: importancia de $x_1 = 13.908$, $p = 0.0019$ (***). Para D_biomarkers también destacan $x_2 \cdot x_3$ y $x_3^2$, coherente con la interacción detectada en el análisis factorial.

**Tabla 10. Óptimos teóricos predichos por RSM en unidades reales**

| Modelo | Target | $x_1$ | $x_2$ | $x_3$ | t_ext (min) | t_des (min) | t_inc (min) | Y* predicho |
|--------|--------|--------|--------|--------|------------|------------|------------|------------|
| RSM_1 | Y_detected | +0.532 | −0.590 | −1.000 | 92.1 | 1.00 | 1.7 | 39.04 metabolitos |
| RSM_2 | Y_intensity | +1.000 | +0.084 | +1.000 | 120.0 | 2.14 | 19.2 | 1.887 × 10⁹ |
| RSM_3 | D_biomarkers | +1.000 | −1.000 | +1.000 | 120.0 | 0.30 | 19.2 | 1.36 (*) |

(*) $D_\text{biomarkers} > 1$ indica extrapolación fuera del espacio de datos; el valor real está acotado a $[0, 1]$. Este resultado es señal adicional de sobreajuste.

**Figura 16 — Parity plots RSM**

![Figura 16](Figuras/05_02_rsm_parity_plots.png)

RSM_2 sigue razonablemente bien la diagonal, coherente con su mejor $R^2_\text{LOOCV}$. RSM_1 muestra puntos que se alejan sistemáticamente en los extremos. RSM_3 presenta el patrón más errático, explicable por la naturaleza de D_biomarkers como media geométrica.

**Figura 17 — Superficies de respuesta RSM**

![Figura 17](Figuras/05_03_rsm_surface_contour.png)

Las superficies confirman que las zonas de mayor respuesta tienden a concentrarse en la región de $x_1$ alto. La curvatura no es despreciable, justificando $x_1^2$ en el modelo. Para D_biomarkers, las superficies muestran un pico fuera del rango $[-1, 1]$, generando el problema de extrapolación comentado.

**Figura 18 — Ranking de términos RSM**

![Figura 18](Figuras/05_06_rsm_importance_ranking.png)

Gráfico de barras que sitúa a $x_1$ y $x_1^2$ muy por encima del resto en RSM_1 y RSM_2. En RSM_3 aparecen además $x_2 \cdot x_3$ y $x_3^2$ con importancias comparables. Las barras en rojo ($p < 0.05$) distinguen términos estadísticamente significativos.

**Figura 19 — Diagnóstico de residuos RSM**

![Figura 19](Figuras/05_04_rsm_residuals_diagnostic.png)

Los residuos de RSM_2 pasan el test de Shapiro-Wilk ($p > 0.05$). RSM_1 y RSM_3 muestran desviaciones más marcadas. El problema principal del RSM no es la distribución de residuos sino la ratio datos/parámetros (17 observaciones para un modelo de 10 términos).

---

## ML Supervisado: comparación de modelos

Se compararon cuatro algoritmos de ML contra RSM con LOOCV como protocolo de validación. Todos los modelos trabajan con las tres variables codificadas ($x_1$, $x_2$, $x_3$) como entradas únicas.

- **PLS** (Partial Least Squares, n_components=2)
- **RandomForest** (n_estimators=50, max_depth=4, min_samples_leaf=2)
- **XGBoost** (n_estimators=50, max_depth=3, learning_rate=0.1)
- **GaussianProcess** (kernel por defecto)

**Tabla 11. Ranking global de generalización ($R^2_\text{CV}$ promedio en los tres targets)**

| Modelo | Y_detected | Y_intensity | D_biomarkers | $R^2_\text{CV}$ medio | Ranking |
|--------|-----------|------------|-------------|----------------------|---------|
| RandomForest | −0.127 | +0.557 | +0.487 | **0.306** | 1º |
| XGBoost | +0.136 | +0.468 | +0.213 | **0.272** | 2º |
| PLS | −0.197 | +0.448 | +0.081 | **0.111** | 3º |
| RSM | −0.613 | +0.330 | −0.083 | **−0.122** | 4º |
| GaussianProcess | −5.127 | +0.701 | +0.205 | **−1.407** | 5º |

RandomForest es el modelo más equilibrado entre targets. XGBoost es el único que consigue $R^2_\text{CV}$ positivo para Y_detected (0.136). GaussianProcess es el mejor modelo individual para Y_intensity ($R^2_\text{CV} = 0.70$), pero el peor en generalización global por el colapso en Y_detected.

El ranking compuesto integra cuatro criterios ponderados: Desempeño 40% ($R^2_\text{CV}$), Interpretabilidad 30%, Robustez 20% y Complejidad 10%. Puntuaciones finales: RandomForest = 36.0, PLS = 35.1, RSM = 34.2, XGBoost = 31.1, GaussianProcess = 8.8.

**Tabla 12. Convergencia entre métodos de interpretabilidad (factor dominante)**

| Target | Factor nº1 por SHAP | Factor nº1 por permutación | Acuerdo |
|--------|---------------------|---------------------------|---------|
| Y_detected | $x_1$ (extracción) | $x_1$ (extracción) | Total |
| Y_intensity | $x_1$ (extracción) | $x_1$ (extracción) | Total |
| D_biomarkers | $x_1$ (extracción) | $x_1$ (extracción) | Total |

Los tres métodos convergen sin excepción: $x_1$ es el factor dominante para todos los targets. RandomForest y XGBoost asignan aproximadamente el **70% de la importancia total a $x_1$**, el ~20% a $x_3$ y el ~10% restante a $x_2$.

**Figura 20 — Parity plots ML**

![Figura 20](Figuras/06_04_ml_parity_plots.png)

Y_intensity presenta los parity plots más ordenados alrededor de la diagonal en todos los modelos. Y_detected muestra la mayor dispersión. Para D_biomarkers, RandomForest es el que mejor sigue la diagonal. GaussianProcess destaca visualmente en Y_intensity, coherente con su $R^2_\text{CV} = 0.70$.

**Figura 21 — Feature importance RF y XGBoost**

![Figura 21](Figuras/06_05_feature_importance.png)

Ambos modelos asignan ~70% de la importancia a $x_1$. Esta jerarquía es completamente coherente con los efectos principales del DOE y con los coeficientes RSM.

**Figura 22 — Curvas de aprendizaje**

![Figura 22](Figuras/06_06_learning_curves.png)

La brecha entre rendimiento en entrenamiento y en validación es notable para N = 17. Los modelos no han alcanzado su capacidad de generalización máxima. Ampliar el DOE con más puntos experimentales mejoraría la predicción más que cualquier ajuste adicional de hiperparámetros.

**Figura 23 — Correlación entre predicciones**

![Figura 23](Figuras/06_07_predictions_correlation.png)

Para Y_intensity, las correlaciones entre modelos son altas ($r > 0.85$), indicando que todos los algoritmos están aprendiendo la misma señal real. Para D_biomarkers el consenso es menor, coherente con la mayor variabilidad de ese target.

---

## Validación cruzada: análisis de residuos

**Tabla 13. Normalidad de residuos LOOCV — test de Shapiro-Wilk**

| Target | PLS | RandomForest | XGBoost |
|--------|-----|-------------|---------|
| Y_detected | $p = 0.001$ — no normal | $p = 0.000$ — no normal | $p = 0.000$ — no normal |
| Y_intensity | $p = 0.218$ — normal | $p = 0.129$ — normal | $p = 0.840$ — normal |
| D_biomarkers | $p = 0.401$ — normal | $p = 0.231$ — normal | $p = 0.368$ — normal |

Y_detected es el target problemático: los residuos LOOCV rechazan la normalidad en los tres modelos. Y_intensity y D_biomarkers presentan residuos compatibles con normalidad, lo que hace más interpretables sus métricas de ajuste.

**Figura 24 — Residuos vs predichos LOOCV**

![Figura 24](Figuras/07_01_residuals_vs_predicted.png)

Y_detected muestra estructura residual clara — los errores no son aleatorios respecto al nivel de predicción. Y_intensity y D_biomarkers presentan residuos más homogéneamente dispersos alrededor de cero.

**Figura 25 — Q-Q plots de residuos**

![Figura 25](Figuras/07_02_qq_plots.png)

Y_detected muestra desviaciones en ambas colas, especialmente la derecha (errores positivos grandes). Y_intensity y D_biomarkers se ajustan razonablemente bien a la diagonal.

---

## Interpretabilidad: SHAP, permutation importance y PDP

**Figura 26 — SHAP summary**

![Figura 26](Figuras/08_01_shap_summary.png)

Gráficos de barras con el $|\text{SHAP}|$ promedio por factor y por target. $x_1$ supera con claridad a $x_2$ y $x_3$ en magnitud de impacto para los tres targets. La jerarquía $x_1 \gg x_3 > x_2$ se mantiene constante.

**Figura 27 — Permutation importance**

![Figura 27](Figuras/08_02_permutation_importance.png)

La misma jerarquía $x_1 > x_3 > x_2$ se reproduce con un método diferente (pérdida de $R^2$ al permutar cada variable, n_repeats=10). La coincidencia con SHAP valida que no hay dependencia metodológica en las conclusiones de interpretabilidad.

**Figura 28 — Partial Dependence Plots (PDP)**

![Figura 28](Figuras/08_03_partial_dependence.png)

$x_1$ presenta una curva claramente creciente y pronunciada: alargar el tiempo de extracción incrementa de forma consistente la respuesta prevista. $x_2$ muestra un efecto menor y no monótono. $x_3$ presenta un efecto moderado o ligeramente decreciente en algunos targets, coherente con el efecto principal negativo observado en el DOE.

---

## Optimización final: Grid Search y decodificación

Se generó una rejilla densa de $15^3 = 3{,}375$ puntos sobre el espacio codificado $[-1, +1]^3$ y se predijo el valor de cada respuesta con RSM, RandomForest y XGBoost. Los óptimos se decodificaron a unidades reales.

**Tabla 14. Óptimos decodificados a unidades reales para cada modelo y target**

| Target | Modelo | t_ext (min) | t_des (min) | t_inc (min) |
|--------|--------|------------|------------|------------|
| Y_detected | RSM | 85.9 | 3.70 | 1.7 |
| Y_detected | RandomForest | 60.3 | 1.51 | 7.9 |
| Y_detected | XGBoost | 120.0 | 2.00 | 11.7 |
| Y_intensity | RSM | 102.9 | 3.70 | 16.7 |
| Y_intensity | RandomForest | 60.3 | 1.51 | 7.9 |
| Y_intensity | XGBoost | 120.0 | 2.00 | 11.7 |
| D_biomarkers | RSM | 94.4 | 0.30 | 2.9 |
| D_biomarkers | RandomForest | 60.3 | 1.51 | 7.9 |
| D_biomarkers | XGBoost | 120.0 | 2.00 | 11.7 |
| **Y_detected — PROMEDIO** | Consenso | **88.7** | **2.40** | **7.1** |
| **Y_intensity — PROMEDIO** | Consenso | **94.4** | **2.40** | **12.1** |
| **D_biomarkers — PROMEDIO** | Consenso | **91.6** | **1.27** | **7.5** |

La señal más consistente entre los tres modelos es cualitativa: $x_1$ siempre tiende a valores altos ($t_\text{ext} \geq 85$ min en todos los promedios). La discordancia más marcada se da en $t_\text{incubación}$: RSM propone extremos (1.7 o 19.2 min) mientras que los promedios caen en rangos intermedios (7–12 min). Esto sugiere que la curvatura de $x_3$ no está bien capturada con N = 17.

**Figura 29 — Óptimos predichos en espacio codificado**

![Figura 29](Figuras/09_01_optimal_space.png)

Tres paneles con los planos $x_1$ vs $x_2$, $x_1$ vs $x_3$ y $x_2$ vs $x_3$. Los puntos de distintos modelos están claramente separados en los planos que involucran $x_2$ y $x_3$. En el plano $x_1$ vs $x_2$, todos los puntos se sitúan en la zona de $x_1$ alto, confirmando el consenso en ese factor. La dispersión en $x_2$ y $x_3$ es la que impide definir un único punto óptimo sin validación experimental.

---

## Conclusiones

**C1 — Hallazgo científico principal**

El tiempo de extracción ($x_1$) es el factor dominante del sistema con independencia del método de análisis utilizado. Este resultado aparece en las correlaciones bivariables ($r = 0.825$ con Y_intensity), en los efectos principales del DOE, en los coeficientes RSM (término más importante en los tres modelos), en la feature importance de RandomForest y XGBoost (~70% del total), en los valores SHAP ($x_1$ primero en todos los targets) y en las curvas PDP (tendencia creciente y pronunciada). La convergencia de ocho métodos distintos le da una solidez excepcional a este hallazgo.

**C2 — Dos funciones objetivo, dos respuestas distintas**

D_global y D_biomarkers no conducen al mismo óptimo observado. D_global señala al experimento 10 (punto axial) como el mejor en el escenario base. D_biomarkers identifica al experimento 5 (factorial) como el mejor observado (D = 0.839), y los puntos centrales como los más estables en promedio (media D = 0.741, std = 0.021). La elección de la función objetivo debe estar guiada por el objetivo biológico: si lo relevante es la reproducibilidad y la cobertura del panel biomarcador, D_biomarkers y los puntos centrales son la referencia más apropiada.

**C3 — Limitaciones del RSM con N = 17**

Los modelos RSM ajustan bien en entrenamiento ($R^2 = 0.81$–$0.92$) pero generalizan mal en LOOCV para Y_detected y D_biomarkers ($R^2_\text{LOOCV}$ negativos). Esto no es un error de implementación sino una consecuencia estructural: un modelo de segundo orden completo con 10 términos aplicado a 17 puntos deja muy poco grado de libertad para validación cruzada. RSM es útil como herramienta de interpretación (curvatura, dirección, importancia de términos) pero no debe usarse como motor predictivo único para la decisión del óptimo.

**C4 — Machine Learning aporta capacidad predictiva adicional**

GaussianProcess es el mejor modelo individual para Y_intensity ($R^2_\text{CV} = 0.70$). RandomForest es el modelo más equilibrado entre targets y el ganador del ranking global. Los modelos de ML confirman la dominancia de $x_1$ y mejoran la generalización respecto a RSM, aunque todos están limitados por el tamaño muestral. La conclusión práctica no es que ML sea "mejor que DOE", sino que ambos enfoques son complementarios: el DOE aporta estructura y eficiencia experimental, y ML aporta flexibilidad predictiva.

**C5 — Los óptimos son regiones, no puntos únicos**

La concordancia baja entre RSM, RandomForest y XGBoost en la fase de optimización es una señal honesta de incertidumbre en la localización del óptimo, y no un defecto del análisis. La región de búsqueda recomendada para una campaña confirmatoria es:

| Factor | Recomendación |
|--------|--------------|
| $t_\text{extracción}$ ($x_1$) | **85 – 120 min** — priorizar el extremo superior del rango |
| $t_\text{desorción}$ ($x_2$) | **1.0 – 2.5 min** — desorción baja a moderada |
| $t_\text{incubación}$ ($x_3$) | **5 – 15 min** — rango intermedio; ajustar según target prioritario |

Validar experimentalmente al menos 3 condiciones en esta región antes de fijar el protocolo definitivo.

**C6 — Aspectos a revisar antes de la siguiente iteración**

- Los paneles de las figuras SHAP, permutation importance y PDP son visualmente muy similares entre targets. Verificar que cada gráfico refleja el modelo entrenado sobre el target correcto.
- RSM_3 extrapola fuera de $[0, 1]$ para D_biomarkers ($Y^* = 1.36$). Restringir el dominio de búsqueda al espacio experimental cubierto por el DOE en análisis futuros.
- N = 17 es un límite real. Si el objetivo es un modelo predictivo fiable, la recomendación del análisis de curvas de aprendizaje es añadir experimentos confirmatorios, no ajustar hiperparámetros.

---

## Estructura del repositorio

```
.
|-- DOE_ML_Optimization.ipynb
|-- Excel/
|   |-- Diseño Experimental.xlsx
|   |-- Optimización Diseño Experimental ML.xlsx
|   |-- Lista Metabolitos de Interés.xlsx
|-- Figuras/
|   |-- 01_validacion_preliminar.png
|   |-- 02_design_matrix_coded.png
|   |-- 03_desirability_escenarios.png
|   |-- 04_01_response_distributions.png
|   |-- 04_02_factors_vs_response.png
|   |-- 04_02b_3d_design_space.png
|   |-- 04_03_correlation_matrix.png
|   |-- 04_05_boxplots_classification.png
|   |-- 04_06_main_effects.png
|   |-- 04_07_interaction_plots.png
|   |-- 05_02_rsm_parity_plots.png
|   |-- 05_03_rsm_surface_contour.png
|   |-- 05_04_rsm_residuals_diagnostic.png
|   |-- 05_06_rsm_importance_ranking.png
|   |-- 06_01_desirability_distributions.png
|   |-- 06_02_desirability_global.png
|   |-- 06_03_desirability_by_classification.png
|   |-- 06_04_desirability_experiments.png
|   |-- 06_05_correlation_matrix.png
|   |-- 06_04_ml_parity_plots.png
|   |-- 06_05_feature_importance.png
|   |-- 06_06_learning_curves.png
|   |-- 06_07_predictions_correlation.png
|   |-- 07_01_residuals_vs_predicted.png
|   |-- 07_02_qq_plots.png
|   |-- 08_01_shap_summary.png
|   |-- 08_02_permutation_importance.png
|   |-- 08_03_partial_dependence.png
|   |-- 09_01_optimal_space.png
```

---

## Requisitos

```bash
pip install pandas numpy matplotlib seaborn scikit-learn xgboost shap scipy statsmodels openpyxl
```

Python 3.9+. Versiones: pandas>=1.5, numpy>=1.23, matplotlib>=3.6, seaborn>=0.12, scikit-learn>=1.1, xgboost>=1.7, shap>=0.41, scipy>=1.9, statsmodels>=0.13, openpyxl>=3.0.

---

## Ejecución

1. Clonar o descargar el repositorio.
2. Asegurarse de que los tres archivos Excel se encuentran en la carpeta `Excel/`.
3. Ejecutar el notebook de forma secuencial (Sección 1 → 10):

```bash
jupyter notebook DOE_ML_Optimization.ipynb
```

---

## Referencias

- Derringer, G., Suich, R. (1980). Simultaneous optimization of several response variables. *Journal of Quality Technology*, 12(4), 214–219.
- Jové, M. et al. (2021). Multivariate exploration of the metabolic profiling of fecal samples. *Analytical and Bioanalytical Chemistry*.
- Box, G.E.P., Hunter, J.S., Hunter, W.G. (2005). *Statistics for Experimenters*. Wiley.
- Lundberg, S.M., Lee, S.I. (2017). A unified approach to interpreting model predictions. *NeurIPS 2017*.
- Breiman, L. (2001). Random forests. *Machine Learning*, 45(1), 5–32.
- Chen, T., Guestrin, C. (2016). XGBoost: A scalable tree boosting system. *KDD 2016*.

---

*Análisis desarrollado como parte de un proyecto de optimización metabolómica mediante diseño estadístico de experimentos y modelos de aprendizaje automático supervisado.*
