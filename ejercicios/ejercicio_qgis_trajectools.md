# Plan B · El mismo ejercicio en QGIS + Trajectools

Mismas preguntas que en `ejercicio_geolibre.md`, resueltas con **QGIS 3.40** (Temporal Controller, nativo) y el complemento **Trajectools 2.7** (MovingPandas dentro de Processing).

Usa el GeoPackage `datos/donana_tiempo_vectorial.gpkg`: está en **EPSG:25829** (metros) y el campo `fecha_hora` ya es de tipo fecha/hora.

---

## 0 · Preparación (hacerlo antes de clase)

1. **Instala MovingPandas ≥ 0.22.3** en el Python de QGIS. Trajectools 2.7 no arranca con versiones anteriores.
   - **Windows (OSGeo4W / instalador de QGIS):** instala antes el complemento **QPIP**. Al activar Trajectools, QPIP ofrece instalar las dependencias. También puedes abrir la *OSGeo4W Shell* y ejecutar `python -m pip install -U movingpandas`.
   - **Linux (QGIS de sistema):** `python3 -m pip install --user -U movingpandas`.
   - Comprueba desde la consola Python de QGIS: `import movingpandas; movingpandas.__version__`.
2. Instala **Trajectools** desde *Complementos → Administrar e instalar complementos*.
3. Tras instalarlo, en la *Caja de herramientas de Processing* aparece el grupo **Trajectools**.

---

## Bloque 1 · Explorar el tiempo (20 min)

1. Carga del GeoPackage las capas `gps_aves_2019_2020`, `inundacion_2019_2020`, `recintos_marisma` y `balsas_veta_la_palma`.
2. Simboliza `gps_aves_2019_2020` por categorías con `nombre_comun`.
3. **Temporal de los puntos:** *Propiedades de la capa → Temporal*.
   - Configuración: **Campo único con fecha/hora** → `fecha_hora`.
   - Duración del evento: **2 horas** (para que los flamencos, que fijan cada 2 h, no parpadeen).
4. **Temporal de la inundación:** *Propiedades → Temporal*.
   - Configuración: **Campos separados para inicio y fin** → `fecha_ini` y `fecha_fin`.
   - **Atención:** `fecha_fin` es el último día válido, incluido. Si el polígono desaparece un día antes de tiempo, activa *Incluir fin* o suma un día con una expresión.
5. Abre el **Controlador temporal** (icono del reloj):
   - Modo **Animación**, paso de **1 día**, rango 2019-09-01 → 2020-08-31.
   - Reprodúcelo.

**Preguntas**

- 1.1 ¿Cuándo llegan los ánsares y cuándo se van? ¿Y las espátulas?
- 1.2 ¿En qué semana se inunda la marisma? ¿Qué hacen los flamencos antes y después?
- 1.3 ¿Ves puntos imposibles, en el mar o a 200 km de Doñana? Anota tres ejemplos (id_ave y fecha).

---

## Bloque 2 · Trayectorias y limpieza (30 min)

1. **Trajectools → Basic → Create trajectories**
   - Capa de entrada: `gps_aves_2019_2020`.
   - Campo identificador: `id_ave`.
   - Campo de tiempo: `fecha_hora`.
   - Genera dos salidas: los **puntos con métricas** (velocidad, dirección…) y las **líneas de trayectoria**.
2. En la tabla de atributos de los puntos, ordena por velocidad descendente.
   - 2.1 ¿Cuántos puntos superan los 80 km/h?
   - 2.2 Mira en el mapa los cinco más rápidos. ¿Son errores o vuelos reales? Pista: FLA03 del 18 de marzo, llegadas de ánsares a finales de octubre.
3. **Trajectools → Trajectory cleaning → Remove speed above threshold**
   - Umbral: 80 km/h.
   - 2.3 Compara el número de puntos antes y después. ¿Se ha llevado algún vuelo real por delante?
4. **Filtro complementario por calidad:** *Seleccionar por expresión* `"hdop" > 5 OR "n_sat" < 5`.
   - 2.4 ¿Cuántos puntos son y dónde caen?
5. **Trajectools → Trajectory splitting → Split trajectories at observation gaps**
   - Hueco: 12 horas.
   - 2.5 ¿Qué ave se parte en dos y en qué fechas? ¿Qué le ha pasado?

---

## Bloque 3 · ¿Estaba el ave en el agua? Unión espacio-temporal (30 min)

Una unión espacial normal cruzaría cada fix con las 26 fechas de inundación. Hay que añadir la condición temporal. Haz la versión en dos pasos:

1. **Vectorial → Herramientas de gestión de datos → Unir atributos por localización**
   - Entidades a unir: los puntos (limpios).
   - Comparación: *intersecan* con `inundacion_2019_2020`.
   - Campos a añadir: `fecha_ini` y `fecha_fin`.
   - Tipo de unión: **uno a muchos**.
   - Marca **Descartar registros que no se pudieron unir**: desactivado.
2. Sobre la capa resultante, **Calculadora de campos** → nuevo campo entero `en_agua`:

   ```
   if( to_date("fecha_hora") >= "fecha_ini" AND to_date("fecha_hora") <= "fecha_fin", 1, 0)
   ```

3. Cada fix aparece ahora una vez por polígono de agua que toca en cualquier fecha. Hay que quedarse con **un valor por fix**: 1 si alguna fila es 1, 0 si no.
   - **Vectorial → Herramientas de análisis → Estadísticas por categorías**, categoría `id_fix`, campo `en_agua`.
   - El máximo es `en_agua` por fix.
