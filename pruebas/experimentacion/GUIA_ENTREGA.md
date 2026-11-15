# Guía para armar la entrega (Ejercicio 2)

Qué pide el taller (§11.4) y dónde sale cada cosa:

| Pide | Dónde lo sacas |
|---|---|
| Las 10 preguntas respondidas en tus palabras | Una sección por pregunta (abajo) |
| El escenario que diseñaste: hilos, subida, duración, temporizadores | Tabla de escenario + captura del comando |
| Screenshots del escenario configurado y de los reportes | Captura del plan en la GUI + tabla *Statistics* y gráficas del reporte HTML |
| Resultados observados (reporte y PerfMon si lo usaste) | Los números de la tabla *Statistics* |
| Una limitación por pregunta | Una línea al final de cada pregunta |

---

## 0. Preparación (una sola vez)

En una terminal (deja esta abierta todo el tiempo):

```bash
cd "D:\academico\noveno\pruebas\Nueva carpeta"
python api/ticketmania_simple.py
```

En otra terminal:

```bash
cd "D:\academico\noveno\pruebas\Nueva carpeta\pruebas\experimentacion"
$jm = "D:\jmeter\apache-jmeter-5.6.3\apache-jmeter-5.6.3\bin\jmeter.bat"
```

> Cada vez que abras una terminal nueva repite la línea `$jm = ...`.
> El nombre después de `-o` debe ser una carpeta nueva en cada corrida.
> Para reiniciar la API (borra las reservas en memoria): Ctrl + C en su terminal y vuelve a correrla.

Capturas: **Win + Shift + S**, seleccionas el área y pegas en Word con Ctrl + V.

## 1. Portada y entorno (al inicio del Word)

Escribe una tabla con el entorno (guía §12, "Registra el entorno"):

| Dato | Cómo lo obtienes |
|---|---|
| Equipo y RAM | Configuración → Sistema → Información |
| Procesador | Mismo lugar |
| Versión de Java | `java -version` |
| Versión de JMeter | 5.6.3 |
| Dónde corre la API | En el mismo computador que JMeter |
| Fecha y hora | La que sale en el reporte (*Test and Report information*) |

## 2. Captura del plan (una sola vez)

1. Abre `D:\jmeter\apache-jmeter-5.6.3\apache-jmeter-5.6.3\bin\jmeter.bat` (doble clic).
2. Archivo → Abrir → `Experimentacion.jmx`.
3. Despliega el árbol y toma captura de:
   - El árbol completo (izquierda).
   - El Grupo de Hilos "Flujo de compra" (muestra `${__P(hilos,20)}`, etc.).
   - El Temporizador Aleatorio Uniforme.
   - Las dos Aserciones JSON de `/api/pago`.
   - El CSV Data Set Config.

Qué decir: *"El plan es parametrizado: los hilos, la subida, la duración y el temporizador se pasan con `-J` en la línea de comandos. Así cambio una sola variable por ejecución sin editar el plan."*

Cierra la GUI antes de medir (se mide en modo no-GUI, guía §12).

## 3. Plantilla para cada pregunta

Usa la misma estructura en las 10:

1. **Pregunta** (copiada del taller).
2. **Escenario:** tabla con hilos, subida, bucles/duración, temporizador. Indica qué variable cambiaste.
3. **Evidencia:** captura del comando en la terminal + captura de la tabla *Statistics* de cada corrida (+ gráfica si aplica).
4. **Resultados:** tabla con tus números (Muestras, Media, Máx, % Error, Rendimiento).
5. **Respuesta:** 3-5 líneas en tus palabras, apoyadas en los números.
6. **Limitación:** una línea.

> La tabla *Statistics* está en la página principal del reporte (`index.html`). La fila **Total** resume todo; las demás filas son por petición.
> Las gráficas están en el menú *Charts* → *Over Time* y *Throughput*.

---

