# Respuestas del Ejercicio 2, basadas en tus resultados

Todos los números salen de las carpetas `reporte_q*` (archivo `statistics.json` de cada reporte) y de los `.jtl`.
Ajusta la redacción a tus palabras antes de pegarla en el Word.

**Cómo leer las cifras:** *Media*, *p90* y *Máx* son tiempos de respuesta en milisegundos. *Rendimiento* es el throughput en peticiones por segundo. Cada usuario hace 3 peticiones: tarifas, seleccion y pago.

**Entorno (para la portada):** JMeter 5.6.3, Java 17 y la API en Python (`ThreadingHTTPServer`, puerto 8080), todo en el mismo computador con Windows 11.

> **Un truco para las gráficas de las preguntas 2 y 5.** El reporte HTML agrupa las gráficas *Over Time* por minuto, así que una prueba de 2 segundos o de 1 minuto sale como un solo punto. Para verlas segundo a segundo, regenera el reporte desde el `.jtl` que ya tienes; no hace falta volver a correr la prueba:
>
> ```bash
> & $jm -g q2_1.jtl -o reporte_q2_1_seg -Jjmeter.reportgenerator.overall_granularity=1000
> ```
>
> Haz lo mismo con `q2_60`, `q5_1min` y `q5_10min`, cambiando el nombre de la carpeta de salida.

---

## Pregunta 1: ¿qué pasa si subo de 20 a 500 usuarios?

**Escenario:** Grupo de Hilos con 20 y luego 500 hilos, subida de 10 s, 1 bucle, Temporizador Aleatorio Uniforme de 500 a 1500 ms. Solo cambia el número de usuarios.

| Usuarios | Muestras | Media | Máx | % Error | Rendimiento |
|---|---|---|---|---|---|
| 20 | 60 | 5,6 ms | 30 ms | 0 % | 5,8 /s |
| 500 | 1.500 | 11,5 ms | 259 ms | 8,93 % | 134,0 /s |

**Respuesta:** cambian los dos.
- **Throughput:** se multiplicó por 23 (de 5,8 a 134 peticiones por segundo). Es casi proporcional al aumento de usuarios, que fue de 25 veces.
- **Tiempo de respuesta:** la media se duplicó (de 5,6 a 11,5 ms) y el máximo se multiplicó por 8,6 (de 30 a 259 ms).
- **Errores:** aparecieron errores que con 20 usuarios no existían, el 8,93 %. Casi todos son `HttpHostConnectException`, es decir, conexión rechazada: con 50 usuarios entrando por segundo, la API no alcanza a aceptar todas las conexiones. Los demás son 8 pagos con 404, porque su `seleccion` había fallado antes y no existía la reserva.

**Limitación:** la prueba de 500 usuarios duró apenas 11 segundos, porque es un solo bucle con subida de 10 s. Mide un pico corto, no una carga sostenida. Además JMeter y la API comparten el computador y compiten por la CPU.

**Capturas:**
1. La terminal con los dos comandos ejecutados.
2. La tabla *Statistics* de `reporte_q1_20`.
3. La tabla *Statistics* de `reporte_q1_500`.
4. La tabla *Errors* de `reporte_q1_500`, para mostrar el tipo de error.

---

## Pregunta 2: ¿qué pasa si el ramp-up pasa de 1 s a 60 s?

**Escenario:** 200 hilos, 1 bucle, con temporizador. Solo cambia la subida: 1 s frente a 60 s.

| Subida | Duración | Media | p90 | Máx | % Error | Rendimiento |
|---|---|---|---|---|---|---|
| 1 s | 2 s | 10,4 ms | 48,9 ms | 105 ms | 15,67 % | 295,1 /s |
| 60 s | 61 s | 5,6 ms | 8,0 ms | 130 ms | 0 % | 9,8 /s |