4. Resume por especie y mes. Crea el campo `mes` con `format_date("fecha_hora", 'yyyy-MM')` y vuelve a usar *Estadísticas por categorías* con categorías `nombre_comun` y `mes`.
   - 3.1 Construye la tabla de % de fixes en agua por especie y mes.
   - 3.2 ¿Qué especie usa menos el agua en abril? ¿Por qué? Pista: mira dónde duerme.
   - 3.3 Repite para los ánsares agrupando por **hora local**: `format_date("fecha_hora" + to_interval('1 hour'), 'H')` en invierno. ¿Qué cambia entre el día y la noche?

> **Alternativa en una sola consulta:** *Capa → Crear capa → Nueva capa virtual* con SQL (SpatiaLite):
>
> ```sql
> SELECT g.id_fix, g.nombre_comun, strftime('%Y-%m', g.fecha_hora) AS mes,
>        MAX(CASE WHEN i.id IS NULL THEN 0 ELSE 1 END) AS en_agua
> FROM gps_aves_2019_2020 g
> LEFT JOIN inundacion_2019_2020 i
>   ON date(g.fecha_hora) BETWEEN i.fecha_ini AND i.fecha_fin
>  AND ST_Intersects(g.geometry, i.geometry)
> GROUP BY g.id_fix
> ```
>
> Con 45.000 puntos puede tardar varios minutos. Prueba primero filtrando un mes.

---

## Bloque 4 · Paradas, encuentros y áreas por mes (30 min)

1. **Trajectools → Event extraction → Extract stop points**
   - Duración mínima: 3 h.
   - Diámetro máximo: 300 m.
   - 4.1 Superpón las paradas de las espátulas entre marzo y junio. ¿Dónde está la colonia?
   - 4.2 Repite con los ánsares y separa paradas nocturnas y diurnas. ¿Coinciden con agua o con arrozal?
2. **Encuentros ANS01–ANS02**
   - Filtra cada ave en su propia capa.
   - *Unir atributos por el más cercano* con distancia máxima de 200 m.
   - Calcula `abs(epoch("fecha_hora") - epoch("n_fecha_hora")) / 60` y quédate con los menores de 30 min.
   - 4.3 ¿Qué porcentaje del tiempo están juntos? ¿Y ANS01 con ANS03?
3. **Áreas de campeo por mes**
   - Crea el campo `id_mes`: `"id_ave" || '_' || format_date("fecha_hora", 'yyyy-MM')`.
   - *Vectorial → Herramientas de geoproceso → Geometría mínima envolvente*, campo `id_mes`, tipo **Envolvente convexa**.
   - 4.4 Grafica el área (`$area / 1e4`, en ha) por mes para las cuatro espátulas. ¿Cuándo se encoge? ¿Por qué? ¿Qué problema tiene el polígono convexo con los fixes erróneos?

---

## Bloque 5 (opcional) · Contar la historia con Animation Workbench

El **Controlador temporal** anima el *tiempo*: filtra entidades por fecha. **Animation Workbench** (Kartoza, v1.4) anima la *cámara* y la *simbología*: recorre las entidades de una capa una tras otra, se detiene en cada una (*hover*) y vuela a la siguiente (*travel*). Exporta GIF o MP4.

**Preparación**
- Instálalo desde *Complementos → Administrar e instalar complementos → Animation Workbench*.
- Para exportar MP4 necesitas **ffmpeg** en el sistema.

### 5.1 Una posición por día del viaje de FLA03

El plugin recorre las entidades en el orden de la capa. Creamos una capa con un punto por día, ordenada por fecha: *Capa → Crear capa → Nueva capa virtual*.

```sql
SELECT date(fecha_hora) AS fecha,
       count(*)         AS n_fixes,
       MakePoint(avg(x_utm), avg(y_utm), 25829) AS geometry
FROM gps_aves_2019_2020
WHERE id_ave = 'FLA03' AND hdop < 5
  AND fecha_hora BETWEEN '2020-03-10' AND '2020-06-15'
GROUP BY date(fecha_hora)
ORDER BY fecha
```

Guárdala como GeoPackage (*Exportar → Guardar objetos como…*) para que los identificadores queden en orden cronológico.

### 5.2 Configurar la animación

1. Abre **Animation Workbench** desde la barra de herramientas.
2. Modo **Planar**. Capa: la de posiciones diarias.
3. Pon duraciones cortas: *hover* 0,3 s y *travel* 0,5 s, a 12-24 fotogramas por segundo.
4. Simbología que reacciona a la animación. En *Tamaño* del símbolo, *Definido por datos*:

   ```
   if($id = @hover_feature_id, 7, 2)
   ```

5. Etiqueta o decoración de título con la fecha del punto activo:

   ```
   format_date(to_date(attribute(@hover_feature, 'fecha')), 'd MMM yyyy')
   ```

6. Prueba con la vista previa y exporta a GIF o MP4.

**Preguntas**

- **5.1** ¿Qué transmite esta animación que no transmiten el mapa estático ni el Controlador temporal?
- **5.2** ¿Qué información se pierde al resumir en una posición diaria? Piensa en las idas y vueltas del día a la noche.
- **5.3** Combina ambas herramientas: deja la capa `inundacion_2019_2020` con el Controlador temporal activo. ¿Se actualiza la inundación mientras la cámara recorre los días? Compruébalo y explica por qué ocurre o por qué no.

---

## Entrega

- Un proyecto QGIS con las capas resultado.
- Una tabla o gráfico con el % de fixes en agua por especie y mes.
- Tres frases que respondan a la pregunta: **¿cómo usan tres especies una marisma que se llena y se vacía?**
