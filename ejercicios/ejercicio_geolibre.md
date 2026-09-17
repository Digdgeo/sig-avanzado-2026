# Ejercicio · La marisma en el tiempo con GeoLibre

**Pregunta:** ¿cómo usan tres especies de aves una marisma que se llena y se vacía?

**Duración:** unas 2 horas · **Herramienta:** [GeoLibre](https://web.geolibre.app) en el navegador, sin instalar nada · **Datos:** carpeta `datos/geoparquet/`

> Los datos GPS son **simulados**: sirven para aprender el método, no para sacar conclusiones sobre estas especies. La inundación sí es real (Landsat, Protocolo v2, ciclo 2019-2020).

> **Cómo usar este guion.** Todas las consultas están escritas: se copian y se ejecutan. Vamos justos de tiempo, así que **el trabajo no es teclear SQL, es leer lo que hace cada consulta y responder a las preguntas numeradas**. Donde pone *"Debe salir"* tienes el resultado esperado, para saber si vas bien sin esperar a nadie.

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

Ahora, sobre la vista `pasos`:

### 2.2 Velocidades por especie

```sql
SELECT nombre_comun,
       round(median(3.6 * dist_prev_m / dt_prev_s), 2)              AS v_mediana_kmh,
       round(quantile_cont(3.6 * dist_prev_m / dt_prev_s, 0.99), 1) AS v_p99_kmh,
       count(*) FILTER (WHERE 3.6 * dist_prev_m / dt_prev_s > 80)   AS n_mas_80kmh
FROM pasos
GROUP BY ALL
```

**Debe salir:** espátula 0,13 · 10,5 · 32 — flamenco 0,11 · 6,2 · 6 — ánsar 0,12 · 27,9 · 23.

- **2.2** La mediana es casi cero en las tres especies. ¿Por qué? ¿Sirve de algo para detectar errores?
- **2.2b** El flamenco tiene 6 pasos rápidos y la espátula 32. ¿Vuela menos el flamenco, o hay otra explicación? Mira su intervalo mediano en la tabla del 1.3.

### 2.3 Los pasos más rápidos: ¿error o vuelo real?

```sql
SELECT id_ave, epoch_ms(timestamp)::VARCHAR AS t_utc,
       round(dist_prev_m / 1000, 1)         AS km_desde_anterior,
       round(3.6 * dist_prev_m / dt_prev_s) AS v_llegada_kmh,
       round(3.6 * dist_next_m / dt_next_s) AS v_salida_kmh,
       hdop, n_sat
FROM pasos
WHERE 3.6 * dist_prev_m / dt_prev_s > 80
ORDER BY v_llegada_kmh DESC
LIMIT 40
```

- **2.3** Clasifica a mano cinco filas como *error* o *vuelo real*. La regla: un **pico** tiene llegada **y** salida rápidas (el ave salta fuera y vuelve); un **vuelo real** solo tiene una de las dos. Fíjate en que los errores aparecen **por parejas de filas consecutivas**: la posición errónea y el fix siguiente, que es inocente.
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

### 2.4 Marcar los sospechosos y crear la vista limpia

Tres sentencias, **una ejecución cada una**:

```sql
CREATE OR REPLACE VIEW gps_marcado AS
SELECT *,
       coalesce(3.6 * dist_prev_m / dt_prev_s > 60 AND 3.6 * dist_next_m / dt_next_s > 60, false) AS pico,
       hdop > 5 OR n_sat < 5 AS mala_geometria
FROM pasos
```

```sql
CREATE OR REPLACE VIEW gps_limpio AS
SELECT * EXCLUDE (pico, mala_geometria)
FROM gps_marcado
WHERE NOT pico AND NOT mala_geometria
```

```sql
SELECT count(*) FILTER (WHERE pico)                   AS n_picos,
       count(*) FILTER (WHERE mala_geometria)         AS n_mala_geometria,
       count(*) FILTER (WHERE pico OR mala_geometria) AS n_descartados,
       count(*)                                       AS n_total
FROM gps_marcado
```

> `CREATE VIEW` no devuelve filas: la tabla de resultados sale vacía y **eso es correcto**. Las vistas persisten durante la sesión, pero se pierden si recargas la página.

- **2.4** Hay **68 errores simulados**. La regla del pico marca **33**, y los 33 son errores de verdad. El HDOP > 5 pilla **49**. ¿Cuántos quedan sin detectar, y por qué ninguna regla sencilla los encuentra?

### 2.5 Ver los descartados en el mapa

```sql
SELECT id_fix, id_ave, epoch_ms(timestamp)::VARCHAR AS t_utc,
       hdop, n_sat, pico, mala_geometria, geom
FROM gps_marcado
WHERE pico OR mala_geometria
```

- **2.5** Pulsa **Añadir como capa**. ¿Dónde caen? ¿Se te ha escapado alguno que se vea claramente fuera de sitio?

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

### 3.2 El porcentaje en agua, por especie y mes

```sql
SELECT nombre_comun,
       strftime(epoch_ms(fecha_hora_local), '%Y-%m') AS mes,
       round(100 * avg(en_agua::INT), 1)             AS pct_en_agua,
       count(*)                                      AS n
FROM fix_agua
GROUP BY ALL
ORDER BY nombre_comun, mes
```

**Debe salir:** espátula 84 % en febrero, 19-22 % de marzo a mayo, 44 % en junio, 80-86 % en julio y agosto · flamenco 77-83 % de septiembre a febrero, 59 % en abril y mayo, 79-81 % de junio a agosto · ánsar 38 % en octubre y 51-55 % de noviembre a febrero.

- **3.1** Exporta el resultado a CSV y haz un gráfico.
- **3.2** ¿Qué especie usa menos el agua en marzo-mayo? ¿Por qué? Pista: ¿dónde pasa la mitad del tiempo?

### 3.3 La misma unión sin `ASOF JOIN`

```sql
SELECT g.nombre_comun,
       round(100.0 * count(DISTINCT CASE WHEN i.id IS NOT NULL THEN g.id_fix END)
             / count(DISTINCT g.id_fix), 1) AS pct_en_agua
FROM gps_limpio g
LEFT JOIN inundacion_2019_2020 i
       ON g.timestamp BETWEEN i.fecha_ini AND i.fecha_fin + 86399999
      AND ST_Intersects(g.geom, i.geom)
GROUP BY ALL
```

- **3.3** ¿Por qué hay que sumar 86.399.999 ms a `fecha_fin`? ¿Salen los mismos porcentajes que con `ASOF JOIN`? ¿Cuál tarda más?

### 3.4 El ritmo diario de los ánsares

```sql
SELECT hour(epoch_ms(fecha_hora_local))  AS hora_local,
       round(100 * avg(en_agua::INT), 1) AS pct_en_agua,
       count(*)                          AS n
FROM fix_agua
WHERE nombre_comun = 'Ánsar común'
GROUP BY ALL
ORDER BY hora_local
```

**Debe salir:** cerca del **90 % de noche** y casi **0 % entre las 8 y las 17 h**.

- **3.4** El 3.2 daba un 51-55 % de media mensual para el ánsar. A la vista de esto, ¿qué estaba escondiendo esa media? Duermen en el agua de la marisma y pastan de día en los arrozales y la vera, fuera de los polígonos de agua.

### 3.5 ¿En qué recinto está cada especie?

```sql
SELECT f.nombre_comun,
       coalesce(r.recinto, 'fuera de los recintos') AS recinto,
       CASE WHEN month(epoch_ms(f.fecha_hora_local)) IN (9,10,11) THEN '1 otoño'
            WHEN month(epoch_ms(f.fecha_hora_local)) IN (12,1,2)  THEN '2 invierno'
            WHEN month(epoch_ms(f.fecha_hora_local)) IN (3,4,5)   THEN '3 primavera'
            ELSE '4 verano' END AS estacion,
       count(*) AS n_fixes
FROM fix_agua f
LEFT JOIN recintos_marisma r ON ST_Intersects(f.geom, r.geom)
GROUP BY ALL
ORDER BY 1, 3, 4 DESC
```

- **3.5** ¿Qué porcentaje de fixes cae fuera de los recintos, y dónde?

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
- **4.5** *(SQL, avanzado)* Lo mismo **en toda la temporada**, que en SQL no tiene límite de pares. La clave está en unir la tabla consigo misma por la hora redondeada, para no comparar todos los fixes con todos:

  ```sql
  WITH u AS (
    SELECT id_ave, timestamp, round(timestamp / 3600000) AS hora,
           ST_Transform(geom, 'EPSG:4326', 'EPSG:25829', always_xy := true) AS g
    FROM gps_limpio
    WHERE nombre_comun = 'Ánsar común'
  ),
  pares AS (
    SELECT a.id_ave AS ave_a, b.id_ave AS ave_b, a.timestamp
    FROM u a JOIN u b
      ON a.hora = b.hora AND a.id_ave < b.id_ave
     AND abs(a.timestamp - b.timestamp) <= 30 * 60 * 1000
     AND ST_DWithin(a.g, b.g, 200)
  ),
  coinciden AS (
    SELECT a.id_ave AS ave_a, b.id_ave AS ave_b, count(*) AS n_horas_comunes
    FROM u a JOIN u b ON a.hora = b.hora AND a.id_ave < b.id_ave
    GROUP BY ALL
  )
  SELECT c.ave_a, c.ave_b, c.n_horas_comunes,
         count(p.timestamp) AS n_encuentros,
         round(100.0 * count(p.timestamp) / c.n_horas_comunes, 1) AS pct_tiempo_juntos
  FROM coinciden c
  LEFT JOIN pares p ON p.ave_a = c.ave_a AND p.ave_b = c.ave_b
  GROUP BY c.ave_a, c.ave_b, c.n_horas_comunes
  ORDER BY pct_tiempo_juntos DESC
  ```

  **Debe salir:** ANS01–ANS02 cerca del **99,9 %** —son pareja— y el resto de combinaciones en torno al 0 %.

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