## Pregunta 1: de 20 a 500 usuarios

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=20 -Jsubida=10 -l q1_20.jtl -e -o reporte_q1_20
& $jm -n -t Experimentacion.jmx -Jhilos=500 -Jsubida=10 -l q1_500.jtl -e -o reporte_q1_500
```

**Capturas:** tabla *Statistics* de ambos reportes.
**Compara:** columna *Throughput* (Rendimiento) y *Average* (Media) de la fila Total.
**Qué responder:** ¿subió el throughput?, ¿subió el tiempo de respuesta?, ¿o solo uno de los dos? Si el throughput sube y el tiempo casi no cambia, la API todavía no está saturada.
**Limitación posible:** con subida de 10 s y un solo bucle la prueba es corta; o JMeter y la API comparten el mismo PC.

## Pregunta 2: subida de 1 s a 60 s

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=200 -Jsubida=1 -l q2_1.jtl -e -o reporte_q2_1
& $jm -n -t Experimentacion.jmx -Jhilos=200 -Jsubida=60 -l q2_60.jtl -e -o reporte_q2_60
```

**Capturas:** *Charts → Over Time → Response Times Over Time* y *Active Threads Over Time* de ambos. Más la tabla *Statistics*.
**Compara:** ¿hay un pico al inicio con subida de 1 s?, ¿desaparece con 60 s?, ¿cambió el Máx o el % Error?
**Limitación posible:** la granularidad de la gráfica (agrupa por intervalos) puede esconder picos cortos.

## Pregunta 3: quitar los temporizadores

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=200 -Jsubida=10 -l q3_con.jtl -e -o reporte_q3_con
& $jm -n -t Experimentacion.jmx -Jhilos=200 -Jsubida=10 -Jtemp_min=0 -Jtemp_rango=0 -l q3_sin.jtl -e -o reporte_q3_sin
```

**Capturas:** tabla *Statistics* de ambos (y *Transactions Per Second* si quieres).
**Compara:** Throughput con y sin temporizador. Calcula cuántas veces subió (sin ÷ con).
**Qué significa:** sin temporizador los usuarios no "piensan"; la prueba deja de parecerse a personas reales y se vuelve una prueba de estrés.
**Limitación posible:** el temporizador de 500-1500 ms es una suposición, no un dato medido de usuarios reales.

## Pregunta 4: escalones 50, 100, 250, 500

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=50 -Jsubida=10 -Jbucles=-1 -Jduracion=60 -l q4_50.jtl -e -o reporte_q4_50
& $jm -n -t Experimentacion.jmx -Jhilos=100 -Jsubida=10 -Jbucles=-1 -Jduracion=60 -l q4_100.jtl -e -o reporte_q4_100
& $jm -n -t Experimentacion.jmx -Jhilos=250 -Jsubida=10 -Jbucles=-1 -Jduracion=60 -l q4_250.jtl -e -o reporte_q4_250
& $jm -n -t Experimentacion.jmx -Jhilos=500 -Jsubida=10 -Jbucles=-1 -Jduracion=60 -l q4_500.jtl -e -o reporte_q4_500
```

(Cada una dura 1 minuto.)

**Capturas:** tabla *Statistics* de las cuatro.
**Arma en Word** una tabla: Usuarios | Throughput | Media | % Error. Si puedes, haz una gráfica en Excel de Usuarios vs Throughput.
**Qué responder:** ¿en qué escalón el throughput deja de crecer al mismo ritmo (o empiezan los errores)? Ese es el punto de saturación.
**Limitación posible:** solo 4 escalones; el punto real puede estar entre dos de ellos.

## Pregunta 5: 1 minuto vs 10 minutos

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=50 -Jsubida=10 -Jbucles=-1 -Jduracion=60 -l q5_1min.jtl -e -o reporte_q5_1min
& $jm -n -t Experimentacion.jmx -Jhilos=50 -Jsubida=10 -Jbucles=-1 -Jduracion=600 -l q5_10min.jtl -e -o reporte_q5_10min
```

**Capturas:** *Response Times Over Time* de ambos + tabla *Statistics*.
**Qué responder:** ¿la línea del tiempo de respuesta se mantiene plana o sube con los minutos? La API guarda todas las reservas en memoria: en 10 minutos acumula muchas más.
**Tip:** reinicia la API antes de la corrida de 10 minutos.
**Limitación posible:** 10 minutos sigue siendo corto para detectar fugas de memoria.

## Pregunta 6: `/api/tarifas` vs `/api/seleccion`

No necesitas corrida nueva: usa el reporte de la pregunta 1 (500 usuarios) o de la 3.

**Capturas:** la tabla *Statistics*, resaltando las filas `GET /api/tarifas` y `POST /api/seleccion`.
**Compara:** Average, 90th pct y Máx de las dos filas, con la misma carga.
**Qué responder:** cuál cuesta más y por qué (una solo lee una lista fija; la otra valida el cuerpo, calcula el total y guarda una reserva).
**Limitación posible:** las diferencias pueden ser de pocos milisegundos, cerca del ruido de medición.

## Pregunta 7: 500 reservas al mismo tiempo

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=0 -Jhilos_sync=500 -l q7.jtl -e -o reporte_q7
```

