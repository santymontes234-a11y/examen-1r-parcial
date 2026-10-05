# Informe — Análisis de sensores industriales

Examen práctico, Manejo Masivo de Datos.

**Resultados del análisis (`analisis`) usados en este informe**

| Resultado | Valor |
|---|---|
| Registros / sensores distintos | 100,000 / 40 (10 por planta) |
| Periodo cubierto | 01/09/26 0:00 a 02/09/26 17:39, una lectura por minuto por sensor |
| Temperatura promedio | Planta_1: 66.62 °C, Planta_2: 66.53 °C, Planta_3: 66.77 °C, Planta_4: 66.67 °C |
| Temperatura máxima | 104.99 °C, empatada en 4 registros (S023, S019, S014, S030) |
| Alertas (> 85 °C) | 6,954 lecturas (6.95 % del total) |
| Planta con más alertas | Planta_3, con 1,777 |
| Tamaño del archivo | 4.6 MB |

---

## 5. Las 5 V aplicadas al proyecto

| V | Relación con el sistema de sensores | Ejemplo concreto | ¿Dónde aparece? |
|---|---|---|---|
| **Volumen** | Cantidad de datos que generan los sensores y que hay que almacenar y procesar. | El CSV tiene 100,000 mediciones . Con 5,000 sensores midiendo cada segundo serían unos 432 millones de registros al día. 
| **Velocidad** | Rapidez con la que llegan los datos y con la que se necesita reaccionar. | Hoy hay una lectura por minuto por sensor. Con lecturas cada segundo, una alerta de 85 °C tendría que emitirse en pocos segundos. | **Ampliación**. El CSV es un archivo ya guardado y no contiene un flujo en tiempo real. |
| **Variedad** | Distintos formatos y fuentes de datos. | Fotografías de máquinas, reportes de mantenimiento en texto libre y mensajes JSON de sensores, además de la tabla actual. | **Ampliación**. El CSV tiene una sola fuente, tabular, con 6 columnas. |
| **Veracidad** | Confianza y calidad de los datos: nulos, duplicados, valores atípicos, sensores mal calibrados. | Revisé el archivo: no tiene nulos, `id_registro` no se repite y no hay lecturas duplicadas por sensor y minuto. Las temperaturas van de 45.00 a 104.99 °C. Que el máximo (104.99) se repita en 4 registros es un detalle a verificar, porque podría ser una limitación de la simulación o un tope del sensor. | **CSV actual** para la revisión de calidad. Al ser datos simulados, no se puede afirmar que reflejen fallas reales de sensores. |
| **Valor** | Utilidad de los datos para decidir y actuar. | Las 6,954 alertas permiten priorizar inspecciones. Planta_3 concentra la mayor cantidad (1,777) y el sensor S027 es el que más alertas tiene (211). | **CSV actual** para identificar dónde ocurren las alertas. Su valor para prevenir fallas dependería de la **ampliación** (historial de fallas, mantenimiento). |

---

## 6. Tipos de datos y procesamiento tradicional

### Clasificación

| Elemento | Tipo | Justificación |
|---|---|---|
| CSV de sensores | **Estructurado** | Filas y columnas fijas, con tipos definidos (fecha, texto, números). |
| Mensaje JSON de un sensor | **Semiestructurado** | Tiene etiquetas y estructura (clave-valor), pero no un esquema rígido de tabla; los campos pueden variar entre mensajes. |
| Fotografía de una máquina | **No estructurado** | Es una imagen sin campos definidos; para extraer información hay que analizar su contenido. |
| Texto libre de un reporte de mantenimiento | **No estructurado** | Lenguaje natural sin formato fijo. |

### ¿Por qué 100,000 registros no son automáticamente Big Data?

Big Data no se define solo por el número de filas, sino por si el volumen, la velocidad y la variedad de los datos superan lo que una herramienta tradicional puede manejar. Este archivo pesa 4.6 MB: pandas lo carga en memoria y lo analiza en un instante en una computadora personal. Además, es un archivo estático, de un solo formato y una sola fuente, por lo que no presenta velocidad ni variedad.

### Limitaciones al aumentar la escala

- **Memoria:** con cientos de millones de registros al día, los datos ya no caben en la memoria de una sola computadora y `pd.read_csv` fallaría o sería muy lento.
- **Tiempo de procesamiento:** un solo equipo tardaría demasiado en recalcular promedios y alertas sobre meses de historial.
- **Almacenamiento:** un CSV único crece sin control (decenas de GB por día) y es difícil de respaldar, consultar y actualizar.
- **Variedad:** el análisis tradicional con tablas no sirve para fotografías ni texto libre.
- **Tiempo real:** un script que lee un archivo completo no puede emitir alertas al momento en que llega una lectura.

---

## 7. Batch y Streaming

**Tipo de procesamiento que realicé: batch (por lotes).** El programa lee un archivo ya guardado con todas las mediciones, las procesa juntas y genera resultados al final. Los datos están completos antes de empezar, y no importa si el cálculo tarda unos segundos porque nadie necesita el resultado de inmediato.

