[README.md](https://github.com/user-attachments/files/29723451/README.md)
# Sistema de Scoring de Riesgo Crediticio

Proyecto de portafolio de Data Science — predicción de default (impago) en solicitudes
de crédito, usando SQL, Python/scikit-learn y Power BI.

## Resumen

Construí un modelo de clasificación (Random Forest, ROC-AUC 0.80) que predice la
probabilidad de que un solicitante de crédito haga default. Pero el hallazgo con
más impacto de negocio no fue el modelo en sí, sino el **umbral de decisión**: al
optimizar el punto de corte según el costo real de cada tipo de error (perder un
préstamo completo vs. perder solo el margen de interés de un buen cliente
rechazado), reduje el costo estimado de la cartera en **53.4%** — de ~104,000 DM a
~48,500 DM — sin cambiar el modelo, solo ajustando dónde se traza la línea entre
"aprobar" y "rechazar". El proyecto cubre el pipeline completo: SQL para
exploración, Python/scikit-learn para el modelo, y un dashboard interactivo en
Power BI para comunicar el resultado a una audiencia de negocio no técnica.

## Problema de negocio

Un banco recibe solicitudes de crédito y necesita decidir a quién aprobar. Aprobar a
un cliente que hará default cuesta dinero (pérdida del préstamo); rechazar a un
buen cliente también cuesta dinero (oportunidad perdida). Este proyecto construye
un modelo de clasificación que estima la probabilidad de default y permite ajustar
el umbral de decisión según el costo relativo de cada tipo de error.

## Dataset

**German Credit Data** — 1,000 solicitudes de crédito con 20 variables (edad, monto,
duración, historial crediticio, propósito, ahorros, etc.) y variable target `default`
(1 = impago, 0 = pagó bien). Tasa de default real: 30%.

Fuente: [selva86/datasets](https://github.com/selva86/datasets) (versión con nombres
de columnas legibles del dataset original de UCI Machine Learning Repository).

## Estructura del proyecto

```
credit_risk_project/
├── data/
│   ├── german_credit.csv           # Dataset original
│   ├── credit_risk.db              # SQLite: tablas credit_applications, credit_features
│   ├── credit_features.csv         # Dataset con feature engineering (Día 2)
│   ├── model_*.pkl                 # Modelos entrenados (Logistic Regression, RF, XGBoost)
│   ├── scaler.pkl                  # Scaler usado para Logistic Regression
│   ├── X_test.csv / y_test.csv     # Set de prueba (Día 4)
│   ├── credit_risk_powerbi.csv     # Dataset final para el dashboard (Día 6)
│   └── kpis_resumen.csv            # KPIs resumidos para tarjetas del dashboard
├── sql/
│   └── 01_exploracion.sql          # Queries de exploración y preguntas de negocio
├── notebooks/
│   ├── 01_setup_exploracion.ipynb            # Día 1: setup + EDA con SQL
│   ├── 02_limpieza_feature_engineering.ipynb # Día 2: limpieza + feature engineering
│   ├── 03_eda_visual.ipynb                   # Día 3: EDA visual completo
│   ├── 04_modelado.ipynb                     # Día 4: entrenamiento de 3 modelos
│   ├── 05_evaluacion_umbral.ipynb            # Día 5: evaluación + ajuste de umbral
│   └── 06_preparacion_powerbi.ipynb          # Día 6: preparación de datos para Power BI
├── dashboard_riesgo_crediticio.pbix          # Dashboard Power BI (2 páginas)
└── README.md
```

## Plan de 7 días

| Día | Actividad | Estado |
|---|---|---|
| 1 | Setup + exploración inicial con SQL, preguntas de negocio | ✅ Completado |
| 2 | Limpieza y feature engineering (encoding, variables derivadas) | ✅ Completado |
| 3 | EDA visual completo: distribuciones, correlaciones | ✅ Completado |
| 4 | Modelado: Logistic Regression (baseline) + Random Forest/XGBoost | ✅ Completado |
| 5 | Evaluación: ROC-AUC, matriz de confusión, ajuste de umbral | ✅ Completado |
| 6 | Dashboard en Power BI: KPIs de cartera, segmentación de riesgo | ✅ Completado |
| 7 | Documentación final y storytelling para portafolio | ✅ Completado |

## Día 1 — Exploración con SQL

- Se cargó el dataset en una base SQLite (`credit_risk.db`) para trabajar con SQL real.
- Se definieron y respondieron 4 preguntas de negocio: ¿qué tan riesgosa es la cartera?,
  ¿el monto/duración influyen en el riesgo?, ¿qué propósito de crédito es más riesgoso?,
  ¿el historial crediticio predice el riesgo actual?
- Se verificó calidad de datos (sin nulos en variables clave).

## Día 2 — Limpieza y feature engineering

- Se crearon 4 variables derivadas con justificación de negocio: `amount_per_month`,
  `age_group`, `has_no_savings`, y scores ordinales para `savings`, `employment_duration`
  y `status` (saldo de cuenta corriente).
- Codificación diferenciada: mapeo ordinal para variables con orden natural, one-hot
  encoding para variables nominales.
- Primer vistazo a correlaciones: `status_score` y `duration` como variables más
  relacionadas con el default.

## Día 3 — EDA visual

- `amount` y `duration` muestran sesgo a la derecha — se probó una transformación log en el Día 4.
- Outliers (7.2% en `amount`, 7.0% en `duration`) identificados pero **no eliminados**:
  son casos reales de negocio, no errores de captura.
- Se evitó el error común de mezclar variables dummy de one-hot encoding en un solo
  heatmap grande (generan correlaciones negativas artificiales entre sí por ser
  mutuamente excluyentes). Se separó en: gráfico de barras para correlación con el
  target + heatmap pequeño solo para variables numéricas/ordinales.
- Investigación de un hallazgo contraintuitivo: el grupo "sin cuenta de ahorros
  conocida" tiene tasa de default más baja que grupos con algo de ahorro — hipótesis:
  la categoría agrupa perfiles heterogéneos, no necesariamente clientes sin recursos.

## Día 4 — Modelado

- Se agregó `log_amount` como feature adicional.
- Split 80/20 estratificado; balanceo de clases (`class_weight='balanced'` /
  `scale_pos_weight`) por el desbalance real de 70/30.
- Comparación de 3 modelos (ROC-AUC en test):

  | Modelo | ROC-AUC |
  |---|---|
  | **Random Forest** | **0.804** |
  | XGBoost | 0.770 |
  | Logistic Regression | 0.768 |

> **Nota de corrección real (no cosmética):** durante este día se detectó que el
> mapeo de `status_score` del Día 2 tenía una categoría mal escrita, generando 274
> valores nulos que Logistic Regression rechazaba. Se corrigió en el notebook del
> Día 2 y se regeneraron los Días 2-4 para mantener consistencia. Es el tipo de bug
> silencioso que vale la pena mencionar en una entrevista: se detectó por un error
> de entrenamiento, no por revisión manual, y se corrigió de raíz en vez de parchearlo.

## Día 5 — Evaluación y ajuste de umbral (el hallazgo principal)

- Curvas ROC confirmaron a Random Forest como el mejor modelo.
- Se definió el costo de negocio de cada error (supuesto explícito y documentado
  como limitación): un default no detectado cuesta el monto completo del préstamo;
  rechazar un buen cliente cuesta solo el margen de interés (15% del monto).
- **Umbral por defecto (0.5):** costo total estimado ≈ 104,071 DM.
- **Umbral optimizado por costo (0.35):** costo total estimado ≈ 48,485 DM.
- **Ahorro estimado: 53.4%**, sin reentrenar el modelo — solo moviendo el punto de
  corte de decisión según el costo real de negocio.

## Día 6 — Dashboard en Power BI

Dashboard interactivo de 2 páginas (`dashboard_riesgo_crediticio.pbix`):

**Página 1 — Panorama de la cartera:** KPIs dinámicos (total de solicitudes, tasa de
default real, ahorro estimado por el umbral optimizado), distribución de la cartera,
y tasa de default por propósito de crédito — filtrado para excluir categorías con
menos de 15 observaciones por baja confiabilidad estadística, con un mensaje
automático que avisa cuando un segmento filtrado no tiene datos suficientes.
Incluye 3 filtros interactivos (rango de edad, tipo de vivienda, propósito).

**Página 2 — Desempeño del modelo:** matriz de confusión en lenguaje de negocio (ej.
"buen cliente rechazado" en vez de "falso positivo"), distribución de solicitudes
por nivel de riesgo, y un gráfico de dispersión que cruza probabilidad de default
con monto del préstamo — coloreado por tipo de resultado (verde = acierto,
naranja/rojo = error) para identificar visualmente dónde se concentran los errores
más costosos.

## Limitaciones y consideraciones honestas

- **El costo de cada error es un supuesto, no un dato real del banco** (100% del
  monto como pérdida por default, 15% de margen perdido por rechazo). En un
  proyecto real, esto se calibraría con el equipo de riesgo/finanzas.
- **El dataset tiene solo 1,000 solicitudes**, y el set de prueba usado para el
  dashboard son 200 — algunas categorías (como ciertos propósitos de crédito)
  tienen muestras demasiado pequeñas para conclusiones robustas; el dashboard filtra
  explícitamente estos casos en vez de ocultarlos silenciosamente.
- **German Credit Data es un dataset muy usado en la comunidad de ML** — el valor de
  este proyecto no está en el dataset, sino en las decisiones de modelado
  (ingeniería de variables, elección y justificación del umbral de negocio,
  comunicación a audiencia no técnica).

## Cómo correr este proyecto

```bash
pip install pandas jupyter matplotlib seaborn scikit-learn xgboost joblib --break-system-packages
jupyter notebook notebooks/01_setup_exploracion.ipynb
```

Los notebooks están numerados y deben ejecutarse en orden (cada uno depende de
archivos generados por el anterior). Para explorar la base de datos con DBeaver:
conectar a `data/credit_risk.db` como base de datos SQLite. Para el dashboard,
abrir `dashboard_riesgo_crediticio.pbix` con Power BI Desktop.