**Respuesta:** sí, el pico inicial cambia por completo.
- **Subida de 1 s:** los 200 usuarios llegan prácticamente juntos y toda la carga se concentra en los primeros 2 segundos. El 15,67 % de las peticiones falló por conexión rechazada, y la petición más golpeada fue `GET /api/tarifas`, con 42 % de error, porque es la primera que hace cada usuario y recibe el golpe de llegada.
- **Subida de 60 s:** entran unos 3 usuarios por segundo, no hubo ningún error y el p90 bajó de 48,9 a 8 ms.
- **El Máx:** fue más alto con 60 s (130 ms), pero es una sola petición aislada. El p90 muestra que el 90 % respondió en 8 ms o menos.

**Limitación:** la prueba con subida de 1 s duró solo 2 segundos. Hay muy pocos datos en el tiempo y con la agrupación por minuto del reporte el pico no se ve en la gráfica; por eso conviene regenerar el reporte a 1 s.

**Capturas:**
1. La tabla *Statistics* de los dos reportes.
2. *Charts → Over Time → Response Times Over Time* y *Active Threads Over Time* de `reporte_q2_1_seg` y `reporte_q2_60_seg` (los regenerados a 1 s).
3. La tabla *Errors* de `reporte_q2_1`.

---

## Pregunta 3: ¿qué pasa si quito los temporizadores?

**Escenario:** 200 hilos, subida de 10 s, 1 bucle. Solo cambia el temporizador: de 500 a 1500 ms, frente a 0 ms. La corrida con temporizador se hizo dos veces.

| Temporizador | Media | Máx | % Error | Rendimiento |
|---|---|---|---|---|
| Con (corrida 1) | 5,0 ms | 51 ms | 0 % | 54,4 /s |
| Con (corrida 2) | 6,7 ms | 120 ms | 0,17 % | 54,3 /s |
| Sin | 3,1 ms | 31 ms | 0 % | 61,5 /s |

**Respuesta:**
- **Cuánto subió:** el throughput subió solo un 13 % (de 54,3 a 61,5 por segundo).
- **Por qué tan poco:** con un solo bucle, el ritmo lo pone la subida. Entran 20 usuarios por segundo y cada uno hace 3 peticiones, lo que da unas 60 peticiones por segundo como máximo, haya o no temporizador. Sin temporizador, cada usuario termina sus 3 peticiones en milisegundos en vez de en un segundo.
- **Qué significa para la prueba:** el temporizador representa el tiempo que una persona real tarda en pensar y hacer clic. Sin él, la prueba deja de simular personas y se convierte en ráfagas de máquina, es decir, en una prueba de estrés. El throughput que mide ya no representa el uso real.
- **Repetición:** las dos corridas con temporizador dieron casi el mismo throughput (54,4 y 54,3). Eso muestra que el resultado es repetible.

**Limitación:** como el escenario tiene un solo bucle, la comparación quedó limitada por la subida y no muestra el efecto completo del temporizador. Para verlo habría que correr por duración con bucles infinitos (ver la corrida opcional abajo). Además, el temporizador de 500 a 1500 ms es una suposición, no un dato medido de usuarios reales.

> **Opcional, para reforzar la respuesta (1 minuto):** compara con `reporte_q4_50`, que tiene el mismo escenario pero con temporizador.
>
> ```bash
> & $jm -n -t Experimentacion.jmx -Jhilos=50 -Jsubida=10 -Jbucles=-1 -Jduracion=60 -Jtemp_min=0 -Jtemp_rango=0 -l q3_sin_dur.jtl -e -o reporte_q3_sin_dur
> ```

**Capturas:**
1. La tabla *Statistics* de `reporte_q3_con`, `reporte_q3_con_2` y `reporte_q3_sin`.
2. En la interfaz de JMeter, el Temporizador Aleatorio Uniforme, para mostrar qué se quitó.

---

## Pregunta 4: escalones de 50, 100, 250 y 500 usuarios. ¿En qué punto deja de crecer el throughput?

**Escenario:** bucles infinitos, duración de 60 s, subida de 10 s, con temporizador. Solo cambia el número de usuarios.

