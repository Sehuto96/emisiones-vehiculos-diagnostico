# Informe de diagnóstico de calidad de datos

**Proyecto:** Emisiones de CO₂ en vehículos
**Curso:** Ciencia de datos con Python
**Autor:** Sebastian Hurtado 
**Fecha:** 30 de octubre de 2026

---

## 1. Objetivo

Diagnosticar la calidad de los dos datasets del proyecto antes de la fase de limpieza y análisis. El diagnóstico debe identificar su estructura, los tipos de datos, los valores faltantes, los duplicados y otros problemas que puedan afectar la respuesta a la pregunta de investigación:

> **¿Qué características del vehículo influyen en sus emisiones de CO₂ por kilómetro?**

## 2. Datasets analizados

| Dataset | Fuente | Origen de los datos | Archivo | Filas | Columnas |
|---|---|---|---|---|---|
| `df_co2` | Kaggle: *CO2 Emission by Vehicles* | Gobierno de Canadá | `data/raw/CO2 Emissions_Canada.csv` | 7 385 | 12 |
| `df_epa` | Data.gov / fueleconomy.gov: *Vehicle Fuel Economy Data* | Agencia de Protección Ambiental de EE. UU. (EPA) | `data/raw/fuel_economy_epa.csv` | 50 407 | 84 |

Como documentación de apoyo se usó `data/raw/Data Description.csv`, el diccionario de datos del Dataset 1.

## 3. Metodología

El diagnóstico se hizo en el notebook `notebooks/diagnostico_calidad.ipynb` con Python y Pandas (versión 3), en un entorno virtual (`.venv`). Los pasos fueron:

1. **Configuración y carga:** se verificó el directorio de trabajo y se cargó cada dataset en su propio DataFrame con `pd.read_csv()`.
2. **Inspección estructural** de cada dataset con `head()`, `tail()`, `shape`, `columns`, `dtypes`, `info()`, `describe()` y `describe(include="str")`.
3. **Revisión de valores faltantes** con `isnull().sum()` y su porcentaje por columna.
4. **Revisión de duplicados** con `duplicated().sum()`. En `df_epa` también se revisó excluyendo la columna `id`.
5. **Verificaciones adicionales:** vehículos con cero emisiones, valores atípicos (método IQR), valores de relleno (-1) y diferencia de unidades entre los datasets.

> **Nota técnica:** en pandas 3 las columnas de texto se muestran como `str` y no como `object`. Por eso se usó `describe(include="str")`, ya que `include="object"` genera una advertencia de obsolescencia.

## 4. Diagnóstico estructural

| Dataset | Filas | Columnas | Tipos de datos | Valores nulos | Duplicados | Memoria |
|---|---|---|---|---|---|---|
| df_co2 | 7 385 | 12 | 7 numéricas, 5 texto | Ninguno | 1 103 | 692,5 KB |
| df_epa | 50 407 | 84 | 59 numéricas (32 float64, 27 int64), 24 texto (str), 1 bool | 18 columnas con nulos (13 con >35 %, estructurales); `displ`/`cylinders` 3,28 % | 0 (40 sin la columna `id`) | 32,0 MB |

### Hallazgos de df_co2

- Cada fila es una configuración de vehículo (marca, modelo y características) con su consumo de combustible y sus emisiones de CO₂ en g/km, que es la variable objetivo.
- **No tiene valores nulos.** Las 12 columnas tienen 7 385 valores no nulos.
- Los tipos de datos son coherentes: 7 variables numéricas (tamaño del motor, cilindros, consumos y emisiones) y 5 de texto (marca, modelo, clase de vehículo, transmisión y tipo de combustible).
- Los rangos son plausibles: motor de 0,9 a 8,4 L, de 3 a 16 cilindros y emisiones de 96 a 522 g/km (media 250,6). No hay ceros ni valores negativos.
- **Tiene 1 103 filas duplicadas (≈14,9 %).**
- No incluye el año del modelo ni vehículos eléctricos o híbridos.

### Hallazgos de df_epa

