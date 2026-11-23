# Emisiones de CO₂ de vehículos

Proyecto individual del curso **Ciencia de datos con Python** (SENA, Análisis y Desarrollo de Software).

## Pregunta analítica

> ¿Qué características del vehículo (cilindrada, número de cilindros, tipo de combustible, transmisión y clase) influyen más en sus emisiones de CO₂ por kilómetro?

**Tipo de análisis:** descriptivo y diagnóstico.

## Fuentes de datos

| Dataset | Archivo en `data/raw/` | Fuente | Contenido |
| --- | --- | --- | --- |
| CO₂ Emission by Vehicles (Canadá) | `CO2 Emissions_Canada.csv` | [Kaggle](https://www.kaggle.com/datasets/debajyotipodder/co2-emission-by-vehicles), con datos del Gobierno de Canadá | 7 385 configuraciones de vehículos, modelos 2014–2020. CO₂ en g/km y consumo en L/100 km |
| Diccionario del dataset de Canadá | `Data Description.csv` | Kaggle | Significado de los códigos de transmisión y combustible |
| Vehicle Fuel Economy (EE. UU.) | `fuel_economy_epa.csv` | [fueleconomy.gov](https://www.fueleconomy.gov/feg/download.shtml) / Data.gov (EPA) | 50 407 vehículos, modelos 1984–2027, 84 columnas. CO₂ en g/milla y consumo en mpg |

Los archivos de `data/raw/` son los originales y **no se modifican**.

## Estructura del repositorio

```
emisiones_vehiculos/
├── README.md
├── .gitignore
├── requirements.txt
├── data/
│   ├── raw/                          # datasets originales (sin modificar)
│   │   ├── CO2 Emissions_Canada.csv
│   │   ├── Data Description.csv
│   │   └── fuel_economy_epa.csv
│   └── processed/                    # datasets limpios y estructurados
│       ├── co2_canada_clean.csv
│       ├── epa_vehicles_clean.csv
│       ├── vehicles_consolidated.csv
│       └── cleaning_log.csv
├── notebooks/
│   ├── data_quality.ipynb            # diagnóstico de calidad (actividad anterior)
│   └── data_preparation.ipynb        # limpieza, análisis y estructuración
└── reports/
    └── data_quality.md               # informe del diagnóstico de calidad
```

## Instrucciones de ejecución

1. Clonar el repositorio y entrar a la carpeta:
   ```bash
   git clone https://github.com/Sehuto96/emisiones-vehiculos-diagnostico.git
   cd emisiones-vehiculos-diagnostico
   ```
2. Crear y activar el entorno virtual:
   ```bash
   python -m venv .venv
   .venv\Scripts\Activate.ps1        # Windows (PowerShell)
   source .venv/bin/activate         # Linux / macOS
   ```
3. Instalar las dependencias:
   ```bash
   pip install -r requirements.txt
   ```
4. Abrir los notebooks en VS Code o en JupyterLab (`jupyter lab`), elegir el kernel `.venv` y ejecutar en este orden:
   1. `notebooks/data_quality.ipynb`: diagnóstico de calidad.
   2. `notebooks/data_preparation.ipynb`: genera los archivos de `data/processed/`.

El notebook usa rutas relativas (`../data/raw` y `../data/processed`), así que debe ejecutarse desde la carpeta `notebooks/`. Fue probado con pandas 3.

## Transformaciones realizadas (`data_preparation.ipynb`)

### Lección 2: limpieza y estandarización

Cada problema se analizó antes de decidir si **eliminar, corregir o conservar**.

| Problema | Dataset | Decisión |
| --- | --- | --- |
| 1 103 duplicados exactos | Canadá | **Eliminar**: el dataset junta 7 años sin columna de año, así que el mismo vehículo se repite |
| 294 duplicados implícitos (`Fusion Hybrid` / `FUSION HYBRID`) | Canadá | **Eliminar**: aparecen al estandarizar mayúsculas |
| 40 duplicados ocultos por la columna `id` | EPA | **Eliminar** |
| `co2 = -1` en los modelos anteriores a 2013 (valor centinela) | EPA | **Corregir** a nulo real; se usa `co2TailpipeGpm` convertido a g/km |
| Nulos de cilindrada y cilindros (1 651 de 1 653 son eléctricos o de hidrógeno) | EPA | **Conservar**: no tienen motor de combustión |
| `atvType` vacío | EPA | **Corregir** con el valor `'Conventional'` |
| Espacios dobles y mayúsculas distintas en marca y modelo | Ambos | **Corregir**: texto limpio y en mayúsculas |
| Transmisión en formatos distintos (`AS6` frente a `Automatic (S6)`) | Ambos | **Corregir**: nuevas columnas `transmission_type` y `gears` |
| 16 y 34 clases de vehículo distintas | Ambos | **Corregir**: 8 grupos comunes en `vehicle_group` |
| 84 columnas en la EPA | EPA | Conservar 17 útiles para la pregunta |
| Tipos de dato | Ambos | `int8`/`int16`/`int32`, tipos *nullable* (`Int8`, `Int16`) y `category` |

La limpieza está en **funciones reutilizables**, encadenadas en un **pipeline con `pipe()`**. Una función de registro anota filas, columnas, nulos y memoria después de cada paso (`cleaning_log.csv`), y la limpieza termina con validaciones mediante `assert`.

### Lección 3: filtrado, ordenamiento y agrupación

Se usan filtros con máscaras booleanas, `isin()`, `between()` y `query()`, ordenamiento con `nlargest()`/`nsmallest()`, `groupby()` + `agg()`, `pd.cut()` y `pivot_table()`.

| Indicador | Resultado principal |
| --- | --- |
| 1. Correlación con el CO₂ | Cilindrada 0,854 · cilindros 0,835 · número de marchas 0,151 |
| 2. CO₂ medio por rango de cilindrada | 198,8 g/km (≤ 2 L) → 341,2 g/km (> 5 L): **+72 %** |
| 3. CO₂ medio por combustible y por transmisión | En pickups, el diésel emite 9 % menos que la gasolina regular. Las diferencias por transmisión se explican sobre todo por la cilindrada |
| 4. % de configuraciones de altas emisiones (> 290 g/km) | Vans 100 %, pickups 51,9 %, automóviles 15,0 % |

**Respuesta a la pregunta:** las características que más influyen en las emisiones de CO₂ por km son la **cilindrada** y el **número de cilindros**, seguidas del **tipo de vehículo** (vans, pickups y deportivos). El combustible y la transmisión muestran diferencias, pero en buena parte se explican porque cada grupo tiene motores de distinto tamaño.

### Lección 4: combinación y estructuración

- **`merge`** de los datos de Canadá con una tabla de códigos de combustible (X, Z, D, E, N), validado con `validate='m:1'` e `indicator=True`.
- **`concat`** de Canadá y EE. UU. (EPA 2014–2020, solo vehículos con motor de combustión) después de armonizar columnas y unidades, con la columna `source` para indicar el origen. No se usó `merge` entre los dos países porque el dataset de Canadá no tiene año de modelo y marca + modelo no identifican un vehículo único.
- Validación de filas, columnas, claves e integridad después de cada combinación.

## Datasets procesados (`data/processed/`)

| Archivo | Filas | Descripción |
| --- | --- | --- |
| `co2_canada_clean.csv` | 5 988 | Canadá limpio, con descripción del combustible |
| `epa_vehicles_clean.csv` | 50 367 | EPA limpio, 22 columnas y CO₂ en g/km |
| `vehicles_consolidated.csv` | 14 624 | Canadá + EE. UU. (2014–2020) con 12 columnas comunes; dataset para las siguientes etapas |
| `cleaning_log.csv` | 10 | Registro de cambios del pipeline de limpieza |

## Calidad antes y después

| Dataset | Filas | Duplicados | Nulos | Memoria |
| --- | --- | --- | --- | --- |
| Canadá (antes → después) | 7 385 → 5 988 | 1 103 → 0 | 0 → 224 (marchas de las cajas CVT, no aplica) | 2,5 → 0,8 MB |
| EPA (antes → después) | 50 407 → 50 367 | 40 → 0 reales | 543 858 → 39 304 | 82,3 → 5,8 MB |

Las cifras de memoria corresponden a pandas 3 sin `pyarrow`.

## Historial de versiones

| Rama | Contenido |
| --- | --- |
| `main` | Diagnóstico de calidad (actividad anterior) |
| `develop` | Rama de integración |
| `feature/project-data-preparation` | Preparación de datos (esta actividad), fusionada en `develop` |

Commits de esta actividad, en orden:

| Commit | Avance |
| --- | --- |
| `feat: clean and standardize project data` | Limpieza y estandarización (Lección 2) |
| `feat: implement data filtering and aggregation` | Filtrado, agrupación e indicadores (Lección 3) |
| `feat: prepare and validate processed datasets` | Combinación, validación y exportación (Lección 4) |
| `docs: update project README` | Documentación |

