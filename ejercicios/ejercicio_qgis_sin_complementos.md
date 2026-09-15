# Plan C · QGIS sin complementos, con datos ligeros

Úsalo cuando **no se puedan instalar complementos** (Trajectools, MovingPandas…) o cuando los equipos vayan justos de memoria.

Solo usa herramientas nativas de QGIS 3.x y el GeoPackage ligero `datos/ligero/donana_ligero.gpkg` (EPSG:25829, en metros):

| Capa | Contenido |
|---|---|
| `gps_aves_ligero` | 6 aves, un fix cada 2 h: FLA01, FLA03 (viaje a Fuente de Piedra), ANS01 + ANS02 (pareja), ESP01, ESP02 (colonia) |
| `inundacion_ligero` | Manchas de agua ≥ 5 ha de las 26 fechas, con `fecha_ini` y `fecha_fin` |
| `recintos_marisma`, `balsas_veta_la_palma` | Cartografía de referencia |

**Versión de QGIS:**
- **3.14 o superior:** Controlador temporal disponible.
- **Anterior a 3.14:** usa el complemento *TimeManager* si está instalado, o sáltate la animación.
- **SQL con funciones de ventana** (`LAG`) en capas virtuales: depende de la versión de SQLite que traiga la instalación. En versiones anteriores a 3.16, pruébalo antes; si falla, usa la alternativa del bloque 2.

---

## Bloque 1 · Explorar el tiempo (20 min)

1. Carga las cuatro capas del GeoPackage. Simboliza los puntos por `nombre_comun`.
2. **Controlador temporal** (QGIS ≥ 3.14):
   - **Puntos:** *Propiedades → Temporal → Campo único con fecha/hora* = `fecha_hora`. Duración del evento: 2 horas.
   - **Inundación:** *Campos separados para inicio y fin* = `fecha_ini` / `fecha_fin`.
   - En el panel del reloj, paso de **1 día**. Reprodúcelo.
3. Mientras avanza la animación:
   - **1.1** ¿Cuándo llegan y se van ánsares y espátulas?
   - **1.2** ¿Cuándo se inunda la marisma?
   - **1.3** ¿Ves puntos imposibles, lejos de Doñana?

> **Si el equipo va lento:** desactiva el mapa base, reduce la animación a un mes (*Rango personalizado*) y apaga la capa de inundación mientras animas los puntos.

---

## Bloque 2 · Trayectorias sin complementos (30 min)

### 2.1 Las líneas

*Caja de herramientas → Creación de vectores → **Puntos a ruta***

- Entrada: `gps_aves_ligero`.
- Expresión de orden: `"fecha_hora"`.
- Campo de agrupamiento: `id_ave`.

Obtienes una línea por ave. **2.1** ¿Qué trayectoria delata los errores de posición a simple vista?

### 2.2 Las métricas de cada paso: capa virtual con SQL

*Capa → Añadir capa → **Añadir/Editar capa virtual…*** → *Importar* `gps_aves_ligero` → pega la consulta:

```sql
SELECT id_fix, id_ave, nombre_comun, fecha_hora, hdop, n_sat,
       ST_Distance(geometry, LAG(geometry) OVER w)                                 AS dist_prev_m,
       (julianday(fecha_hora) - julianday(LAG(fecha_hora) OVER w)) * 86400         AS dt_prev_s,
       ST_Distance(geometry, LEAD(geometry) OVER w)                                AS dist_next_m,
       (julianday(LEAD(fecha_hora) OVER w) - julianday(fecha_hora)) * 86400        AS dt_next_s,
       geometry
FROM gps_aves_ligero
WINDOW w AS (PARTITION BY id_ave ORDER BY fecha_hora)
```

`LAG` toma el fix **anterior** del mismo ave y `LEAD` el **siguiente**. Como la capa está en EPSG:25829, la distancia sale en metros.

> **Si da error de sintaxis** (SQLite antiguo, sin funciones de ventana): usa *Caja de herramientas → Análisis vectorial → **Distancia al vecino más próximo (de centro a centro)*** o simplemente la herramienta *Puntos a ruta* + *Explotar líneas* + `$length` en la calculadora.