**Capturas:** tabla *Statistics* y la tabla *Errors* / *Top 5 Errors by sampler* (más abajo en la misma página).
**Qué responder:** ¿todas fueron exitosas? Mira el % Error y el tipo de error (por ejemplo `HttpHostConnectException` = conexión rechazada).
**Ojo:** si `seleccion` falla, `pago` falla también (404), así que cuenta las fallas de `seleccion` aparte.
**Limitación posible:** el rechazo puede venir del límite de conexiones del sistema operativo/servidor de Python, no de la lógica de la API.

## Pregunta 8: sincronizado vs distribuido

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=0 -Jhilos_sync=100 -l q8_sync.jtl -e -o reporte_q8_sync
& $jm -n -t Experimentacion.jmx -Jhilos=100 -Jsubida=60 -Jtemp_min=0 -Jtemp_rango=0 -l q8_dist.jtl -e -o reporte_q8_dist
```

**Capturas:** tabla *Statistics* de ambos + *Response Times Over Time*. Captura también en la GUI el Temporizador Sincronizado.
**Compara:** % Error y Máx con la misma cantidad de usuarios (100).
**Qué responder:** qué pasa cuando llegan todos en el mismo instante frente a cuando llegan repartidos en 60 s.
**Limitación posible:** el grupo "distribuido" también incluye `GET /api/tarifas`, el sincronizado no.

## Pregunta 9: total cobrado = precio × cantidad

Usa cualquier corrida con carga (por ejemplo la de 500 usuarios de la pregunta 1).

**Capturas:**
1. El archivo `datos.csv` abierto (columnas `tarifa, cantidad, total_esperado`).
2. En la GUI, la Aserción JSON `total = total_esperado` (`$.total` contra `${total_esperado}`).
3. La tabla *Statistics* de la corrida con carga (% Error).

**Prueba de que la aserción funciona** (recomendado): abre `datos.csv`, cambia un `total_esperado` por un valor incorrecto (por ejemplo `160000` → `999`), guarda y corre:

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=20 -Jsubida=5 -l q9_mal.jtl -e -o reporte_q9_mal
```

Captura la tabla *Errors*: debe aparecer el fallo de la aserción. **Después devuelve el valor original.**

**Qué responder:** cómo el CSV trae el total esperado por fila y la aserción lo compara con lo que cobra la API en cada compra; si bajo carga algún total no coincide, aparece como error.
**Limitación posible:** solo valida las 5 combinaciones del CSV, no todas las posibles.

## Pregunta 10: CPU y memoria con 500 usuarios

Abre el **Administrador de tareas** (Ctrl + Shift + Esc) → pestaña **Rendimiento**, y déjalo visible. Luego corre:

```bash
& $jm -n -t Experimentacion.jmx -Jhilos=500 -Jsubida=10 -Jbucles=-1 -Jduracion=120 -l q10.jtl -e -o reporte_q10
```

**Capturas (durante la prueba):**
- Gráfica de CPU y de Memoria en la pestaña Rendimiento.
- Pestaña **Detalles**: los procesos `python.exe` (API) y `java.exe` (JMeter) con su % de CPU y memoria.

**Capturas (al terminar):** tabla *Statistics* y *Response Times Over Time*.
**Qué responder:** ¿la CPU o la memoria llegan cerca del 100 %?, ¿quién consume más (python o java)?, ¿el tiempo de respuesta sube en los momentos de más CPU? Compáralo con la Media de la pregunta 1 (20 usuarios).
**Limitación (obligatoria aquí):** no se usó PerfMon; el Administrador de tareas no queda sincronizado con el reporte. Además JMeter y la API corren en el mismo PC y se roban CPU entre ellos.

---

## Al final del Word

- **Conclusión general** (3-4 líneas): hasta cuántos usuarios aguanta bien la API en tu PC y qué falló primero.
- **Buenas prácticas aplicadas:** modo no-GUI, una variable por corrida, sin Debug Sampler ni Árbol de Resultados.
- Si te da el tiempo, repite 2 escenarios y muestra que los números son parecidos (guía §12, "repite cada escenario").
