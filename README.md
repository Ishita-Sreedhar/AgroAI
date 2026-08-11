# AgroAI

An ML-powered crop advisory system. Given soil nutrient levels and climate conditions, AgroAI recommends the most suitable crop (plus 3 alternatives), estimates its water requirement, and predicts time to harvest.

## Status

🚧 In progress — data collection and pipeline setup underway.

## Problem

Farmers often lack quick access to data-driven guidance on crop selection, irrigation planning, and harvest scheduling. AgroAI combines three ML models into a single tool that takes basic soil/climate inputs and returns actionable recommendations.

## Features

- **Crop recommendation** — predicts the best-suited crop from soil (N, P, K, pH) and climate (temperature, humidity, rainfall) inputs, with the top 3 alternative crops
- **Water requirement prediction** — estimates irrigation water needed for the recommended crop
- **Harvest time prediction** — estimates days to harvest based on crop and conditions

## Data Sources

| Module | Dataset | Notes |
|---|---|---|
| Crop recommendation | [Crop Recommendation Dataset (Kaggle)](https://www.kaggle.com/datasets/atharvaingle/crop-recommendation-dataset) | 2200 rows, 22 crops, balanced (100 samples/crop) |
| Water requirement | [Irrigation Water Requirement Prediction Dataset (Kaggle)](https://www.kaggle.com/datasets/miadul/irrigation-water-requirement-prediction-dataset) | Covers a subset of the 22 crops |
| Water requirement (gap-fill) | FAO Irrigation & Drainage Paper 24, [Crop Water Needs, Table 8](https://www.fao.org/4/s2022e/s2022e07.htm) | Crop coefficient (Kc) reference table, used to derive water need for crops missing from the Kaggle dataset |
| Harvest time | [Agriculture Crop Yield Dataset (Kaggle)](https://www.kaggle.com/datasets/samuelotiattakorah/agriculture-crop-yield) | 1M rows, covers 6 crops (Wheat, Rice, Maize, Barley, Soybean, Cotton) with a `Days_to_Harvest` column |
| Harvest time (gap-fill) | Agronomy reference maturity-period ranges | Static lookup table with climate-based adjustment, for the 16 crops not covered by the Kaggle dataset |

### Water requirement — merge methodology

The Kaggle irrigation dataset doesn't cover all 22 crops from the recommendation dataset. Missing crops are gap-filled using the FAO Kc reference table:

1. Diff the crop lists to identify coverage gaps
2. For covered crops, use the Kaggle data directly (`source: measured`)
3. For missing crops, compute `water_need_mm ≈ Kc_mid × ETo × growth_stage_days`, generating synthetic rows across a realistic range of climate inputs (`source: fao_derived`)
4. Concatenate both into one merged dataset, with the `source` column retained for downstream evaluation

Model performance is evaluated separately on `measured` vs `fao_derived` rows to surface any gap in reliability between real and estimated data — see Limitations below.

**Perennial crops (banana, mango, grapes, apple, orange, papaya, coconut, coffee, pomegranate)** don't have a single `growth_stage_days` value the way annual crops do — they're handled as a separate case in the merge logic, using an annual water requirement instead of a per-season one.

## Tech Stack

- Python, scikit-learn, pandas, numpy
- Streamlit (app interface)
- Hugging Face Spaces (deployment)
- Developed in Google Colab

## Project Structure

```
AgroAI/
├── data/
│   ├── raw/            # original downloaded datasets + FAO Kc reference
│   └── processed/       # cleaned/merged datasets
├── models/               # saved .pkl models
├── notebooks/
│   ├── 01_setup.ipynb
│   ├── 02_crop_recommendation.ipynb
│   ├── 03_water_requirement.ipynb
│   ├── 04_harvest_time.ipynb
│   └── 05_integration.ipynb
├── app/                  # Streamlit app
└── README.md
```

## Roadmap

| Days | Phase |
|---|---|
| 1 | Setup — folders, datasets, FAO Kc table, environment |
| 2–4 | Crop recommendation model |
| 5–7 | Water requirement — merge + model (includes separate annual-basis handling for perennial crops) |
| 8–9 | Harvest time — model + lookup table |
| 10–11 | Integration — unified prediction class |
| 12–14 | Streamlit app + deployment |
| 15 | Documentation |
| 16–17 | Buffer / polish |

## Known Limitations

- Water requirement for ~16 of 22 crops relies on FAO-derived synthetic estimates rather than measured data
- Harvest time model is directly trained on only 6 crops; the remaining 16 use a static lookup table with a climate-based adjustment rule, which is less precise than a learned model
- ETo (reference evapotranspiration) is approximated with a regional constant rather than computed from live weather data
- Perennial crops use annual water requirement figures rather than the per-growth-stage estimates used for annual crops, since they don't have a fixed growth-stage duration

## Setup

1. Clone/open in Google Colab
2. Mount Google Drive, ensure `data/raw/` contains the datasets listed above
3. Run notebooks in order (`01` → `05`)
4. Run the Streamlit app locally or via the Hugging Face Space link (once deployed)
