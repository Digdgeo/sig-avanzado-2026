# Ejercicio · La marisma en el tiempo con GeoLibre

**Pregunta:** ¿cómo usan tres especies de aves una marisma que se llena y se vacía?

**Duración:** unas 2 horas · **Herramienta:** [GeoLibre](https://web.geolibre.app) en el navegador, sin instalar nada · **Datos:** carpeta `datos/geoparquet/`

> Los datos GPS son **simulados**: sirven para aprender el método, no para sacar conclusiones sobre estas especies. La inundación sí es real (Landsat, Protocolo v2, ciclo 2019-2020).

---

## Antes de empezar: cinco cosas que conviene saber de GeoLibre

1. **Todo corre en tu navegador.** Tus datos no salen del ordenador. Si recargas la página, pierdes las capas y las vistas SQL: guarda el proyecto (*Proyecto → Guardar*) al final de cada bloque.
2. **Las fechas llegan como números.** Al cargar un GeoParquet, GeoLibre guarda `timestamp`, `fecha_hora_local`, `fecha_ini` y `fecha_fin` como **milisegundos desde 1970**. En SQL se convierten con `epoch_ms(timestamp)`. Se pueden comparar y restar directamente (una hora son 3.600.000 ms).
3. **El Espacio de trabajo SQL ejecuta una sola sentencia cada vez.** Nada de encadenar varias con `;`. Las vistas (`CREATE VIEW`) sí se mantienen entre ejecuciones.
4. **En la tabla de resultados, las fechas se ven como números.** `epoch_ms(timestamp)` devuelve una fecha de verdad, pero la tabla la muestra en milisegundos. Para leerla, conviértela a texto: `epoch_ms(timestamp)::VARCHAR`.
5. **GeoLibre se actualiza varias veces al día.** Si la página se queda en blanco o un menú deja de responder, recarga con **Ctrl+Shift+R** (antes guarda el proyecto). Algún nombre de menú puede no coincidir exactamente con este guion.

---

## Bloque 1 · Explorar el tiempo (20 min)

### 1.1 Cargar los datos

1. Abre **web.geolibre.app**. Si no está en español: *Configuración → Idioma → Español*.
2. **Arrastra al mapa** los ficheros, de uno en uno (también puedes usar *Añadir datos → Capa vectorial*):
   - `gps_aves_2019_2020.parquet` (45.544 puntos)
   - `inundacion_2019_2020.parquet` (3.460 polígonos)
   - `recintos_marisma.parquet`
3. Ordena las capas: recintos arriba (solo contorno), puntos en medio e inundación al fondo.
4. Estilo de los puntos: *Estilo → Categorizado* por `nombre_comun`.

**Pregunta 1.1.** Encuadra toda la capa de puntos. ¿Por qué el mapa llega hasta Portugal, Málaga y el mar de Alborán si las aves viven en Doñana? No borres nada todavía; lo resolveremos en el bloque 2.

### 1.2 Animar con el Control de tiempo

1. *Complementos → Control de tiempo → Activar*. Aparece una línea temporal en la parte inferior del mapa.
2. En la capa de puntos, abre el menú de acciones (`⋯`) → **Vincular al control deslizante de tiempo…**
   - *Propiedad de tiempo*: `timestamp`.
   - *Mostrar entidades*: **En el paso actual**.
3. En la línea temporal, abajo a la derecha, elige el paso **D** (día). Así cada paso muestra solo los fixes de ese día. Pulsa ▶ y ajusta la velocidad (ms por paso).
4. Vincula también `inundacion_2019_2020` con la propiedad `fecha_ini` y *Mostrar entidades*: **Dentro de tres pasos antes y después**.

**Preguntas**

- **1.2** ¿Cuándo llegan y cuándo se van los ánsares? ¿Y las espátulas? ¿Por dónde entran y por dónde salen?
- **1.3** Localiza la semana en que se inunda la marisma. ¿Qué cambia en la distribución de los flamencos?
- **1.4** El Control de tiempo filtra por **un** campo de fecha, pero cada polígono de inundación es válido **entre** `fecha_ini` y `fecha_fin`. Con el paso diario y ±3 pasos, cada máscara solo aparece 3 días antes y 3 días después de su `fecha_ini`. Las imágenes Landsat llegan cada 8-16 días: ¿cuántos días verás la marisma sin ninguna máscara? ¿Cuándo verías dos superpuestas?
- **1.5** Cambia *Mostrar entidades* de la capa de puntos a **Todo hasta el paso actual (acumulativo)**. ¿Qué ves que no veías con el paso de 1 día? ¿Qué se pierde?

### 1.3 Primer vistazo en SQL

*Procesamiento → Espacio de trabajo SQL*. Las capas cargadas aparecen como tablas en *Capas consultables*.

```sql
WITH p AS (
  SELECT id_ave, nombre_comun, timestamp,
         (timestamp - LAG(timestamp) OVER (PARTITION BY id_ave ORDER BY timestamp)) / 60000.0 AS dt_min
  FROM gps_aves_2019_2020
)
SELECT id_ave, nombre_comun, count(*) AS n_fixes,
       epoch_ms(min(timestamp))::VARCHAR AS primer_fix_utc,
       epoch_ms(max(timestamp))::VARCHAR AS ultimo_fix_utc,
       round(median(dt_min))    AS intervalo_mediano_min,
       round(max(dt_min) / 60, 1) AS mayor_hueco_h
FROM p
GROUP BY ALL
ORDER BY id_ave
```

- **1.6** ¿Qué frecuencia de muestreo tiene cada especie? ¿Qué ave tiene un hueco anómalo, de cuántas horas y en qué fechas?

---

## Bloque 2 · De puntos a trayectorias (30 min)

### 2.1 Con la herramienta

*Procesamiento → Caja de herramientas de GeoLibre → Vectorial → Movimiento y tiempo → **Velocidad de trayectoria***

- Capa: `gps_aves_2019_2020`.
- Identificador: `id_ave`.
- Tiempo: `timestamp`.

El resultado son **segmentos** entre fixes consecutivos, con distancia, duración y velocidad. Simbolízalos por velocidad con *Estilo → Graduado*.

- **2.1** ¿Dónde están los segmentos más rápidos? ¿Parecen vuelos o errores?

### 2.2 Con SQL: la función de ventana

Crea una vista con las métricas de cada paso. `LAG` mira el fix **anterior** y `LEAD` el **siguiente** del mismo individuo. Las distancias se miden en EPSG:25829 (metros).

```sql
CREATE OR REPLACE VIEW pasos AS
WITH u AS (
  SELECT *, ST_Transform(geom, 'EPSG:4326', 'EPSG:25829', always_xy := true) AS g_utm
  FROM gps_aves_2019_2020
)
SELECT id_fix, id_ave, nombre_comun, timestamp, fecha_hora_local, hdop, n_sat, geom,
       ST_Distance(g_utm, LAG(g_utm)  OVER w)        AS dist_prev_m,
       ST_Distance(g_utm, LEAD(g_utm) OVER w)        AS dist_next_m,
       (timestamp - LAG(timestamp) OVER w) / 1000.0  AS dt_prev_s,
       (LEAD(timestamp) OVER w - timestamp) / 1000.0 AS dt_next_s
FROM u
WINDOW w AS (PARTITION BY id_ave ORDER BY timestamp)
```

Ahora, sobre la vista `pasos`, escribe tú las consultas:

- **2.2** Velocidad mediana, percentil 99 y nº de pasos por encima de 80 km/h, **por especie**. La velocidad en km/h es `3.6 * dist_prev_m / dt_prev_s`. Pista: `median()`, `quantile_cont(x, 0.99)` y `count(*) FILTER (WHERE …)`.
- **2.3** Lista los 30 pasos más rápidos con su velocidad de llegada **y** de salida, HDOP y nº de satélites. Clasifica a mano cinco de ellos como *error* o *vuelo real*. Pista: un **pico** tiene llegada **y** salida rápidas; un **vuelo real** solo una de las dos.
- **2.3b** En esa lista no hay ni un flamenco, aunque FLA03 hace el viaje más largo de todos. Míralo aparte:

  ```sql
  SELECT id_ave, epoch_ms(timestamp)::VARCHAR AS t_utc,
         round(dist_prev_m / 1000, 1)         AS km_desde_anterior,
         round(3.6 * dist_prev_m / dt_prev_s) AS v_llegada_kmh,
         round(3.6 * dist_next_m / dt_next_s) AS v_salida_kmh,
         hdop, n_sat
  FROM pasos
  WHERE id_ave = 'FLA03' AND dist_prev_m > 50000
  ORDER BY timestamp
  ```

  Salen tres filas: dos el 27 de noviembre y una el 18 de marzo. ¿Cuál es un error y cuál un vuelo real? ¿Por qué el HDOP no ayuda aquí? Y sobre todo: **¿por qué ninguna de las tres aparecía en la lista de los 30 más rápidos?** Compara el intervalo mediano de muestreo de la tabla del 1.3.
- **2.4** Define una regla de **pico** (llegada y salida rápidas a la vez) y otra de **mala geometría** (`hdop`, `n_sat`). ¿Cuántos fixes marca cada una? Crea con ellas dos vistas: `gps_marcado`, con dos columnas booleanas, y `gps_limpio`, sin los sospechosos.
- **2.5** Ejecuta una consulta que devuelva **solo los fixes descartados con su `geom`** y pulsa **Añadir como capa**. ¿Dónde caen? ¿Se te ha escapado alguno que se vea claramente fuera de sitio?

> **Para discutir:** ¿un umbral fijo de velocidad sirve para las tres especies? ¿Borrarías los fixes o los marcarías? ¿Qué pasa con el error pequeño (3-10 km) y HDOP bueno?

---

## Bloque 3 · ¿Estaba el ave en el agua? (30 min)

Una unión espacial normal cruzaría cada fix con las 26 inundaciones a la vez. Hay que añadir la **condición temporal**: comparar cada fix con la máscara vigente **en su fecha**.

### 3.1 Unión con `ASOF JOIN`

`ASOF JOIN` une cada fila con la fila **anterior más próxima** de otra tabla. Es perfecto para series con muestreos distintos: fixes cada hora, máscaras cada 8-16 días.

```sql
CREATE OR REPLACE VIEW fix_agua AS
WITH fechas AS (SELECT DISTINCT fecha_ini FROM inundacion_2019_2020),
fix AS (
  SELECT g.*, f.fecha_ini AS fecha_mascara
  FROM gps_limpio g
  ASOF JOIN fechas f ON g.timestamp >= f.fecha_ini
)
SELECT fix.* EXCLUDE (geom), fix.geom,
       bool_or(i.id IS NOT NULL) AS en_agua
FROM fix
LEFT JOIN inundacion_2019_2020 i
       ON i.fecha_ini = fix.fecha_mascara
      AND ST_Intersects(fix.geom, i.geom)
GROUP BY ALL
```

> Si no has creado `gps_limpio` en el bloque 2, cambia `gps_limpio` por `gps_aves_2019_2020`.

**Preguntas**

- **3.1** Calcula el **% de fixes en agua por especie y mes**. Usa `strftime(epoch_ms(fecha_hora_local), '%Y-%m')` para el mes. Exporta el resultado a CSV y haz un gráfico.
- **3.2** ¿Qué especie usa menos el agua en marzo-mayo? ¿Por qué? Pista: ¿dónde pasa la mitad del tiempo?
- **3.3** Reescribe la unión **sin** `ASOF JOIN`, usando el periodo de validez: `g.timestamp BETWEEN i.fecha_ini AND i.fecha_fin + 86399999`. ¿Por qué hay que sumar 86.399.999 ms a `fecha_fin`? ¿Salen los mismos porcentajes? ¿Cuál tarda más?
- **3.4** Repite la pregunta 3.1 para los ánsares, pero agrupando por **hora local**: `hour(epoch_ms(fecha_hora_local))`. ¿Qué te dice el resultado sobre la media mensual del 3.1?
- **3.5** Cruza también con `recintos_marisma`. ¿En qué recintos está cada especie en cada estación? ¿Qué porcentaje de fixes cae fuera de los recintos y dónde?

---

## Bloque 4 · Patrones en el tiempo (30 min)

Elige **al menos dos** de las tres.

### 4.1 Paradas: colonia y dormideros

> La herramienta admite **como máximo 5.000 puntos**. Una sola espátula tiene unos 4.600 fixes, así que se trabaja **con un ave cada vez**.

1. En el Espacio de trabajo SQL, crea la capa de un ave y pulsa **Añadir como capa**:
   ```sql
   SELECT id_fix, id_ave, nombre_comun, timestamp,
          epoch_ms(fecha_hora_local)::VARCHAR AS hora_local, geom
   FROM gps_limpio
   WHERE id_ave = 'ESP01'
   ```
2. *Procesamiento → Caja de herramientas de GeoLibre → Vectorial → Movimiento y tiempo → **Detectar paradas*** sobre esa capa (se llama *SQL result* y la hora; puedes renombrarla).
   - Campo de tiempo: `timestamp`. Identificador: `id_ave`.
   - Distancia máxima: 300 m.
   - Duración mínima: **10800 s** (3 h; la herramienta pide segundos).

- **4.1** ¿Dónde está la colonia? ¿Cuándo empiezan y acaban las paradas largas en ella?
- **4.2** Repite con un ánsar (`ANS01`). ¿Las paradas nocturnas caen en agua? ¿Y las diurnas?
- **4.3** Cambia a 1.000 m y 43200 s (12 h). ¿Qué paradas desaparecen? ¿Qué parámetros justificarías para cada especie?

### 4.2 Encuentros

> Esta herramienta compara todos los pares de puntos y admite **como máximo 2 millones de pares**. Con los cuatro ánsares completos no cabe: se trabaja con **una semana**.

1. Crea la capa y pulsa **Añadir como capa**:
   ```sql
   SELECT id_fix, id_ave, timestamp, geom
   FROM gps_limpio
   WHERE nombre_comun = 'Ánsar común'
     AND timestamp BETWEEN epoch_ms(TIMESTAMP '2019-12-01') AND epoch_ms(TIMESTAMP '2019-12-08')
   ```
2. *Vectorial → Movimiento y tiempo → **Proximidad espacio-temporal*** sobre esa capa: campo de tiempo `timestamp`, identificador `id_ave`, 200 m y 30 minutos.

- **4.4** ¿Qué pares de aves se encuentran esa semana? ¿Cuántas veces? ¿Es un encuentro o una pareja que vuela junta?
- **4.5** *(SQL, avanzado)* Calcula para cada par de ánsares, **en toda la temporada**, el % de horas en que están a menos de 200 m. En SQL no hay límite de pares. Pista: une la tabla consigo misma con `round(timestamp / 3600000)` como clave para no comparar todos los fixes con todos.

### 4.3 Área de campeo como serie temporal

```sql
SELECT id_ave, nombre_comun,
       strftime(epoch_ms(fecha_hora_local), '%Y-%m') AS mes,
       count(*) AS n_fixes,
       round(ST_Area(ST_Transform(ST_ConvexHull(ST_Collect(list(geom))),
             'EPSG:4326', 'EPSG:25829', always_xy := true)) / 1e4) AS area_ha,
       ST_ConvexHull(ST_Collect(list(geom))) AS geom
FROM gps_limpio
GROUP BY ALL
HAVING count(*) >= 100
```

**Añadir como capa** y actívala en el Control de tiempo si quieres animarla.

- **4.6** ¿En qué meses es mayor el área de cada especie? ¿Tiene sentido biológico o es un artefacto?
- **4.7** Repite con `gps_aves_2019_2020` (sin limpiar). ¿Cuánto cambia el área por culpa de unos pocos errores?
- **4.8** ¿Qué alternativa al polígono mínimo convexo usarías? *Estadísticas espaciales* ofrece densidad de núcleo y *Emerging Hot Spot*. ¿Qué pregunta temporal respondería este último?

---

## Puesta en común (10 min)

Cada grupo responde en **tres frases**:

1. Qué hace cada especie con la marisma a lo largo del ciclo.
2. Qué decisión de limpieza o de parámetros ha cambiado más vuestros resultados.
3. Qué no se puede saber con estos datos: frecuencia de muestreo, fechas de las máscaras, relleno de nubes…

**Entrega:** el proyecto de GeoLibre guardado, el CSV de la pregunta 3.1 y las tres frases.