| Usuarios | Muestras | Media | Máx | % Error | Rendimiento total | Rendimiento exitoso* |
|---|---|---|---|---|---|---|
| 50 | 8.183 | 2,6 ms | 56 ms | 0 % | 136,6 /s | 136,6 /s |
| 100 | 16.269 | 2,4 ms | 23 ms | 0,07 % | 271,5 /s | 271,3 /s |
| 250 | 40.316 | 6,1 ms | 550 ms | 57,52 % | 672,8 /s | 285,8 /s |
| 500 | 61.350 | 91,4 ms | 1.505 ms | 76,35 % | 1.022,2 /s | 241,7 /s |

\* Rendimiento exitoso = rendimiento × (1 − % Error). Lo calculas tú con la tabla.

**Respuesta:** el throughput deja de crecer entre 100 y 250 usuarios.
- **De 50 a 100 usuarios:** el throughput se duplicó exacto (de 136 a 271 por segundo), sin errores y con la media estable en unos 2,5 ms.
- **A 250 usuarios:** el rendimiento total parece seguir subiendo, pero es engañoso: el 57 % de las peticiones falló en milisegundos, y un error rápido también cuenta como "petición por segundo". Contando solo las exitosas, el throughput se quedó en unas 286 por segundo, casi igual que con 100 usuarios.
- **A 500 usuarios:** las exitosas bajaron a 242 por segundo, la media subió de 2,4 a 91 ms y el máximo llegó a 1,5 s. El sistema ya está saturado: más usuarios no producen más compras, solo más errores.

**Limitación:**
- **Origen de los errores:** muchos errores de 250 y 500 usuarios son `java.net.BindException: Address already in use`. Ese error no viene de la API: Windows se quedó sin puertos disponibles en el lado de JMeter, porque cada petición abre una conexión nueva. Parte de la saturación es del generador de carga y no de la API.
- **Puntos medidos:** solo se probaron 4 escalones, así que el punto exacto de saturación puede estar entre 100 y 250 usuarios.

**Capturas:**
1. La tabla *Statistics* de los 4 reportes.
2. La tabla *Errors* de `reporte_q4_250`, donde se ve `BindException` y `HttpHostConnectException`.
3. Tu tabla resumen, o una gráfica en Excel de usuarios contra rendimiento exitoso.

---

## Pregunta 5: ¿qué pasa si la prueba dura 10 minutos en lugar de 1?

**Escenario:** 50 hilos, subida de 10 s, bucles infinitos, con temporizador. Solo cambia la duración: 60 s frente a 600 s.

| Duración | Muestras | Media | p90 | Máx | % Error | Rendimiento |
|---|---|---|---|---|---|---|
| 1 min | 8.120 | 3,7 ms | 6 ms | 34 ms | 0,10 % | 135,6 /s |
| 10 min | 87.327 | 4,7 ms | 8 ms | 194 ms | 0,66 % | 145,6 /s |

Media minuto a minuto de la prueba de 10 minutos, en ms: 3,5 · 4,7 · 4,8 · 5,4 · 5,4 · 3,8 · 4,2 · 5,1 · 4,3 · 5,2

**Respuesta:** el tiempo de respuesta se mantiene estable y no sube de forma continua.
- **Media general:** subió apenas 1 ms (de 3,7 a 4,7 ms).
- **Minuto a minuto:** oscila entre 3,5 y 5,4 ms. Sube y baja (por ejemplo, el minuto 6 bajó a 3,8 ms), así que no hay una tendencia a empeorar con el tiempo.
- **Throughput:** se sostuvo en unas 145 peticiones por segundo durante los 10 minutos.
- **Lo que sí cambió:** en 10 minutos aparecieron más picos aislados (máximo de 194 ms frente a 34 ms) y más errores esporádicos de conexión (0,66 % frente a 0,10 %). Una prueba larga expone eventos raros que una corta no alcanza a mostrar.

**Limitación:** 10 minutos sigue siendo poco para detectar una degradación lenta. La API guarda en memoria todas las reservas (más de 29.000 en esta prueba) y ese efecto se vería en pruebas más largas o midiendo la memoria, que aquí no se midió.

