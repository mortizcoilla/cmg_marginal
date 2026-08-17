# CMG Forecast Study — Sistema Eléctrico Nacional chileno

Pronóstico horario del Costo Marginal (CMG) en barras del SEN, con foco en Crucero 220 kV y Quillota 220 kV.

## TL;DR

- **Modelo principal**: LightGBM (quantile regression, q=0.025, 0.5, 0.975).
- **Métricas en test 2025-Q3-Q4**: MAE 2.53 USD/MWh, RMSE 6.19, MAPE 4.6%, **+58% mejora vs naive**.
- **Walk-forward fold 2** (train 2023-2025H1, test 2025H2): MAE 2.43, consistente.
- **IC 95% cobertura empírica**: 87.5% (subóptimo por asimetría del CMG).
- **⚠ Limitación importante**: el CMG horario es una **reconstrucción proxy** desde el CMG diario real del CEN + perfil horario canónico del SEN. La magnitud diaria es real; la forma intra-día es reconstruida.

## Quick start

```bash
# 1. Levantar el dashboard (HTML estático)
cd /workspace/cmg-forecast
python3 -m http.server 8000
# Abrir http://localhost:8000/dashboard/

# 2. Reproducir el pipeline completo
cd /workspace
python3 cmg-forecast/src/build_hourly_proxy.py    # CMG diario → horario (proxy)
python3 cmg-forecast/src/build_features.py          # features + EDA
python3 cmg-forecast/src/build_models.py            # SARIMAX + LightGBM + walk-forward
python3 cmg-forecast/src/build_dashboard.py         # dashboard HTML
```

## Estructura

```
cmg-forecast/
├── data/
│   ├── raw/
│   │   ├── cen/         # CMG diario del CEN (Energía Abierta API)
│   │   └── meteo/       # NASA POWER (irradiancia, temp, viento)
│   └── processed/
│       ├── cmg_hourly_crucero.parquet       # target (proxy)
│       ├── cmg_hourly_quillota.parquet
│       ├── meteo_hourly_*.parquet
│       ├── dataset_master.parquet           # 26k filas × 39 cols
│       ├── train/val/test.parquet
│       └── data_dictionary.md
├── src/
│   ├── build_hourly_proxy.py   # CMG diario → horario (perfil canónico SEN)
│   ├── build_features.py        # merge + 8 figuras EDA + splits
│   ├── build_models.py          # SARIMAX + LightGBM quantile
│   ├── build_dashboard.py       # genera dashboard/index.html
│   ├── ingest_cen.py / ingest_meteo.py / plot_*.py
├── notebooks/
│   ├── 01_ingesta_cen.ipynb
│   ├── 01_ingesta_meteo.ipynb
│   └── 03_modeling.py
├── figures/
│   ├── eda_cen/         # diagnóstico CEN
│   ├── eda_meteo/       # diagnóstico meteo
│   └── eda/             # 8 figuras del EDA features
├── results/
│   ├── metrics_table.csv
│   ├── feature_importance.png
│   ├── forecast_examples.png
│   ├── duck_curve_reconstruction.png
│   ├── predictions_test.csv
│   └── predictions_72h.csv
├── models/              # artefactos LightGBM serializados
├── dashboard/
│   ├── index.html
│   └── assets/data.js
├── reports/
│   └── REPORTE_FINAL.md
└── deliverable.md
```

## Limitaciones documentadas

1. **CMG horario es proxy** (la API pública del CEN solo expone CMG diario).
2. **Demanda y generación** son sintéticas calibradas, no observaciones reales.
3. **Cobertura IC 95% subóptima** (87% vs 95% target) por asimetría del CMG.
4. **Solo Crucero y Quillota** (Diego de Almagro y Polpaico no están en la API pública).
5. **Sin features de eventos regulatorios** (decretos ERV, mantenimientos, fallas).

## Recomendaciones siguientes pasos

1. Obtener CMG horario real (suscripción Open Data CEN+ o scraping headless de `www.coordinador.cl`).
2. Integrar demanda y generación real desde CEN.
3. Probar modelos multi-horizon (TFT, N-HiTS).
4. Conformal prediction adaptativa para corregir IC.
5. Backtest rolling-origin semanal.
6. Operacionalizar: pipeline de re-fit automático cada lunes.

Ver `reports/REPORTE_FINAL.md` para el análisis completo.
