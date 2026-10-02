# Auditoría del proyecto CMG Forecast Study

**Fecha:** 2026-09-30 · **Alcance:** repositorio `CMG marginal` (commit `5dbbfda`) + metodología documentada en `README.md` y `REPORTE_FINAL.md`.

---

## 1. Hallazgo principal: el repositorio no contiene código

El README describe una estructura completa (`src/*.py`, `data/`, `notebooks/`, `figures/`, `results/`, `models/`, `dashboard/`), pero **el repositorio solo contiene 2 archivos Markdown** (`README.md`, `REPORTE_FINAL.md`). El único commit (`first commit`) confirma que nunca se versionó código aquí.

Consecuencias:

- Los comandos del *Quick start* (`python3 cmg-forecast/src/build_models.py`, etc.) **no pueden ejecutarse** desde este repo: fallan al no existir los archivos, además de referenciar rutas absolutas de otro entorno (`/workspace/cmg-forecast`).
- Las métricas y figuras del reporte **no son verificables ni reproducibles** desde este repositorio.
- El dashboard `dashboard/index.html` mencionado tampoco está versionado.

**Acción:** versionar `src/`, `notebooks/`, `results/*.csv` y, idealmente, los datos procesados (o usar DVC/git-lfs). Reemplazar rutas absolutas por relativas al root del proyecto.

## 2. Riesgo metodológico mayor: fuga estructural por target proxy

El CMG horario se construye como `CMG_diario × perfil_canónico[h] × (1 + ε)`, `ε ~ 3%`. Eso implica:

- El "pronóstico horario" se reduce, en gran medida, a **aprender el perfil determinista** (función de la hora) + persistir el nivel diario (que llega vía `cmg_lag_1h`).
- El piso teórico de error (un "oracle" que conociera el perfil y persistiera el nivel diario) sería ≈ la magnitud del ruido inyectado (~3% ≈ 1.7 USD/MWh de MAE). El MAE reportado de **2.53 está cerca de ese piso**: buena parte del desempeño proviene de la construcción del target, no del aprendizaje de dinámicas reales de mercado.
- La **mejora de +58.3% vs naive está inflada por construcción**: el naive no usa la hora del día; el modelo sí, y la hora determina el perfil.

**Acción:** reportar además una métrica honesta sobre el CMG **diario** real (que sí lo es): p. ej. MAE de predecir el nivel diario, o comparar contra un "oracle de perfil" para acotar cuánto aporta el modelo sobre la construcción.

## 3. Anomalía: SARIMAX 4× peor que naive

SARIMAX obtiene MAE 24.19 / MAPE 63.6% frente a 6.07 / 12.6% del naive. Un SARIMAX correctamente especificado rara vez es 4× peor que persistencia; esto sugiere un **problema de implementación** (escala mal alineada, exógenas con fuga o desalineadas, orden/diferenciación inadecuado, forecast mal invertido) más que un modelo débil. Al no estar el código en el repo, no es verificable.

**Acción:** revisar `src/build_models.py`; verificar alineación temporal de exógenas y escala del pronóstico.

## 4. Chequeo de no-leakage insuficiente para este target

El reporte valida `cmg_lag_1h[t] == cmg_usd_mwh[t-1]` (correcto pero insuficiente). Con un target proxy determinista en la forma intra-día, features como `hour`, `elevation_deg` y `GHI` **determinan la forma del target**. No es leakage técnico (están disponibles en tiempo real), pero sí **contaminación conceptual**: el modelo puede reconstruir el target sin información de mercado.

## 5. Cobertura del IC 95%: 87.5% (documentado, diagnóstico correcto)

La explicación del reporte (asimetría/colas del CMG) es razonable y el remedio propuesto (conformal post-hoc o cuantiles asimétricos) es el adecuado. El dashboard incluye un laboratorio interactivo de *conformal scaling* que muestra la cobertura empírica en función del factor de escala.

## 6. Consistencia interna (punto a favor)

- MAE 2.53 con MAPE 4.6% implica media ≈ 55 USD/MWh, coherente con el rango del duck curve documentado (~28 mediodía – ~87 peak).
- RMSE/MAE = 6.19/2.53 ≈ 2.45 indica errores de cola pesada, coherente con la asimetría documentada y con la cobertura IC por debajo del nominal.
- Períodos coherentes: datos 2023-01→2025-12, train/val/test bien delimitados, fold 2 del walk-forward = test principal.

## 7. Otras observaciones

- **Datos sintéticos de demanda/generación**: correctamente documentados como limitación; degradan las exógenas de sistema.
- **Sin features de eventos** (ERV, mantenimientos, fallas): los spikes reales no serán anticipados; con el target proxy además desaparecen del target.
- **Serpie de fechas**: reporte fechado 2026-07-20 con datos hasta 2025-12 — consistente.
- **README vs reporte**: estructuras de directorios consistentes entre ambos; ninguna contradicción detectada.

## 8. Resumen de severidad

| # | Hallazgo | Severidad |
|---|----------|-----------|
| 1 | Código no versionado / no reproducible | 🔴 Bloqueante |
| 2 | Fuga estructural por target proxy (métricas infladas) | 🔴 Alto |
| 3 | SARIMAX anómalo (posible bug de implementación) | 🟠 Medio-alto |
| 4 | No-leakage check insuficiente para target proxy | 🟠 Medio |
| 5 | Cobertura IC 87.5% < 95% | 🟡 Conocido/mitigable |
| 6 | Exógenas sintéticas | 🟡 Documentado |
| 7 | Sin features de eventos regulatorios | 🟡 Documentado |

---

*El dashboard interactivo (`dashboard/index.html`) presenta estos hallazgos en su sección "Auditoría" y visualiza la serie, el duck curve, los modelos y el laboratorio de intervalos con datos reconstruidos fielmente desde el reporte. Debe sustituirse por los CSV reales (`results/predictions_test.csv`) en cuanto el código se versione.*