**Capturas:**
1. La tabla *Statistics* de los dos reportes.
2. *Response Times Over Time* de `reporte_q5_10min`, que ya muestra 10 puntos por minuto.
3. Si regeneraste a 1 s, la misma gráfica de `reporte_q5_1min_seg`.

---

## Pregunta 6: ¿cuál cuesta más, `/api/tarifas` o `/api/seleccion`, con la misma carga?

**Escenario:** no hace falta una corrida nueva. Se comparan las filas de las dos peticiones dentro de los mismos reportes, así que tienen la misma carga. Recuerda que "costo" significa la Media del tiempo de respuesta.

| Reporte | Muestras por petición | Media de tarifas | Media de seleccion | Diferencia |
|---|---|---|---|---|
| q4_50 (50 usuarios, 1 min) | ~2.700 | 2,3 ms | 3,0 ms | +30 % |
| q4_100 (100 usuarios, 1 min) | ~5.400 | 2,2 ms | 2,8 ms | +27 % |
| q5_1min (50 usuarios) | ~2.700 | 3,4 ms | 4,2 ms | +24 % |
| q5_10min (50 usuarios, 10 min) | ~29.000 | 4,1 ms | 5,5 ms | +34 % |

**Respuesta:** `/api/seleccion` cuesta más.
- **La medición:** en todos los reportes con muchas muestras, `seleccion` tardó entre 24 % y 34 % más que `tarifas`.
- **La razón:**
  - `tarifas` solo devuelve una lista fija de precios.
  - `seleccion` lee y valida el JSON del cuerpo, calcula el total y genera un código de reserva.
  - Además, `seleccion` guarda la reserva dentro de un candado (`threading.Lock`). Ese candado obliga a las peticiones simultáneas a esperar su turno.

**Limitación:**
- **Pruebas pequeñas:** en las corridas con pocas muestras, como la de 20 usuarios, `tarifas` salió más lenta (6,3 ms frente a 5,8 ms). Es la primera petición de cada usuario y paga el costo de la primera conexión, así que con pocas muestras ese efecto pesa más que la diferencia real.
- **Ruido de medición:** la diferencia es de menos de 1,5 ms, cerca del ruido de medición.

**Capturas:**
1. La tabla *Statistics* de `reporte_q4_50` y de `reporte_q5_10min`, resaltando las filas `GET /api/tarifas` y `POST /api/seleccion` (puedes marcarlas con un recuadro en Word).

---

## Pregunta 7: ¿qué pasa si 500 usuarios crean reservas al mismo tiempo?

**Escenario:** grupo "Concurrencia" con 500 hilos y un Temporizador Sincronizado que agrupa a los 500 y los libera en el mismo instante. El grupo "Flujo de compra" estaba apagado (`-Jhilos=0`).

| Petición | Muestras | % Error | Exitosas | Media | Máx |
|---|---|---|---|---|---|
| POST /api/seleccion | 500 | 98,6 % | 7 | 546,9 ms | 866 ms |
| POST /api/pago | 500 | 99,6 % | 2 | 26,6 ms | 352 ms |

**Respuesta:** no. De 500 reservas simultáneas solo 7 se crearon; el 98,6 % falló.
- **El error:** casi todas las fallas son `HttpHostConnectException`, es decir, conexión rechazada. La API nunca llegó a procesar esas peticiones.
- **La causa:** el servidor de Python (`ThreadingHTTPServer`) tiene una cola de conexiones pendientes de solo 5. Cuando llegan 500 en el mismo milisegundo, el sistema operativo rechaza las que no caben.
- **Los pagos:** fallaron en cadena. Sin reserva, la variable `num_reserva` queda como `num_reserva_not_found` y el pago responde 404, o también es rechazado.
- **La media de 547 ms:** es el tiempo que JMeter esperó antes de recibir el rechazo, no el tiempo de crear una reserva.