Con la capa virtual cargada, abre la **calculadora de campos**. Mejor aún, trabaja sobre una copia: *Exportar → Guardar objetos como…* en GeoPackage, porque las capas virtuales se recalculan en cada refresco y en un equipo lento se nota.

- **2.2** Campo `v_kmh` = `3.6 * "dist_prev_m" / "dt_prev_s"`. ¿Cuántos pasos superan 80 km/h? Usa *Seleccionar por expresión*.
- **2.3** Campo `pico` = `3.6*"dist_prev_m"/"dt_prev_s" > 60 AND 3.6*"dist_next_m"/"dt_next_s" > 60`. ¿Cuántos picos hay? ¿Coinciden con HDOP alto?
- **2.4** Crea `gps_limpio` con *Extraer por expresión*: `NOT coalesce("pico", false) AND "hdop" <= 5`. ¿Qué vuelos reales sobreviven al filtro?

---

## Bloque 3 · ¿Estaba el ave en el agua? (30 min)

1. **Vectorial → Herramientas de gestión de datos → Unir atributos por localización**
   - Entrada: `gps_limpio`.
   - Unir con: `inundacion_ligero` (*intersecan*).
   - Campos: `fecha_ini`, `fecha_fin`.
   - Tipo: **uno a muchos**.
   - *Descartar registros que no se pudieron unir*: **desactivado**.
2. **Calculadora de campos** sobre el resultado, campo entero `en_agua`:

   ```
   if(to_date("fecha_hora") >= "fecha_ini" AND to_date("fecha_hora") <= "fecha_fin", 1, 0)
   ```

3. Un fix aparece una vez por cada mancha de agua que toca **en cualquier fecha**. Para quedarte con un valor por fix:
   - *Análisis vectorial → **Estadísticas por categorías***, categoría `id_fix`, campo `en_agua`.
   - El **máximo** dice si ese fix estaba en agua en su fecha.
4. Une ese máximo de vuelta a los puntos: *Propiedades → Uniones* por `id_fix`. Crea `mes` = `format_date("fecha_hora", 'yyyy-MM')` y repite *Estadísticas por categorías* con `nombre_comun` y `mes`.

- **3.1** ¿Qué % de fixes está en agua por especie y mes? Haz la tabla o el gráfico con *Plotly*, si está, o exporta a CSV.
- **3.2** ¿Qué especie usa menos el agua en primavera? ¿Por qué?
- **3.3** Ánsares por **hora local**: `format_date("fecha_hora" + make_interval(hours:=1), 'H')` en invierno. ¿Día frente a noche?

---

## Bloque 4 · Patrones (30 min · elige dos)

### 4.1 Tiempo de residencia (colonia y dormideros)

1. *Extraer por expresión*: `"nombre_comun" = 'Espátula común' AND "fecha_hora" >= '2020-03-01' AND "fecha_hora" < '2020-06-20'`.
2. *Creación de vectores → **Crear cuadrícula***: hexágonos de 500 m sobre la extensión de los recintos.
3. *Análisis vectorial → **Contar puntos en polígono***. Cada punto son 2 horas, así que `NUMPOINTS * 2` son horas de residencia.
4. **4.1** ¿Dónde está la colonia? ¿Cuántas horas acumula esa celda?

### 4.2 Encuentros de la pareja

1. Separa ANS01 y ANS02 con *Extraer por expresión*.
2. *Gestión de datos → **Unir atributos por el más cercano***: entrada ANS01, unir ANS02, máximo 200 m, campos `fecha_hora`.
3. Campo `dt_min` = `abs(epoch("fecha_hora") - epoch("n_fecha_hora")) / 60000`.
4. **4.2** ¿Qué % de fixes de ANS01 tiene a ANS02 a menos de 200 m **y** 30 min?

### 4.3 Área de campeo por mes

1. Campo `id_mes` = `"id_ave" || '_' || format_date("fecha_hora", 'yyyy-MM')`.
2. *Geoproceso → **Geometría mínima envolvente***, campo `id_mes`, tipo *Envolvente convexa*.
3. **4.3** Área en ha (`$area / 10000`) por mes para ESP01 y ESP02. ¿Qué meses se disparan y por qué? Repite con los puntos sin limpiar.

---

## Entrega

Proyecto QGIS guardado, tabla del bloque 3 y tres frases: **¿cómo usan tres especies una marisma que se llena y se vacía?**
