# SIG Avanzado 2026 · El tiempo, del SIG a la nube

Materiales de la parte de **Diego García Díaz y David Aragonés** (Laboratorio de SIG y Teledetección, Estación Biológica de Doñana, CSIC) en el *Curso avanzado de SIG en ecología*, 14–18 de septiembre de 2026.

## Presentación

**▶ [https://digdgeo.github.io/sig-avanzado-2026](https://digdgeo.github.io/sig-avanzado-2026)**

Teclas: ← → para navegar · **⇧← ⇧→** saltar las fichas · **O** orden del día · **F** pantalla completa.

Las diapositivas marcadas como **Ficha** son teoría de apoyo (Mann-Kendall y Sen, fenología, hidroperiodo, GeoParquet, clasificadores). Están para consultarlas con calma: si en clase vamos justos de tiempo, se saltan con ⇧→.

## Antes de las prácticas

1. **Comprueba tu navegador:** [comprobar_navegador.html](https://digdgeo.github.io/sig-avanzado-2026/ejercicios/comprobar_navegador.html). Tiene que salir en verde para usar GeoLibre.
2. **Crea tu cuenta de Google Earth Engine** siguiendo la sección [Google Earth Engine y tu cuenta](https://digdgeo.github.io/sig-avanzado-2026/#s-gee) de la presentación. Sin un proyecto registrado no funcionan las prácticas de Earth Engine ni de ndvi2gif.

## Ejercicios · la marisma en el tiempo

| Enunciado | Herramienta |
|---|---|
| [ejercicio_geolibre.md](ejercicios/ejercicio_geolibre.md) | GeoLibre web, en el navegador |
| [ejercicio_qgis_trajectools.md](ejercicios/ejercicio_qgis_trajectools.md) | QGIS + Trajectools |
| [ejercicio_qgis_sin_complementos.md](ejercicios/ejercicio_qgis_sin_complementos.md) | QGIS sin complementos, con datos ligeros |
| [00_chequeo_equipos.md](ejercicios/00_chequeo_equipos.md) | Comprobación de los equipos del aula |

## Python y Google Colab

| Notebook | |
|---|---|
| Introducción a Python | [![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Digdgeo/sig-avanzado-2026/blob/main/python/01_intro_python_complete.ipynb) |
| Ejercicios de Python | [![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Digdgeo/sig-avanzado-2026/blob/main/python/01_intro_python_exercises.ipynb) |
| Programación orientada a objetos | [![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Digdgeo/sig-avanzado-2026/blob/main/python/01b_python_classes.ipynb) |
| Introducción a pandas | [![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Digdgeo/sig-avanzado-2026/blob/main/python/01c_pandas_dataframes.ipynb) |
| Composites y series con ndvi2gif | [![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Digdgeo/sig-avanzado-2026/blob/main/python/02_ndvi2gif_curso.ipynb) |
| GeoLibre desde código | [![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Digdgeo/sig-avanzado-2026/blob/main/python/03_geolibre_codigo.ipynb) |
| ¿Reverdecen las cumbres? Sierra Nevada y Pirineos | [![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Digdgeo/sig-avanzado-2026/blob/main/python/04_cumbres_reverdecen.ipynb) |

Guías: [Google Colab](python/introduccion_colab.md) · [Anaconda](python/introduccion_anaconda.md)

## Google Earth Engine · editor de código

| Script | Qué hace |
|---|---|
| [ndvi_max_zonas.js](python/gee/ndvi_max_zonas.js) | NDVI máximo de 2023 con Landsat 8 y zonificación por altitud sobre Doñana. Se pega entero en [code.earthengine.google.com](https://code.earthengine.google.com). |

## Datos

Todo en [`datos/`](datos/):

| Carpeta o archivo | Contenido |
|---|---|
| `geoparquet/` | Formato recomendado para GeoLibre (EPSG:4326) |
| `geojson/` | Los mismos datos en GeoJSON (EPSG:4326) |
| `donana_tiempo_vectorial.gpkg` | Todo en un GeoPackage para QGIS (EPSG:25829) |
| `ligero/` | Versión reducida para equipos con poca memoria |
| `inundacion_por_recinto_2019_2020.csv` | Superficie inundada por recinto y fecha |
| `hidroperiodo_2019_2020_marisma.tif` | Hidroperiodo del ciclo 2019-2020 |

- **GPS de aves:** los datos son **simulados**. Sirven para aprender el método, no para sacar conclusiones sobre las especies.
- **Inundación e hidroperiodo:** proceden de las máscaras de agua del Protocolo v2 del LAST-EBD (Landsat, ciclo 2019-2020).

## Descargarlo todo

[Descargar el repositorio en ZIP](https://github.com/Digdgeo/sig-avanzado-2026/archive/refs/heads/main.zip)
