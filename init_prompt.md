# Proyecto: Producción solar estimada con regresión lineal

Este proyecto predice la energía solar diaria disponible —o, si dispones de mediciones, la generación fotovoltaica real— a partir de variables meteorológicas e irradiancia. Es visual, aplicable al sector de energías renovables y permite construir un modelo de regresión lineal interpretable con Python.

La idea central es responder: **¿cuánta energía puede producir una instalación fotovoltaica en una ubicación concreta, dadas la radiación, la temperatura, la nubosidad y la hora del día?** NASA POWER ofrece series diarias de irradiancia solar y meteorología, mientras que Open-Meteo permite obtener datos históricos horarios, incluida la radiación sobre un panel inclinado.

---

## 1. Objetivo del proyecto

Construir un modelo de regresión que estime la producción energética de un sistema fotovoltaico y analizar qué factores explican mejor su variabilidad.

Puedes plantearlo en dos niveles:

| Nivel | Variable objetivo | Datos necesarios | Resultado |
| :--- | :--- | :--- | :--- |
| **Básico** | Irradiancia global horizontal diaria (GHI) | NASA POWER u Open-Meteo | Estimar $\text{kWh/m}^2/\text{día}$ disponibles |
| **Avanzado** | Energía generada por un panel ($\text{kWh/día}$) | PVGIS o datos reales de una planta + clima | Estimar producción eléctrica real |

> **Recomendación:** Para un proyecto de regresión lineal, es conveniente empezar por el nivel básico y añadir después la estimación fotovoltaica. NASA POWER incluye el parámetro `ALLSKY_SFC_SW_DWN`, correspondiente a irradiancia de onda corta descendente bajo todas las condiciones de cielo; es una base sólida para modelar la energía solar incidente.

---

## 2. Preguntas de investigación

- ¿Es posible predecir la energía solar diaria de una ubicación a partir de temperatura, nubosidad, humedad, viento y variables astronómicas como la duración del día?
- **Extensión:** ¿Cuánto mejora la predicción al incorporar la irradiancia sobre un plano inclinado, en lugar de usar únicamente la radiación horizontal?

> Open-Meteo permite solicitar `global_tilted_irradiance` (GTI), que representa la radiación recibida por un panel con inclinación y orientación definidas; también ofrece GHI, radiación directa y difusa.

---

## 3. Estructura base del proyecto

```text
solar-regression/
│
├── data/
│   ├── raw/
│   │   ├── nasa_power_daily.csv
│   │   └── open_meteo_hourly.csv
│   └── processed/
│       └── solar_dataset.csv
│
├── notebooks/
│   ├── 01_exploracion_datos.ipynb
│   ├── 02_modelo_regresion.ipynb
│   └── 03_resultados_visualizacion.ipynb
│
├── src/
│   ├── config.py
│   ├── download_data.py
│   ├── preprocess.py
│   ├── features.py
│   ├── train_model.py
│   └── evaluate.py
│
├── reports/
│   ├── figures/
│   └── informe_final.md
│
├── requirements.txt
├── README.md
└── .gitignore
```

---

## 4. Tecnologías y librerías

- **Python 3.10 o superior**
- **pandas:** Manipulación de datos y series temporales.
- **numpy:** Cálculo numérico.
- **requests:** Llamadas a las APIs.
- **matplotlib** y **seaborn:** Visualizaciones.
- **scikit-learn:** Regresión lineal, Ridge, Lasso, validación y métricas.
- **statsmodels:** Interpretación estadística, intervalos de confianza y p-valores.
- **joblib:** Guardado del modelo entrenado.

### `requirements.txt` inicial

```text
pandas
numpy
requests
matplotlib
seaborn
scikit-learn
statsmodels
joblib
jupyterlab
```

---

## 5. Fuentes de datos

| Fuente | Datos principales | Frecuencia | Rol en el proyecto |
| :--- | :--- | :--- | :--- |
| **NASA POWER** | Irradiancia solar, temperatura, presión, viento | Diaria, desde 1981 | Variable objetivo o variables explicativas |
| **Open-Meteo Historical Weather** | GHI, radiación directa/difusa, GTI, temperatura, nubes, viento | Horaria o diaria | Variables meteorológicas y radiación sobre panel |
| **PVGIS** | Estimación de producción fotovoltaica, radiación, temperatura y viento | Horaria; serie 2013–2024 | Variable objetivo simulada o validación |
| **Instalación real** | Energía generada ($\text{kWh}$) | Horaria o diaria | Mejor opción si puedes acceder a datos de una planta |

