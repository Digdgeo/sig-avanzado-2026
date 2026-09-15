# Chequeo de equipos · aula con Ubuntu (≈2017) y 8 GB de RAM

**Plan principal:** GeoLibre web (**web.geolibre.app**) con los datos completos en `datos/geoparquet/`.

Con 8 GB de RAM hay margen de sobra: DuckDB-WASM usa como mucho 4 GB por pestaña y el ejercicio completo se queda muy por debajo. El riesgo real en un Ubuntu de 2017 es el **navegador**, no la memoria.

---

## Antes de clase (si puedes, hoy mismo)

### 1 · Pregunta o comprueba en un equipo del aula

Abre un terminal (`Ctrl + Alt + T`):

```bash
lsb_release -d                                   # versión de Ubuntu
free -h                                          # RAM total y libre
firefox --version; google-chrome --version; chromium-browser --version
qgis --version 2>/dev/null | head -1             # por si hay que ir al plan C
```

### 2 · Qué navegador sirve

GeoLibre se publica sin adaptar el código a navegadores antiguos (`target: esnext`). El mínimo teórico es **Chrome/Edge 111 o Firefox 114**, pero lo seguro es una versión **del último año**.

| Navegador | Situación en Ubuntu 16.04 / 18.04 |
|---|---|
| **Google Chrome** (dices que está instalado) | ⚠️ Mira la versión en `chrome://version`. Chrome actual exige Ubuntu 20.04 o posterior, así que en un Ubuntu de 2017 **no pasará de una versión antigua aunque se actualice**. Si da < 111 no funcionará; entre 111 y ~120, prueba |
| **Firefox** actual | ✅ Solo pide glibc 2.17 y GTK 3.14. Es la salida segura |
| Firefox del repositorio de Ubuntu | ⚠️ Puede estar congelado. Mira `firefox --version` |

**La forma rápida de saberlo:** abre en cada equipo **https://digdgeo.github.io/sig-avanzado-2026/ejercicios/comprobar_navegador.html** (o el archivo `ejercicios/comprobar_navegador.html` con doble clic). Dice en verde o rojo si ese navegador puede con GeoLibre. Si sale en rojo, instala el Firefox portátil (apartado 3).

### 3 · Firefox portátil en la carpeta personal (sin sudo)

```bash
cd ~
wget -O firefox.tar.xz "https://download.mozilla.org/?product=firefox-latest-ssl&os=linux64&lang=es-ES"
tar -xJf firefox.tar.xz
~/firefox/firefox -P --no-remote &      # crea un perfil nuevo, sin tocar el del sistema
```

- **Tamaño y tiempo:** la descarga pesa unos 80 MB, así que en 20 equipos cuesta unos minutos de red.
- **Alternativa en red lenta:** si el aula tiene carpeta compartida o USB, descarga el archivo una vez y cópialo.
- **Aula que se restaura al apagar** (congeladas): hay que hacerlo al principio de la sesión.

---

## En clase · 5 minutos

1. **WebGL2.** Abre `https://get.webgl.org/webgl2/`. Debe verse el cubo girando: el mapa de GeoLibre (MapLibre) lo necesita. En equipos de 2017 con gráfica Intel suele funcionar; si no, en Firefox ve a `about:config` y comprueba que `webgl.disabled` está en `false`.
2. **GeoLibre.**
   - Abre **web.geolibre.app**.
   - *Añadir datos → Capa vectorial* → `datos/geoparquet/gps_aves_2019_2020.parquet`.
3. **SQL.**
   - *Procesamiento → Espacio de trabajo SQL* → `SELECT count(*) FROM gps_aves_2019_2020` → **Ejecutar**.
   - Deben salir **45.544** filas.

Si los puntos aparecen y el recuento sale en menos de 30 s, **ese equipo está listo**.

## Decisión según lo que pase

| Situación | Qué hacer |
|---|---|
| Todo va bien | `ejercicio_geolibre.md` con `datos/geoparquet/` |
| Carga pero va lento (animación a saltos, SQL > 20 s) | Mismo enunciado con `datos/ligero/`: `gps_aves_2019_2020` → `gps_aves_ligero` e `inundacion_2019_2020` → `inundacion_ligero` |
| El mapa sale en blanco o da error de WebGL | Probar Firefox portátil; si sigue, trabajo en pareja con un equipo que funcione |
| Internet cae o va muy lento | Demostración en el proyector (descarga GeoLibre en tu portátil antes) y `ejercicio_qgis_sin_complementos.md` si el QGIS del aula es ≥ 3.14 |

## Consejos

- **Solo GeoLibre abierto:** cierra el resto de pestañas y aplicaciones.
- **Recarga forzada** (`Ctrl + Mayús + R`) si la interfaz se ve rara: GeoLibre publica versiones a menudo y la caché vieja da problemas.
- **GeoParquet, no GeoJSON:** pesa unas 10 veces menos y carga mucho antes.
- **Guarda el proyecto** al final de cada bloque: si se recarga la pestaña, se pierden las capas y las vistas SQL.
- **No instales GeoLibre de escritorio** en estos equipos: el paquete para Linux se compila sobre Ubuntu 22.04 y no arrancará en un Ubuntu de 2017.
