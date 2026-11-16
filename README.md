# Vaca Muerta: pronóstico de producción y detección de anomalías por pozo

El proyecto analiza la producción mensual de petróleo, gas y agua de pozos operados por YPF en Vaca Muerta. Después de explorar y preparar los datos, comparamos un pronóstico supervisado con un predictor de persistencia y usamos Isolation Forest para identificar perfiles de pozos y meses inusuales en el historial observado.

## Preguntas de trabajo

- **Pronóstico:** ¿podemos estimar las tasas del mes siguiente mejor que suponiendo que se mantienen iguales? Comparamos Random Forest con un predictor de persistencia, que repite el valor del mes actual.
- **Anomalías:** ¿qué pozos tienen un perfil inusual en sus primeros doce meses y qué registros mensuales muestran un comportamiento inusual dentro de su grupo? También exploramos si esas señales coinciden con cambios de extracción, aumentos de producción o valores atípicos de los boxplots.

## Datos y alcance

La fuente es el conjunto público de **producción de petróleo y gas por pozo — Capítulo IV**, de la Secretaría de Energía de Argentina, en su archivo de producción no convencional.

- [Catálogo de datos](https://datos.energia.gob.ar/dataset/produccion-de-petroleo-y-gas-por-pozo).
- [Documentación de la fuente](https://github.com/datosenergia/produccion-de-petroleo-y-gas-por-pozo).

La exploración guardada informa 431.068 registros y 40 columnas en el archivo original, con datos entre enero de 2006 y septiembre de 2026. El CSV procesado incluido en este repositorio contiene:

| Característica | Alcance de la muestra procesada |
| --- | --- |
| Operadora seleccionada | YPF |
| Formación seleccionada | Vaca Muerta |
| Período observado | Octubre de 2013 a septiembre de 2026 |
| Frecuencia | Mensual, por `idpozo` |
| Registros | 30.325 |
| Identificadores de pozo | 512 |
| Columnas | 21 |
| Tipos de pozo | Petrolífero y gasífero |

| Área | Pozos distintos | Registros |
| --- | ---: | ---: |
| Loma Campana | 146 | 13.739 |
| La Amarga Chica | 303 | 12.211 |
| El Orejano | 63 | 4.375 |

La selección conserva pozos cuyo historial observado de métodos de extracción contiene únicamente surgencia natural, o la combinación de surgencia natural y bombeo mecánico. Esta decisión restringe la muestra: los resultados no representan a todos los pozos de Vaca Muerta.

La clave `idpozo` identifica al pozo por formación productiva en la fuente. El archivo procesado no presenta duplicados en la combinación `idpozo`, `anio` y `mes`.

## Recorrido del análisis

### 1. Exploración y preparación

En [01_exploracion_inicial.ipynb](notebooks/01_exploracion_inicial.ipynb) se revisan identificadores, fechas, valores faltantes, duplicados, estados y métodos de extracción. Luego se seleccionan la operadora, las áreas, la formación y las combinaciones de extracción que definen la muestra.

El análisis incluye un mapa de pozos, distribuciones de producción, comparaciones entre métodos de extracción, producción por edad observada, volúmenes anuales y correlaciones por pozo. Los años con cobertura parcial se identifican para evitar comparaciones directas con años completos.

También se construyen variables de producción por día efectivo de funcionamiento, a partir del volumen mensual y de `tef`, y una aproximación a la edad del pozo basada en el historial disponible. No contamos con el dato especifico de `vida_util` por pozo y tuvimos que descartarlo porque no nos aportaba data significativa para nuestro analisis y posterior modelo.

### 2. Pronóstico del mes siguiente

En [02_modelos_supervisados.ipynb](notebooks/02_modelos_supervisados.ipynb) se implementa un `MultiOutputRegressor` con un `RandomForestRegressor` para cada uno de estos objetivos:

| Objetivo | Interpretación | Unidad cuando `tef > 0` |
| --- | --- | --- |
| `prod_pet_dia_lead_1` | Tasa de petróleo del próximo mes | m³ por día efectivo |
| `prod_gas_dia_lead_1` | Tasa de gas del próximo mes | Miles de m³ por día efectivo |
| `prod_agua_dia_lead_1` | Tasa de agua del próximo mes | m³ por día efectivo |

Las variables predictoras incluyen producción del mes actual, rezagos de uno, dos y tres meses, método de extracción actual y sus rezagos, profundidad, tipo de pozo, área, edad observada, año y una codificación cíclica del mes.

Los rezagos y objetivos se comprueban contra la distancia entre meses: un registro separado por un hueco temporal no se interpreta como el mes inmediatamente anterior o siguiente. Las filas con valores faltantes en las variables seleccionadas se excluyen del modelado.

#### Evaluación temporal

La separación se realiza por meses completos, compartidos por todos los pozos, en orden cronológico:

- **Desarrollo:** enero de 2014 a enero de 2024, 121 meses.
- **Test:** febrero de 2024 a agosto de 2026, 31 meses.
- **Validación interna:** cinco particiones expansivas de `TimeSeriesSplit`, con ventanas de validación de 12 meses dentro del conjunto de desarrollo.
- **Ajuste:** 10 ensayos de Optuna, maximizando el R² medio entre objetivos y particiones.

Estas fechas corresponden al **mes de referencia de las variables predictoras**. El objetivo está en el mes siguiente; por ejemplo, agosto de 2026 se utiliza para predecir septiembre de 2026.

El modelo se evalúa con R², RMSE y MAE, y se compara con el predictor de persistencia sobre las mismas observaciones de test.

#### Resultados preliminares guardados

| Fluido | R² Random Forest | R² persistencia | RMSE Random Forest | RMSE persistencia |
| --- | ---: | ---: | ---: | ---: |
| Petróleo | 0,8436 | 0,8146 | 15,0709 | 16,4061 |
| Gas | 0,9402 | 0,9461 | 4,0001 | 3,7951 |
| Agua | 0,6233 | 0,5575 | 15,3552 | 16,6431 |

El RMSE se expresa en la unidad del objetivo correspondiente. En esta ejecución, Random Forest mejora el pronóstico de petróleo y agua frente a persistencia; para gas, persistencia obtiene mejores resultados. Estas métricas evalúan tasas de producción, no la detección de fallas ni la utilidad de un ranking de revisión.

### 3. Detección de anomalías con Isolation Forest

[03_modelos_no_supervisados.ipynb](notebooks/03_modelos_no_supervisados.ipynb)
Ajusta **cuatro modelos**: uno para detectar pozos anómalos y otro para meses anómalos para cada grupo:

- **La Amarga Chica y Loma Campana:** 449 pozos petrolíferos.
- **El Orejano:** 63 pozos de un yacimiento predominantemente gasífero.

#### Preparación y criterios de evaluación

Se excluyen de los cuatro modelos los meses con **petróleo, gas y agua simultáneamente en cero**; un cero en uno o dos fluidos se conserva.

| Modelo | Qué evalúa | Variables principales |
| --- | --- | --- |
| Pozos anomalos | Una fila por pozo. Requiere los doce meses de edad 0–11 presentes y válidos después del filtro. | Media, mediana, desviación estándar, mínimo y máximo de las tasas y del tiempo operativo; proporciones de categorías y cambios de método dentro del primer año. Se aplica `log1p` a los resúmenes de producción. |
| Meses anomalos | Una fila por pozo y mes, desde el mes de edad 1, con el mes actual y el anterior de calendario válidos. Incluye también pozos que el modelo del primer año no pudo evaluar. | Tasas actuales y cambios respecto del mes anterior en escala logarítmica, edad, fracción de tiempo operativo y categorías del mes. Los aumentos y las transiciones de cero a un valor positivo por fluido se registran como contexto para interpretar las alertas. |

Las categorías de extracción, estado, tipo de pozo, yacimiento y clasificación se codifican como indicadores. Para el perfil anual se promedian dentro de los meses 0–11 y se descartan indicadores según el grupo. El identificador, la fecha, las coordenadas, la inyección de agua y la subclasificación quedan fuera de los predictores.

La puntuación es `-score_samples`: cuanto mayor, más inusual dentro de ese modelo. El umbral automático es 0,5 en esa escala y no fija un porcentaje de anomalías.

#### Resultados de los cuatro modelos

| Grupo | Modelo | Casos evaluables | Casos marcados | % marcado |
| --- | --- | ---: | ---: | ---: |
| La Amarga Chica y Loma Campana | Pozos anómalos | 384 | 53 | 13,80 % |
| El Orejano | Pozos anómalos | 56 | 11 | 19,64 % |
| La Amarga Chica y Loma Campana | Meses anómalos | 22.662 | 3.801 | 16,77 % |
| El Orejano | Meses anómalos | 3.594 | 394 | 10,96 % |

El modelo mensual marcó al menos un mes en 436 de los 443 pozos evaluables del primer grupo y en 62 de los 63 de El Orejano. Los pozos con más meses observados tienen más oportunidades de acumular alertas.

## Estructura del repositorio

```text
vaca-muerta-produccion/
├── data/
│   ├── raw/
│   │   └── produccin-de-pozos-de-gas-y-petrleo-no-convencional.csv
│   └── processed/
│       └── produccion_limpia_filtrada.csv
├── notebooks/
│   ├── 01_exploracion_inicial.ipynb
│   ├── 02_modelos_supervisados.ipynb
│   └── 03_modelos_no_supervisados.ipynb
├── reports/
│   └── figures/
├── .gitattributes
├── .gitignore
├── README.md
└── requirements.txt
```

## Cómo ejecutar

Se requiere Python 3, Git y, para recuperar el archivo original, Git LFS. Las dependencias se detallan en [requirements.txt](requirements.txt), que incluye las del notebook 3 (entre ellas scikit-learn, Folium e ipywidgets). El notebook 2 también utiliza **Optuna**, que todavía no está incluido en ese archivo.

### Preparar el entorno

```bash
git lfs install
git clone https://github.com/molijoaquin-DA/vaca-muerta-produccion.git
cd vaca-muerta-produccion
git lfs pull

python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pip install optuna
python -m jupyter lab
```

En Windows, activar el entorno con `.venv\Scripts\Activate.ps1` desde PowerShell.

El archivo de `data/raw/` está gestionado con Git LFS. Antes de ejecutar la exploración, comprobar que contiene los datos CSV y no un pequeño archivo de texto que empieza con `version https://git-lfs.github.com/spec/v1`. Una descarga ZIP puede contener solamente ese puntero.

### Ejecutar los notebooks

Abrir Jupyter desde la raíz del repositorio o desde `notebooks/`.

1. Ejecutar `01_exploracion_inicial.ipynb` para revisar y preparar los datos originales.
2. Guardar el resultado en `data/processed/produccion_limpia_filtrada.csv`.
3. Ejecutar `02_modelos_supervisados.ipynb` para construir rezagos, ajustar los modelos y evaluar el pronóstico.
4. Ejecutar `03_modelos_no_supervisados.ipynb` de arriba abajo para ajustar los cuatro modelos

La celda final de exportación del primer notebook utiliza `google.colab.files`. Para ejecutar localmente, reemplazar esa celda por:

```python
ruta_salida = raiz / "data" / "processed" / "produccion_limpia_filtrada.csv"
ruta_salida.parent.mkdir(parents=True, exist_ok=True)
df_filtrado.to_csv(ruta_salida, index=False, encoding="utf-8-sig")
```
