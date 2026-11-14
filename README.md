# Proyecto Final Big Data: siniestralidad vial en Bogotá

Proyecto integrador del curso CADI (Especialización en Analítica y Ciencia de Datos). Recorre un caso aplicado de punta a punta: recolección, almacenamiento, procesamiento y análisis de datos con Apache Hadoop, Apache Spark, CockroachDB (NewSQL) y MongoDB Atlas (NoSQL).

Autor: Jampier Santiago Moreno Arenas

## Pregunta de trabajo

¿Dónde, cuándo y en qué condiciones se concentran los siniestros viales graves en Bogotá, y qué se puede decir del riesgo que implican? El enfoque es de análisis de riesgo vial, útil como insumo para seguros de movilidad. Los datos no traen primas ni montos asegurados, así que no se hace tarificación.

## Datos

- **Fuente:** Histórico de Siniestros Bogotá D.C., Datos Abiertos Bogotá (Secretaría Distrital de Movilidad). Licencia Creative Commons Attribution 4.0.
- **Contenido:** 199.146 siniestros georreferenciados entre 2015 y 2021, en un CSV de 38 MB y 16 columnas.
- **Obtención:** El CSV esta en la carpeta data del repositorio.
- **Vigencia:** el portal indica como última actualización de los datos el 22 de octubre de 2021, por lo que 2021 no es un año completo y no se compara año contra año.

## Arquitectura

```
CSV (portal)
    |
    v
HDFS (capa cruda) ---> Hadoop MapReduce (conteos por localidad, año y hora)
    |
    v
Spark: limpieza, variables derivadas, modelo dimensional
    |
    v
Parquet (capa limpia)
    |-------------------------> Spark: análisis, modelo de severidad, K-Means
    |
    +--> CockroachDB: hechos y dimensiones, consultas SQL
    |
    +--> MongoDB Atlas: documentos con GeoJSON, consultas geoespaciales
                |
                v
        Verificación de consistencia entre Parquet, CockroachDB y MongoDB
```

El diagrama de bloques de la presentación está en `docs/`.

## Cuadernos

Se ejecutan en Google Colab y en este orden. Cada cuaderno corre en su propia sesión, así que se pasan archivos de uno a otro.

| #   | Cuaderno                             | Qué hace                                                                                               | Entradas                   | Salidas                    |
| --- | ------------------------------------ | ------------------------------------------------------------------------------------------------------ | -------------------------- | -------------------------- |
| 1   | `01_recoleccion_y_limpieza.ipynb`    | Verifica el archivo, lo perfila, lo limpia con Spark, arma el modelo dimensional y escribe Parquet     | CSV                        | `interim_parquet.zip`      |
| 2   | `02_hdfs_mapreduce_y_analisis.ipynb` | Sube el CSV a HDFS, corre tres trabajos MapReduce y hace el análisis, el modelo y el K-Means con Spark | CSV, `interim_parquet.zip` | `resultados_cuaderno2.zip` |
| 3   | `03_bases_de_datos.ipynb`            | Carga a CockroachDB y MongoDB Atlas, ejecuta consultas y compara los conteos                           | `interim_parquet.zip`      | `resultados_cuaderno3.zip` |

## Cómo ejecutarlo

1. **Cuaderno 1:** arrastrar el CSV al panel de Archivos de Colab (`/content`) y ejecutar todo. Al final descarga `interim_parquet.zip`.
2. **Cuaderno 2:** arrastrar al panel de Archivos el CSV y `interim_parquet.zip`, y ejecutar todo. La primera vez descarga Hadoop (unos 700 MB), lo que puede tardar varios minutos. Se recomienda subir los archivos por el panel y no con `files.upload()`, que es mucho más lento.
3. **Cuaderno 3:**
   - Crear una base en CockroachDB y un clúster en MongoDB Atlas.
   - En los _Secretos_ de Colab (con acceso al cuaderno activado) guardar `COCKROACH_URL` y `ATLAS_URI`. Opcionalmente, `COCKROACH_CERT_URL` con la URL del certificado del clúster (`https://cockroachlabs.cloud/clusters/<id>/cert`), que el cuaderno descarga solo.
   - En Atlas, _Network Access_ debe permitir la IP de Colab, que cambia en cada sesión. Una opción es `0.0.0.0/0` durante la ejecución, que conviene quitar después.
   - Arrastrar `interim_parquet.zip` al panel de Archivos y ejecutar todo.

El cuaderno 3 borra y recrea el esquema `siniestros_bogota` en CockroachDB y la colección `siniestros` en MongoDB. No toca nada más. Las credenciales nunca se escriben en el código ni se imprimen.

## Resultados