**Limitación:**
- **Dónde está el fallo:** está en la capa de red del servidor (la cola de conexiones), no en la lógica de reservas. Con esta prueba no se puede saber si la lógica tendría problemas de concurrencia, como reservas duplicadas, porque casi ninguna petición llegó a ejecutarla.
- **Origen de la carga:** los 500 usuarios salen del mismo computador.

**Capturas:**
1. En la interfaz de JMeter, el grupo "Concurrencia" con el Temporizador Sincronizado.
2. La tabla *Statistics* de `reporte_q7`.
3. La tabla *Errors* y *Top 5 Errors by sampler* de `reporte_q7`.

---

## Pregunta 8: ¿sincronizado o distribuido?

**Escenario:** los mismos 100 usuarios en los dos casos. Sincronizado: Temporizador Sincronizado con los 100 liberados a la vez. Distribuido: subida de 60 s y sin temporizador aleatorio.

| Llegada | Duración | Media | Máx | % Error | seleccion exitosas |
|---|---|---|---|---|---|
| Sincronizada | 0,2 s | 24,7 ms | 96 ms | 95,5 % | 7 de 100 |
| Distribuida (60 s) | 60 s | 3,7 ms | 22 ms | 0 % | 100 de 100 |

**Respuesta:** con la misma cantidad de usuarios el resultado es opuesto.
- **Sincronizada:** los 100 llegaron en 0,2 segundos y el 95,5 % de las peticiones falló por conexión rechazada. Solo 7 de 100 reservas se crearon.
- **Distribuida:** los 100 llegaron repartidos en 60 segundos, que son menos de 2 por segundo, y no hubo ningún error. La media fue de 3,7 ms.
- **Conclusión:** el problema no es cuántos usuarios hay, sino cuántos llegan en el mismo instante. La API aguanta bien 100 usuarios, pero no 100 conexiones simultáneas. En una venta real, como la apertura de boletería, la llegada se parece más al caso sincronizado.

**Limitación:**
- **Peticiones distintas:** el grupo distribuido también hace `GET /api/tarifas` y el sincronizado no. Las dos corridas no tienen exactamente las mismas peticiones, aunque `seleccion` y `pago` sí se pueden comparar.
- **Pocos datos en el tiempo:** la prueba sincronizada duró 0,2 s.

**Capturas:**
1. La tabla *Statistics* de `reporte_q8_sync` y de `reporte_q8_dist`.
2. La tabla *Errors* de `reporte_q8_sync`.

---

## Pregunta 9: ¿qué pasa si el total cobrado no coincide con precio × cantidad bajo carga? ¿Cómo lo validarías con el CSV?

**Prueba negativa (`reporte_q9_mal2`):** se cambió `general,2,160000` por `general,2,999` en `datos.csv`, se corrió con 20 usuarios y subida de 5 s, y después se devolvió el valor original.

| Petición | Muestras | % Error | Tipo de error |
|---|---|---|---|
| GET /api/tarifas | 20 | 0 % | — |
| POST /api/seleccion | 20 | 0 % | — |
| POST /api/pago | 20 | 20 % (4 pagos) | Assertion failed |
| Total | 60 | 6,67 % | |

**Cómo se validó:**
- **El CSV:** `datos.csv` tiene una columna `total_esperado` con el resultado de precio × cantidad para cada fila. Por ejemplo, general a 80.000 por 2 boletas da 160.000, y VIP a 350.000 por 7 da 2.450.000.
- **La aserción:** cada usuario lee una fila del CSV. Después de pagar, una Aserción JSON compara el campo `$.total` de la respuesta con `${total_esperado}`. Si no coinciden, la petición queda marcada como error y aparece en el % Error del reporte.

**Respuesta:**
- **Bajo carga:** en todas las corridas, desde 20 hasta 500 usuarios, hubo 50.741 pagos exitosos y ninguno falló por la aserción del total. Cada vez que la API respondió, cobró exactamente precio × cantidad, incluso bajo estrés.
- **Prueba negativa:** al cambiar a propósito un total esperado en el CSV a 999, exactamente 4 de los 20 pagos fallaron con "Assertion failed". Son los 4 usuarios a los que les tocó esa fila: los 20 usuarios se reparten entre las 5 filas del CSV, 4 por fila. La API respondió 200 OK con el total real (160.000), pero la aserción lo marcó como error porque no coincidía con lo esperado. Eso demuestra que, si bajo carga la API llegara a cobrar mal, la prueba lo detectaría aunque la respuesta HTTP fuera exitosa.