- **NASA POWER:** Admite peticiones diarias por punto en JSON o CSV, con un máximo de 20 parámetros por solicitud para un punto concreto.
- **Open-Meteo:** Permite consultar su endpoint histórico `/v1/archive` indicando coordenadas, fechas y variables; para datos diarios requiere especificar zona horaria.

---

## 6. Variables del modelo

### Variable objetivo
Para la versión recomendada:

$$\text{energia\_solar\_diaria\_kwh\_m2} = \frac{\text{shortwave\_radiation\_sum}}{3.6}$$

*Open-Meteo devuelve `shortwave_radiation_sum` en $\text{MJ/m}^2$ por día; dividir entre $3.6$ convierte megajulios a $\text{kWh/m}^2$, ya que $1\text{ kWh} = 3.6\text{ MJ}$.*

### Variables predictoras iniciales

| Variable | Nombre sugerido | Justificación |
| :--- | :--- | :--- |
| Temperatura media | `temperature_mean` | La temperatura afecta al rendimiento de los paneles |
| Temperatura máxima | `temperature_max` | Puede capturar sobrecalentamiento |
| Nubosidad media | `cloud_cover_mean` | Reduce la radiación recibida |
| Humedad relativa media | `relative_humidity_mean` | Relacionada con condiciones atmosféricas |
| Velocidad media del viento | `wind_speed_mean` | Ayuda a refrigerar los módulos |
| Duración del día | `daylight_duration` | Determina las horas potenciales de generación |
| Duración de insolación | `sunshine_duration` | Mide horas con radiación directa suficiente |
| Radiación directa | `direct_radiation_sum` | Componente más valiosa para paneles orientados |
| Radiación difusa | `diffuse_radiation_sum` | Importante en días nublados |
| Radiación sobre plano inclinado | `global_tilted_irradiance_sum` | Más cercana a la energía recibida por el panel |
| Mes / estación | `month_sin`, `month_cos` | Captura estacionalidad sin tratar el mes como lineal |

---

## 7. Pautas metodológicas

1. **Delimita el caso de uso:**  
   Elige una única ubicación para la primera versión (por ejemplo Sevilla, Madrid o Zaragoza). Define coordenadas, orientación e inclinación del panel (ej. $30^\circ$ de inclinación y orientación sur). No mezcles ubicaciones con climas muy distintos en el primer modelo.
2. **Usa datos temporales correctamente:**  
   Separa cronológicamente para evitar fugas temporales:
   - **Entrenamiento:** primeros 70–80% del periodo (ej. 2015–2022).
   - **Validación:** siguiente 10–15% (ej. 2023).
   - **Test final:** último 10–15% (ej. 2024).
3. **Evita fuga de datos (*data leakage*):**  
   No uses como predictor una variable que sea prácticamente una reexpresión de la variable objetivo (ej. no usar `shortwave_radiation_sum` para predecir energía solar incidente). Si el objetivo es producción eléctrica real en $\text{kWh}$, la radiación sí actúa como predictor legítimo.
4. **Prioriza la interpretabilidad:**  
   La regresión lineal debe servir para explicar relaciones físicas:  
   *"Un aumento de $1\text{ kWh/m}^2$ en radiación sobre el plano inclinado se asocia con un aumento medio de $X\text{ kWh}$ en producción diaria, manteniendo temperatura y viento constantes."*
5. **Compara modelos sencillos:**  
   Construye la progresión:
   1. Regresión lineal simple (solo radiación).
   2. Regresión lineal múltiple (radiación, temperatura, nubes y viento).
   3. Regularización Ridge o Lasso (si hay colinealidad).
   4. Modelo no lineal de referencia (Random Forest o Gradient Boosting).

---

## 8. Paso a paso de implementación

### Paso 1: Define la ubicación y el sistema
Crea un archivo `src/config.py`:

```python
LATITUDE = 37.3891
LONGITUDE = -5.9845
CITY = "Sevilla"

START_DATE = "2015-01-01"
END_DATE = "2024-12-31"

PANEL_TILT = 30
PANEL_AZIMUTH = 0  # 0 = sur

TARGET = "solar_energy_kwh_m2"
```

### Paso 2: Descarga los datos meteorológicos (Open-Meteo)