- Es mucho más amplio: 50 407 registros y 84 columnas, de las cuales solo unas pocas son necesarias para la pregunta.
- 18 columnas tienen nulos. **13 superan el 35 %**, y algunas llegan al 99,7 % (`c240bDscr`, `c240Dscr`). Son **nulos estructurales**: describen características que solo tienen algunos vehículos, como carga eléctrica, turbo o supercargador, combustible alterno o sistema start-stop.
- Las variables clave (`co2TailpipeGpm`, `fuelType`, `VClass`, `year`, `make`, `model`) **no tienen nulos**.
- `displ` y `cylinders` tienen 3,28 % de nulos. Se verificó que 1 651 de los 1 653 registros sin cilindrada corresponden a vehículos sin motor de combustión (1 607 eléctricos, 42 de hidrógeno y 2 eléctricos/hidrógeno). **Solo 2 registros de gasolina son datos faltantes reales.**
- No hay duplicados exactos porque cada fila tiene un `id` único. Sin embargo, **al excluir `id` aparecen 40 registros repetidos.**

## 5. Problemas de calidad identificados

| # | Dataset | Problema | Impacto en el análisis | Acción propuesta |
|---|---|---|---|---|
| 1 | df_co2 | 1 103 filas duplicadas (≈14,9 %) | Distorsiona promedios y correlaciones | Eliminar con `drop_duplicates()` |
| 2 | df_epa | 40 registros repetidos ocultos por el `id` | Sobrerrepresenta algunos vehículos | Eliminar duplicados ignorando `id` |
| 3 | df_epa | 13 columnas casi vacías (nulos estructurales) | Ruido y peso innecesario | Seleccionar solo las variables relevantes |
| 4 | df_epa | Nulos en `displ` y `cylinders` (3,28 %) | Casi todos son vehículos sin motor de combustión | Separar eléctricos/hidrógeno; eliminar los 2 nulos reales |
| 5 | df_epa | Vehículos con 0 g/mi de CO₂ por el escape | Sesgan la relación motor–emisiones | Analizarlos aparte o excluirlos |
| 6 | df_epa | Valores de relleno (-1) en la columna `co2` | `info()` no los detecta como faltantes | Usar `co2TailpipeGpm` o reemplazar -1 por `NaN` |
| 7 | df_co2 | Posibles valores atípicos en emisiones (máx. 522 g/km frente a P75 de 288) | Pueden influir en medias y modelos | Revisarlos con el método IQR antes de decidir |
| 8 | Ambos | Unidades distintas (g/km y L/100 km frente a g/mi y mpg) | No se pueden comparar directamente | Convertir df_epa a g/km |
| 9 | df_co2 | Nombres de columnas poco uniformes | Dificulta el código y la unión de datasets | Estandarizar nombres |
| 10 | df_co2 | Códigos abreviados en `Fuel Type` y `Transmission` | No se interpretan sin el diccionario | Mapear con `Data Description.csv` |

La cuantificación de los problemas 5, 6 y 7 está en la sección 4 del notebook (verificaciones 1 a 3).

## 6. Conclusiones

- Los dos datasets son útiles para responder la pregunta de investigación. Las variables clave (emisiones, cilindrada, cilindros, tipo de combustible, transmisión y clase de vehículo) están disponibles y casi completas en ambos.
- **df_co2** es pequeño, completo y fácil de usar, pero su principal problema son los duplicados.
- **df_epa** es más rico (incluye años, eléctricos e híbridos), pero exige más preparación: seleccionar columnas, separar vehículos sin motor de combustión y tratar valores de relleno.
- La revisión mostró que no todos los problemas se ven con `info()`. Los duplicados ocultos por un identificador, los valores de relleno (-1) y los nulos estructurales solo aparecen al interpretar los datos con conocimiento del contexto.

## 7. Próximos pasos

1. Eliminar duplicados de los dos datasets.
2. Seleccionar las columnas relevantes de `df_epa`.
3. Separar los vehículos eléctricos y de hidrógeno del análisis de motores de combustión.
4. Tratar los valores de relleno (-1) y los nulos reales.
5. Estandarizar nombres de columnas y unidades (g/km).
6. Revisar los valores atípicos.