**Alerta pocos segundos después de una lectura > 85 °C: streaming.** Cada lectura se procesa en cuanto llega, evaluando la condición `temperatura > 85` evento por evento y enviando la alerta al instante (por ejemplo, con un sistema de mensajería de eventos y un procesador de flujos). Un proceso por lotes no sirve aquí porque habría que esperar a juntar un lote, y para entonces la máquina podría haber seguido sobrecalentándose.

**Resumen al terminar el día: batch.** El resumen (promedio por planta, total de alertas, planta con más alertas) necesita todas las mediciones del día. No es urgente: se puede calcular una vez al día, por ejemplo en la noche, sobre los datos acumulados.

**Relación con el tiempo de respuesta:**

| Necesidad | Tiempo aceptable | Enfoque |
|---|---|---|
| Alerta de temperatura | Segundos | Streaming |
| Resumen diario | Horas (una vez al día) | Batch |
| Análisis de este examen | Sin urgencia | Batch |

---

## 8. Lambda y Kappa

### Escenario A: **Arquitectura Lambda**

Se elige porque la empresa quiere **dos rutas distintas**: una por lotes que recalcule el historial completo y otra rápida para las mediciones recientes. Eso es justamente lo que define a Lambda: capa batch (resultados completos y precisos, pero con retraso), capa de velocidad (resultados inmediatos sobre lo reciente) y una capa de servicio que combina ambas para consultar.

```
                 +-----------------------+      +-------------------+
            +--> | Capa batch            | ---> | Vistas batch      | --+
            |    | (recalcula historial) |      +-------------------+   |
+---------+ |    +-----------------------+                              |   +-------------------+     +---------+
| Sensores| +                                                           +-> | Capa de servicio  | --> | Consulta|
+---------+ |    +-----------------------+      +-------------------+   |   | (une ambas vistas)|     | / alerta|
            +--> | Capa de velocidad     | ---> | Vistas en tiempo  | --+   +-------------------+     +---------+
                 | (lecturas recientes)  |      | real              |
                 +-----------------------+      +-------------------+
```

Ventaja: combina precisión del historial con rapidez. Desventaja: hay que mantener la misma lógica en dos códigos distintos.

### Escenario B: **Arquitectura Kappa**

Se elige porque la empresa quiere **una sola lógica de procesamiento de eventos** y conservar las mediciones para reprocesarlas cuando haga falta. En Kappa todo es un flujo de eventos: se guardan en un registro (log) inmutable, un solo procesador de flujos los trata, y si cambia la lógica, se vuelve a procesar el log desde el principio.

```
+---------+     +-----------------------------+     +---------------------+     +-------------------+     +---------+
| Sensores| --> | Registro de eventos (log)   | --> | Procesador de flujos| --> | Almacén de        | --> | Consulta|
+---------+     | conserva todas las lecturas |     | (una sola lógica)   |     | resultados        |     | / alerta|
                +-----------------------------+     +---------------------+     +-------------------+     +---------+
                              ^                                |
                              |   Reprocesamiento: se vuelve a leer el log con la lógica nueva
                              +--------------------------------+
```

Ventaja: una sola lógica, más simple de mantener. Desventaja: reprocesar mucho historial puede ser costoso.

---

## 9. Analítica descriptiva, predictiva y prescriptiva

### Descriptiva (qué pasó)

1. Hubo **6,954 lecturas con alerta** (> 85 °C), el **6.95 %** de las 100,000 mediciones.
2. **Planta_3 es la planta con más alertas (1,777)**, seguida de Planta_1 (1,737), Planta_4 (1,732) y Planta_2 (1,708). La diferencia entre plantas es pequeña: las temperaturas promedio son casi iguales (entre 66.53 y 66.77 °C).

Otros datos reales: la temperatura máxima fue 104.99 °C (4 registros empatados) y el sensor con más alertas fue S027, con 211.

### Predictiva (qué podría pasar)

**Pregunta:** ¿Qué máquinas tienen mayor probabilidad de fallar en los próximos días, a partir de la evolución de su temperatura y vibración?

**Datos adicionales que necesitaría:**
- Historial de fallas reales de cada máquina (fecha y tipo), para tener algo que predecir.
- Registro de mantenimientos realizados.
- Una relación de cada sensor con su máquina concreta, tipo de máquina y antigüedad.
- Carga de trabajo y temperatura ambiente.
- Un periodo mucho más largo que los dos días del archivo, para ver tendencias.

Una lectura por encima del umbral es una alerta del ejercicio; por sí sola no demuestra que una máquina vaya a fallar.

### Prescriptiva (qué hacer)

**Acción propuesta:** programar una revisión preventiva de las máquinas cuyos sensores acumulan alertas de forma repetida (por ejemplo, S027 con 211), empezando por Planta_3, que concentra más alertas.

**Información que revisaría antes de decidir:**
- Si las alertas son aisladas o consecutivas y sostenidas en el tiempo.
- Si la temperatura muestra una tendencia al alza, y si la vibración también la acompaña (en este archivo la vibración promedio con alerta, 3.02 mm/s, es casi igual a la de sin alerta, 3.00 mm/s, así que no hay relación visible).
- Historial de fallas y de mantenimiento de la máquina.
- Que el sensor esté bien calibrado y no esté generando lecturas erróneas.
- El costo de detener la máquina frente al riesgo de no revisarla.