```python
import requests
import pandas as pd

url = "https://archive-api.open-meteo.com/v1/archive"

params = {
    "latitude": 37.3891,
    "longitude": -5.9845,
    "start_date": "2015-01-01",
    "end_date": "2024-12-31",
    "hourly": ",".join([
        "temperature_2m",
        "relative_humidity_2m",
        "cloud_cover",
        "wind_speed_10m",
        "shortwave_radiation",
        "direct_radiation",
        "diffuse_radiation",
        "global_tilted_irradiance"
    ]),
    "daily": ",".join([
        "temperature_2m_max",
        "temperature_2m_min",
        "daylight_duration",
        "sunshine_duration",
        "shortwave_radiation_sum"
    ]),
    "timezone": "Europe/Madrid"
}

response = requests.get(url, params=params)
response.raise_for_status()
data = response.json()

hourly = pd.DataFrame(data["hourly"])
daily = pd.DataFrame(data["daily"])

hourly["time"] = pd.to_datetime(hourly["time"])
daily["time"] = pd.to_datetime(daily["time"])

hourly.to_csv("data/raw/open_meteo_hourly.csv", index=False)
daily.to_csv("data/raw/open_meteo_daily.csv", index=False)
```

### Paso 3: Descarga datos de NASA POWER

```python
import requests
import pandas as pd

url = "https://power.larc.nasa.gov/api/temporal/daily/point"

params = {
    "parameters": "ALLSKY_SFC_SW_DWN,T2M,WS10M,PS",
    "community": "SB",
    "longitude": -5.9845,
    "latitude": 37.3891,
    "start": "20150101",
    "end": "20241231",
    "format": "CSV"
}

response = requests.get(url, params=params)
response.raise_for_status()

with open("data/raw/nasa_power_daily.csv", "wb") as f:
    f.write(response.content)
```

### Paso 4: Limpia y unifica los datos (`src/preprocess.py`)

```python
import pandas as pd

hourly = pd.read_csv("data/raw/open_meteo_hourly.csv", parse_dates=["time"])
daily = pd.read_csv("data/raw/open_meteo_daily.csv", parse_dates=["time"])

hourly["date"] = hourly["time"].dt.date

daily_features = hourly.groupby("date").agg(
    temperature_mean=("temperature_2m", "mean"),
    relative_humidity_mean=("relative_humidity_2m", "mean"),
    cloud_cover_mean=("cloud_cover", "mean"),
    wind_speed_mean=("wind_speed_10m", "mean"),
    direct_radiation_sum=("direct_radiation", "sum"),
    diffuse_radiation_sum=("diffuse_radiation", "sum"),
    tilted_irradiance_sum=("global_tilted_irradiance", "sum")
).reset_index()

daily_features["date"] = pd.to_datetime(daily_features["date"])
daily = daily.rename(columns={"time": "date"})

df = daily.merge(daily_features, on="date", how="inner")

# Conversión de MJ/m² a kWh/m²
df["solar_energy_kwh_m2"] = df["shortwave_radiation_sum"] / 3.6

df.to_csv("data/processed/solar_dataset.csv", index=False)
```

### Paso 5: Crea variables adicionales (`features.py`)

```python
import numpy as np
import pandas as pd

df = pd.read_csv("data/processed/solar_dataset.csv", parse_dates=["date"])

# Estacionalidad cíclica
df["month"] = df["date"].dt.month
df["month_sin"] = np.sin(2 * np.pi * df["month"] / 12)
df["month_cos"] = np.cos(2 * np.pi * df["month"] / 12)

df["day_of_year"] = df["date"].dt.dayofyear
df["day_of_year_sin"] = np.sin(2 * np.pi * df["day_of_year"] / 365)
df["day_of_year_cos"] = np.cos(2 * np.pi * df["day_of_year"] / 365)

# Lags temporales
df["tilted_irradiance_lag_1"] = df["tilted_irradiance_sum"].shift(1)
df["temperature_lag_1"] = df["temperature_mean"].shift(1)

df = df.dropna()

df.to_csv("data/processed/solar_dataset_features.csv", index=False)
```

### Paso 6: Explora los datos (`01_exploracion_datos.ipynb`)

Análisis prioritarios:
- Serie temporal y medias estacionales.
- Diagramas de dispersión: radiación vs. energía, nubosidad vs. energía.
- Matriz de correlación y boxplots mensuales.
- Comparativa por tipo de cielo y detección de anomalías.