**Limitación:**
- **Combinaciones:** solo se validan las 5 combinaciones de tarifa y cantidad del CSV, no todas las posibles.
- **Pagos fallidos:** los pagos que fallaron por conexión rechazada no llegaron a validar el total. No se puede decir nada del total en esos casos.

**Capturas:**
1. `datos.csv` abierto, mostrando la columna `total_esperado`.
2. En la interfaz de JMeter, la Aserción JSON `total = total_esperado`, con `$.total` y `${total_esperado}`.
3. La tabla *Statistics* de una corrida con carga, como `reporte_q4_100`: 16.269 muestras sin fallas por total.
4. La tabla *Statistics* de `reporte_q9_mal2`, con 20 % de error en `POST /api/pago`.
5. La tabla *Errors* de `reporte_q9_mal2`, con "Assertion failed" (4 errores).
6. Opcional: la terminal con el comando y el resumen `Err: 4 (6.67%)`.

---

## Pregunta 10: CPU y memoria con 500 usuarios (pendiente: no la has corrido)

Pasos:
1. Abre el Administrador de tareas (Ctrl + Shift + Esc), pestaña **Rendimiento**.
2. Corre la prueba:
   ```bash
   & $jm -n -t Experimentacion.jmx -Jhilos=500 -Jsubida=10 -Jbucles=-1 -Jduracion=120 -l q10.jtl -e -o reporte_q10
   ```
3. Durante la prueba, captura la gráfica de **CPU** y la de **Memoria**.
4. Captura también la pestaña **Detalles**, ordenada por CPU, con `python.exe` (la API) y `java.exe` (JMeter).

**Plantilla (complétala con lo que veas):**
> Con 500 usuarios durante 2 minutos, la CPU del equipo llegó a ___ % y la memoria a ___ GB (___ %). El proceso que más CPU consumió fue ___ (java.exe es JMeter, python.exe es la API). [Si la CPU estuvo cerca del 100 %: el equipo se saturó.] En ese momento la media de respuesta fue ___ ms y el máximo ___ ms, frente a 2,6 ms de media con 50 usuarios (pregunta 4). Es decir, el tiempo de respuesta se multiplicó por ___. La memoria [se mantuvo estable / subió sin bajar], lo que [indica / no indica] que algo se acumula; la API guarda todas las reservas en memoria.

**Limitación:** no se usó PerfMon con ServerAgent, sino el Administrador de tareas, cuyas gráficas no quedan sincronizadas con el reporte de JMeter. Además, la CPU medida es la del equipo completo: JMeter y la API comparten el mismo computador y se quitan recursos entre ellos, así que no se puede separar cuánto consume cada uno solo por la gráfica general.

**Capturas:**
1. Administrador de tareas → Rendimiento: CPU y Memoria durante la prueba.
2. Administrador de tareas → Detalles: python.exe y java.exe.
3. La tabla *Statistics* de `reporte_q10`.
4. *Response Times Over Time* de `reporte_q10`.

---

## Conclusión general (para el final del Word)

En este computador la API atiende sin errores hasta unos 100 usuarios con tiempo de pensar, alrededor de 270 peticiones por segundo y menos de 3 ms de media. Su punto débil no es el cálculo ni la lógica: el total siempre fue correcto. Lo que falla es la capacidad de aceptar muchas conexiones al mismo tiempo. Cuando los usuarios llegan juntos (subida de 1 s, Temporizador Sincronizado o más de 250 usuarios), la cola de conexiones del servidor se llena y las peticiones se rechazan. Parte de esos errores vienen del propio computador que genera la carga, que se queda sin puertos.
