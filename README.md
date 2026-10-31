# Emisiones de CO₂ en vehículos: diagnóstico de calidad de datos

Proyecto del curso **Ciencia de datos con Python**. Se diagnostica la calidad de dos datasets abiertos sobre emisiones de CO₂ de vehículos, como paso previo a la limpieza y el análisis.

**Autor:** Sebastian

## Pregunta de investigación

> ¿Qué características del vehículo influyen en sus emisiones de CO₂ por kilómetro?

Tipo de análisis: descriptivo / diagnóstico.

## Datasets

| Dataset | Fuente | Archivo | Filas × columnas |
|---|---|---|---|
| CO2 Emission by Vehicles (Canadá) | [Kaggle](https://www.kaggle.com/datasets/debajyotipodder/co2-emission-by-vehicles) | `data/raw/CO2 Emissions_Canada.csv` | 7 385 × 12 |
| Vehicle Fuel Economy Data (EE. UU., EPA) | [Data.gov](https://catalog.data.gov/dataset/fuel-economy-data) / [fueleconomy.gov](https://www.fueleconomy.gov/feg/download.shtml) | `data/raw/fuel_economy_epa.csv` | 50 407 × 84 |

`data/raw/Data Description.csv` es el diccionario de datos del dataset de Kaggle.

## Estructura del repositorio

```
emisiones_vehiculos/
├── data/
│   └── raw/                          # Datos originales sin modificar
│       ├── CO2 Emissions_Canada.csv
│       ├── Data Description.csv
│       └── fuel_economy_epa.csv
├── notebooks/
│   └── diagnostico_calidad.ipynb     # Inspección y diagnóstico con Pandas
├── reports/
│   └── informe_diagnostico.md        # Informe de resultados
├── .gitignore
├── README.md
└── requirements.txt
```

## Instalación y ejecución

1. Clonar el repositorio:
   ```bash
   git clone <URL-del-repositorio>
   cd emisiones_vehiculos
   ```
2. Crear y activar el entorno virtual:
   ```bash
   python -m venv .venv
   # Windows
   .venv\Scripts\activate
   # macOS / Linux
   source .venv/bin/activate
   ```
3. Instalar las dependencias:
   ```bash
   pip install -r requirements.txt
   ```
4. Abrir `notebooks/diagnostico_calidad.ipynb` en VS Code o JupyterLab (`jupyter lab`), seleccionar el kernel `.venv` y ejecutar todas las celdas.

> El proyecto usa **pandas 3**, que muestra las columnas de texto como `str` y no como `object`.

## Contenido del notebook

1. Configuración del entorno
2. Carga de datos
3. Inspección estructural: `head()`, `tail()`, `shape`, `columns`, `dtypes`, `info()`, `describe()`, nulos y duplicados por dataset, y tabla de diagnóstico estructural
4. Diagnóstico de problemas de calidad: verificaciones adicionales y tabla de problemas
5. Conclusiones del diagnóstico

## Principales hallazgos

- **df_co2:** sin valores nulos, pero con 1 103 filas duplicadas (≈14,9 %).
- **df_epa:** 13 columnas con más de 35 % de nulos estructurales, 40 registros repetidos ocultos por el `id`, vehículos eléctricos/hidrógeno sin datos de motor y valores de relleno (-1).
- Los datasets usan unidades distintas (g/km frente a g/mi), así que deben convertirse antes de compararlos.

El detalle completo está en [`reports/informe_diagnostico.md`](reports/informe_diagnostico.md).