```python
import matplotlib.pyplot as plt
import seaborn as sns

fig, ax = plt.subplots(figsize=(12, 5))
sns.lineplot(data=df, x="date", y="solar_energy_kwh_m2", ax=ax)
ax.set_title("Energía solar diaria estimada")
ax.set_ylabel("kWh/m²")
plt.tight_layout()
plt.savefig("reports/figures/serie_temporal.png")
```

### Paso 7: Entrena la regresión lineal

```python
import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_absolute_error, r2_score

df = pd.read_csv("data/processed/solar_dataset_features.csv", parse_dates=["date"])

features = [
    "tilted_irradiance_sum",
    "temperature_mean",
    "cloud_cover_mean",
    "wind_speed_mean",
    "daylight_duration",
    "sunshine_duration",
    "month_sin",
    "month_cos"
]

target = "solar_energy_kwh_m2"

train = df[df["date"] < "2023-01-01"]
test = df[df["date"] >= "2023-01-01"]

X_train, y_train = train[features], train[target]
X_test, y_test = test[features], test[target]

model = LinearRegression()
model.fit(X_train, y_train)

predictions = model.predict(X_test)

print("R2:", r2_score(y_test, predictions))
print("MAE:", mean_absolute_error(y_test, predictions))
```

### Paso 8: Valida e interpreta

| Métrica | Interpretación |
| :--- | :--- |
| **MAE** | Error medio absoluto, en las mismas unidades que la variable objetivo |
| **RMSE** | Raíz del error cuadrático medio; penaliza con mayor severidad desviaciones grandes |
| **$R^2$** | Proporción de la varianza explicada por el modelo |
| **MAPE** | Error porcentual medio |

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score
import statsmodels.api as sm
import numpy as np

mae = mean_absolute_error(y_test, predictions)
rmse = np.sqrt(mean_squared_error(y_test, predictions))
r2 = r2_score(y_test, predictions)

print(f"MAE: {mae:.3f}")
print(f"RMSE: {rmse:.3f}")
print(f"R2: {r2:.3f}")

# Diagnóstico inferencial
X_train_sm = sm.add_constant(X_train)
ols = sm.OLS(y_train, X_train_sm).fit()
print(ols.summary())
```

---

## 9. Extensión a producción fotovoltaica

Para modelar la energía eléctrica generada ($\text{kWh}$):

$$\text{energia\_generada\_kwh} \sim \beta_0 + \beta_1 \cdot \text{radiacion\_inclinada} + \beta_2 \cdot \text{temperatura} + \beta_3 \cdot \text{viento} + \beta_4 \cdot \text{nubosidad}$$

| Opción | Variable objetivo | Cómo proceder |
| :--- | :--- | :--- |
| **Simulada** | Generación estimada por PVGIS | Descargar series horarias de la API de PVGIS y agregarlas a diario |
| **Real** | Generación de una instalación real | Utilizar registros de inversor, vatímetro o plataforma de monitorización |

---

## 10. Informe final y entregables

### Estructura del informe (`reports/informe_final.md`)
1. **Contexto:** Importancia de modelar el recurso solar.
2. **Objetivo:** Formulación del problema de estimación.
3. **Ubicación y periodo:** Coordenadas, geometría del panel y rango temporal.
4. **Fuentes:** NASA POWER, Open-Meteo y/o PVGIS.
5. **Limpieza de datos:** Tratamiento de unidades, agregaciones y nulos.
6. **Análisis exploratorio:** Correlaciones y visualizaciones clave.
7. **Modelo:** Definición de predictores y split temporal cronológico.
8. **Resultados:** MAE, RMSE, $R^2$ y parámetros del ajuste.
9. **Interpretación:** Magnitud y dirección física de los coeficientes.
10. **Limitaciones:** Ausencia de suciedad, sombras cercanas, pérdidas de cableado/inversor.
11. **Mejoras futuras:** Modelado horario, modelos no lineales, múltiples ubicaciones.

### Entregables finales
- Dataset procesado: `data/processed/solar_dataset.csv`.
- Notebooks reproducibles (`01_`, `02_`, `03_`).
- Modelo entrenado serializado con `joblib`.
- Gráficos clave: serie temporal, matriz de correlación, predicción vs. real y análisis de residuos.
- `README.md` con instrucciones de ejecución.

---

## 11. Posibles extensiones
- Comparar distintas zonas climáticas de España.
- Modelar en resolución horaria en lugar de diaria.
- Evaluar seguimiento a un eje frente a panel fijo.
- Estimar ahorro económico y emisiones de $\text{CO}_2$ evitadas.
- Desplegar una aplicación interactiva en **Streamlit**.