**Calidad de datos.** De 199.146 registros no se eliminó ninguno: no hay duplicados, fechas nulas ni años inconsistentes. Hay 11 coordenadas fuera de Bogotá, que se anulan sin descartar el registro. `LATITUD` y `LONGITUD` venían como texto con puntos de miles mal ubicados (por ejemplo `-74.090.923.953`) y se reconstruyeron. Las columnas `X` e `Y` se descartan: no coinciden en todos los registros con `LATITUD` y `LONGITUD`. Los siniestros sin localidad se agrupan en `SIN DATO`.

**Gravedad.** 64,4 % solo daños (128.207), 34,0 % con heridos (67.700) y 1,6 % con muertos (3.239).

**Tendencia.**

| Año            | Siniestros | % graves |
| -------------- | ---------: | -------: |
| 2015           |     27.885 |     36,6 |
| 2016           |     31.928 |     32,3 |
| 2017           |     32.415 |     31,4 |
| 2018           |     33.418 |     35,7 |
| 2019           |     32.962 |     36,8 |
| 2020           |     22.424 |     39,3 |
| 2021 (parcial) |     18.114 |     40,5 |

En 2020 los siniestros caen 32 % frente a 2019, pero la proporción de graves sube.

**Franjas del día.** La madrugada es la menos frecuente pero la más grave: 42,5 % de graves y 3,79 % con muertos, frente a 31,8 % y 1,14 % en la tarde. La noche también está por encima del promedio (39,7 % y 2,11 %).

**Zonas calientes (K-Means, k = 6, silueta 0,563).** La zona centrada en Ciudad Bolívar tiene el mayor porcentaje de graves (50,4 %); la de Kennedy concentra el mayor número de siniestros con muertos (779). La zona de Chapinero es la menos grave (25,1 %).

**Modelo de severidad** (grave = con heridos o muertos; entrenado con 2015 a 2020 y probado con 2021):

| Modelo              | AUC ROC | AUC PR | Precisión | Recall |
| ------------------- | ------: | -----: | --------: | -----: |
| Regresión logística |   0,725 |  0,711 |     0,728 |  0,432 |
| Random Forest       |   0,728 |  0,714 |     0,916 |  0,307 |

El desempeño es moderado. La clase de accidente se conoce una vez ocurrido el siniestro, así que el modelo sirve para estimar la severidad esperada y entender factores de riesgo, no para anticipar siniestros.

**HDFS y MapReduce.** El CSV queda en HDFS como un archivo de 38,3 MB en un bloque, con replicación 1 y estado `HEALTHY`. Tres trabajos con Hadoop Streaming (mapper y reducer en Python, con combiner):

| Trabajo              | Segundos | Registros de entrada | Salida del combiner |
| -------------------- | -------: | -------------------: | ------------------: |
| Localidad y gravedad |     10,6 |              199.147 |                  63 |
| Año y clase          |      9,7 |              199.147 |                  45 |
| Hora y gravedad      |      8,7 |              199.147 |                  72 |

El combiner reduce 199.146 pares a menos de un centenar antes del reducer. La entrada incluye el encabezado, por eso hay un registro más que siniestros.

**CockroachDB y MongoDB.** Los tres almacenes dan los mismos conteos por gravedad (199.146 en total). Tiempos medidos en Colab, una sola corrida:

| Operación                                       |  Segundos |
| ----------------------------------------------- | --------: |
| Carga de 199.146 filas en CockroachDB           |      25,2 |
| Carga de 199.146 documentos en MongoDB          |     143,3 |
| Consultas SQL analíticas (JOIN, LAG, rank)      | 0,4 a 0,5 |
| Consultas geoespaciales en MongoDB (`$geoNear`) |      0,01 |
| Celdas calientes de ~1 km en MongoDB            |       0,7 |

El plan de ejecución de la consulta filtrada por localidad y año usa el índice secundario `idx_fact_loc_anio`.

## Decisiones y límites

- **Tamaño.** 38 MB no es Big Data por volumen. El valor del proyecto está en el flujo completo y en que cada etapa usa herramientas pensadas para escalar, no en el tamaño del archivo.
- **MapReduce.** HDFS corre como servicio real (NameNode y DataNode, un solo nodo). Los trabajos usan el ejecutor local de Hadoop, sin YARN, porque Colab es una sola máquina.
- **Hora de los siniestros.** `FECHA_HORA_ACC` trae un sufijo `+00`. Se tratan como hora local de Bogotá porque la distribución horaria muestra picos en horas laborales. Es una inferencia, no algo que confirme el portal.
- **Modelo dimensional.** El archivo original es una tabla plana. Las dimensiones (localidad, clase, gravedad, tiempo) se construyen en el cuaderno 1.
- **Seguros.** Los datos son de siniestralidad vial, no de seguros. Sirven para dimensionar riesgo, no para calcular primas.

## Fuente y licencia de los datos

Secretaría Distrital de Movilidad de Bogotá, a través de Datos Abiertos Bogotá, bajo licencia Creative Commons Attribution 4.0.
