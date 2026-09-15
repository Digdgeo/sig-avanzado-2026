# SIG Avanzado 2026 · El tiempo, del SIG a la nube

Materiales de la parte de **Diego García Díaz y David Aragonés** (Laboratorio de SIG y Teledetección, Estación Biológica de Doñana, CSIC) en el *Curso avanzado de SIG en ecología*, 14–18 de septiembre de 2026.

## Presentación

**▶ [https://digdgeo.github.io/sig-avanzado-2026](https://digdgeo.github.io/sig-avanzado-2026)**

Teclas: ← → para navegar · **O** orden del día · **F** pantalla completa.

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

Guías: [Google Colab](python/introduccion_colab.md) · [Anaconda](python/introduccion_anaconda.md)

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
