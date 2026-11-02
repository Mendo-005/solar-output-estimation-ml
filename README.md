# solar-output-estimation-ml
# Estimación de energía solar diaria con regresión lineal

Proyecto reproducible para Sevilla que estima la energía solar diaria incidente en **kWh/m²/día** usando variables meteorológicas y estacionalidad. Toda la implementación está en un único notebook, [`solar_regression.ipynb`](./solar_regression.ipynb); no hay módulos Python separados. El notebook crea automáticamente las carpetas `data/raw/`, `data/processed/` y `reports/figures/` al ejecutarse.

## Requisitos y ejecución

- Python 3.10 o superior
- Conexión a internet para descargar el histórico de [Open-Meteo Archive](https://open-meteo.com/en/docs/historical-weather-api)
- Dependencias listadas en [`requirements.txt`](./requirements.txt)

Desde la raíz del repositorio, instala las dependencias y abre el notebook:

```powershell
python -m pip install -r requirements.txt
jupyter lab solar_regression.ipynb
```

Ejecuta las celdas en orden. La descarga guarda las series horaria y diaria en `data/raw/`; el preprocesamiento agrega las variables meteorológicas horarias por fecha local y guarda `data/processed/solar_dataset.csv`.

## Metodología y prevención de fuga

- Ubicación: Sevilla (37.3891, -5.9845), zona horaria `Europe/Madrid`, periodo 2015–2024.
- La variable objetivo diaria de Open-Meteo, `shortwave_radiation_sum`, está expresada en MJ/m²/día. Se convierte a kWh/m²/día dividiendo entre **3.6**.
- Temperatura, humedad, nubosidad y viento horarios se agregan mediante medias diarias; no se suman valores horarios de irradiancia expresados como potencia.
- Los predictores son exclusivamente meteorología y variables cíclicas del calendario. Se excluyen radiación e insolación porque el objetivo se deriva de la propia irradiancia.
- Partición cronológica: entrenamiento hasta 2022; validación en 2023; test final en 2024. El modelo final se ajusta con entrenamiento y validación y el test no participa en el ajuste.
- Se comparan regresión lineal y Random Forest mediante `TimeSeriesSplit` de cinco pliegues expansivos dentro del entrenamiento. La selección final se realiza por RMSE en la validación de 2023; el test de 2024 se reserva para la evaluación final.
- `statsmodels` informa coeficientes, p-valores e intervalos de confianza del 95% con errores HAC, usando solo los datos anteriores al test.

El análisis OLS usa errores robustos HAC (Newey–West, 7 retardos) para tener en cuenta heterocedasticidad y autocorrelación de corto plazo. Los p-valores son asociativos, no causales.

Con los 3.287 días de entrenamiento + validación, los siete predictores del OLS resultaron estadísticamente significativos (`p < 0.001`, errores HAC). Humedad, nubosidad y viento presentan coeficientes negativos; la temperatura media y máxima tienen signos opuestos al condicionarse mutuamente y por estacionalidad, por lo que no deben interpretarse como efectos causales independientes. La tabla completa de coeficientes e intervalos del 95% está en el notebook.

### Métricas observadas

Resultados de la ejecución con Open-Meteo para 2015–2024. La validación cruzada usa cinco cortes temporales expansivos solo en train (2015–2022); la validación compara modelos en 2023 y la evaluación final del modelo seleccionado se hace en 2024:

| Modelo / partición | R² | MAE (kWh/m²/día) | RMSE (kWh/m²/día) |
|---|---:|---:|---:|
| Regresión lineal — CV train (media) | 0.9333 | 0.4125 | 0.5428 |
| Random Forest — CV train (media) | 0.9391 | 0.3484 | 0.5207 |
| Regresión lineal — validación 2023 | 0.9428 | 0.3772 | 0.5013 |
| Random Forest — validación 2023 | 0.9583 | 0.2871 | 0.4278 |
| Random Forest — test final 2024 | 0.9468 | 0.3507 | 0.5027 |

En esta ejecución se seleccionó Random Forest por su menor RMSE en la validación de 2023. El notebook recalcula las métricas al ejecutarse; pueden variar si el proveedor revisa el histórico.

## Salidas

- `data/raw/open_meteo_hourly.csv`, `data/raw/open_meteo_daily.csv`
- `data/processed/solar_dataset.csv`, `data/processed/test_predictions.csv`
- `data/processed/solar_best_model.joblib`
- `reports/figures/serie_temporal.png`, `correlaciones_entrenamiento.png`, `prediccion_test_2024.png`

Este proyecto estima irradiación incidente, no generación eléctrica real. Para modelar producción fotovoltaica hacen falta datos de una instalación o una simulación de generación (por ejemplo, PVGIS), además de considerar potencia, orientación, sombras, temperatura del módulo y pérdidas del sistema.