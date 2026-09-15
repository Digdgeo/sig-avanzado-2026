# Introducción a Google Colab

## ¿Qué es Google Colab?

**Google Colab** (Colaboratory) es un entorno de notebooks Jupyter gratuito que se ejecuta en la nube, sin necesidad de instalar nada en tu ordenador. Solo necesitas una cuenta de Google.

Accede en: [colab.research.google.com](https://colab.research.google.com)

---

## ¿Por qué usar Colab?

- Sin instalación: todo funciona desde el navegador.
- Acceso gratuito a CPUs (y GPUs/TPUs para tareas de machine learning).
- Integración directa con Google Drive para guardar y compartir notebooks.
- Ideal para primeros pasos con Python y para compartir ejemplos reproducibles.

---

## Interfaz básica

Un notebook de Colab está formado por **celdas** de dos tipos:

- **Celda de código**: ejecuta Python. Pulsa `Shift + Enter` para correr la celda.
- **Celda de texto**: escribe en Markdown (explicaciones, títulos, fórmulas).

Para añadir una celda nueva usa los botones `+ Code` / `+ Text` en la parte superior, o el atajo `Ctrl + M B` (celda debajo) / `Ctrl + M A` (celda encima).

---

## Atajos de teclado útiles

| Acción | Atajo |
|--------|-------|
| Ejecutar celda | `Shift + Enter` |
| Ejecutar celda sin avanzar | `Ctrl + Enter` |
| Añadir celda debajo | `Ctrl + M B` |
| Añadir celda encima | `Ctrl + M A` |
| Eliminar celda | `Ctrl + M D` |
| Convertir a texto | `Ctrl + M M` |
| Convertir a código | `Ctrl + M Y` |
| Buscar y reemplazar | `Ctrl + H` |

---

## Instalar librerías

Colab incluye muchas librerías preinstaladas (NumPy, Pandas, Matplotlib...). Para instalar algo adicional usa `pip` directamente en una celda:

```python
!pip install geopandas
```

El `!` indica que es un comando de sistema (no Python puro). La instalación dura solo mientras la sesión está activa — si cierras y reabres el notebook hay que instalar de nuevo.

---

## Integración con Google Drive

Para acceder a tus ficheros en Drive:

```python
from google.colab import drive
drive.mount('/content/drive')
```

Tras autenticarte con tu cuenta de Google, tus ficheros estarán disponibles en `/content/drive/MyDrive/`.

---

## Subir ficheros locales

Para subir un fichero desde tu ordenador directamente a la sesión:

```python
from google.colab import files
uploaded = files.upload()
```

Se abrirá un selector de ficheros. El fichero quedará disponible en `/content/`.

---

## Limitaciones de Colab

| Aspecto | Detalle |
|---------|---------|
| Tiempo de sesión | ~12 horas (se reinicia si hay inactividad) |
| RAM | ~12 GB (plan gratuito) |
| Almacenamiento | Temporal — los ficheros se pierden al cerrar la sesión |
| GPU/TPU | Disponibles pero con tiempo limitado en plan gratuito |

> **Importante:** Colab es perfecto para aprender y para prototipar. Para proyectos en producción o análisis muy grandes, es mejor trabajar en local con un entorno conda.

---

## Colab vs entorno local (conda)

| | Google Colab | Entorno local (conda) |
|---|---|---|
| Instalación | Ninguna | Miniconda + environment.yml |
| Persistencia | No (sesión temporal) | Sí |
| Control de versiones | Limitado | Total |
| Velocidad para datos grandes | Limitada | Depende del equipo |
| Ideal para | Aprender, compartir | Proyectos reales |

En este curso usamos Colab para Python, Google Earth Engine y `ndvi2gif`: así no hay que instalar nada en los ordenadores del aula.
