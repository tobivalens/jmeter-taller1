# Experimentación: ¿Qué pasaría si...?

Plan: `Experimentacion.jmx` (distinto de `pruebas/jmeter/Ticketmania.jmx`; el pipeline no lo ejecuta).

## Cómo está armado

```
Plan de Pruebas
├── CSV Data Set Config (datos.csv: tarifa, cantidad, total_esperado)
├── Gestor de Cookies HTTP
├── Valores por defecto HTTP (127.0.0.1:8080)
├── Gestor de Cabecera HTTP (Content-Type: application/json)
├── Reporte resumen
├── Grupo "Flujo de compra"            hilos=${hilos} subida=${subida} bucles=${bucles} duración=${duracion}
│   ├── GET /api/tarifas
│   ├── POST /api/seleccion
│   │   ├── Extractor JSON: num_reserva ($.reserva)
│   │   └── Temporizador Aleatorio Uniforme (temp_min + 0..temp_rango ms)
│   └── POST /api/pago
│       ├── Aserción JSON: estado = APROBADO
│       └── Aserción JSON: total = total_esperado (CSV)   ← pregunta 9
└── Grupo "Concurrencia"               hilos=${hilos_sync} subida=${subida_sync}
    ├── POST /api/seleccion
    │   ├── Extractor JSON: num_reserva
    │   └── Temporizador Sincronizado (agrupa hilos_sync usuarios)
    └── POST /api/pago (mismas aserciones)
```

Los valores se cambian con `-J` al ejecutar, sin editar el `.jmx`. Así cambias **una sola variable por ejecución** (guía §12).

| Propiedad | Por defecto | Qué controla |
|---|---|---|
| `hilos` | 20 | Usuarios del grupo "Flujo de compra" (0 lo apaga) |
| `subida` | 1 | Periodo de subida en segundos |
| `bucles` | 1 | Contador del bucle (-1 = infinito, para pruebas por duración) |
| `duracion` | 3600 | Duración máxima en segundos (corta antes si terminan los bucles) |
| `temp_min` | 500 | Desplazamiento constante del temporizador (ms) |
| `temp_rango` | 1000 | Retardo máximo aleatorio (ms). `0` y `0` = sin temporizador |
| `hilos_sync` | 0 | Usuarios del grupo "Concurrencia" (0 lo apaga) y tamaño del grupo sincronizado |
| `subida_sync` | 0 | Subida del grupo de concurrencia |
| `espera_sync` | 60000 | Tiempo máximo (ms) que el Temporizador Sincronizado espera a juntar el grupo |

## Cómo ejecutar

1. En una terminal, levanta la API y déjala abierta:

```bash
python api/ticketmania_simple.py
```

2. En otra terminal, desde `pruebas/experimentacion/`, ejecuta (cambia el nombre de la carpeta del reporte en cada corrida, porque debe estar vacía):

```bash
jmeter -n -t Experimentacion.jmx -Jhilos=20 -l q1_20.jtl -e -o reporte_q1_20
```

3. Abre `reporte_q1_20/index.html` y lee **Muestras, Media, Máx, % Error y Rendimiento** en la tabla *Statistics*.

> Reinicia la API entre preguntas si quieres partir de cero: guarda las reservas en memoria.

## Comandos sugeridos por pregunta

Son un punto de partida. Tú decides los valores finales y debes poder justificarlos.

| # | Qué cambia | Ejecuciones |
|---|---|---|
| 1 | Usuarios: 20 → 500 | `-Jhilos=20 -Jsubida=10` y `-Jhilos=500 -Jsubida=10` |
| 2 | Subida: 1 s → 60 s | `-Jhilos=200 -Jsubida=1` y `-Jhilos=200 -Jsubida=60` (mira la gráfica *Response Times Over Time* del reporte) |
| 3 | Temporizador sí/no | `-Jhilos=200 -Jsubida=10` y `-Jhilos=200 -Jsubida=10 -Jtemp_min=0 -Jtemp_rango=0` |
| 4 | Escalones 50/100/250/500 | Cuatro ejecuciones: `-Jhilos=50 -Jbucles=-1 -Jduracion=60 -Jsubida=10`, luego 100, 250, 500. Compara el Rendimiento de cada una |
| 5 | Duración 1 vs 10 min | `-Jhilos=50 -Jbucles=-1 -Jduracion=60` y `-Jhilos=50 -Jbucles=-1 -Jduracion=600` (mira *Response Times Over Time*) |
| 6 | tarifas vs seleccion | Cualquier corrida del flujo: compara la **Media** de `GET /api/tarifas` y `POST /api/seleccion` en la misma tabla |
| 7 | 500 reservas a la vez | `-Jhilos=0 -Jhilos_sync=500` |
| 8 | Sincronizado vs distribuido | `-Jhilos=0 -Jhilos_sync=100` y `-Jhilos=100 -Jsubida=60 -Jtemp_min=0 -Jtemp_rango=0` |
| 9 | total = precio × cantidad | Cualquier corrida con carga (por ejemplo `-Jhilos=200 -Jsubida=10`): si la aserción `total = total_esperado` falla, aparece en % Error |
| 10 | CPU y memoria con 500 | Agrega en la GUI el listener *PerfMon Metrics Collector* (plugin jpgc, ServerAgent en el puerto 4444) con CPU y Memory, guarda a archivo, y ejecuta `-Jhilos=500 -Jsubida=10 -Jbucles=-1 -Jduracion=120` |

## Notas que ya se observaron en una prueba de humo

- Con `-Jhilos_sync=50`, muchas peticiones fallan con `Non HTTP response code: org.apache.http.conn.HttpHostConnectException` (conexión rechazada). La API usa `ThreadingHTTPServer` de Python, cuya cola de conexiones pendientes es pequeña: si llegan muchas conexiones en el mismo instante, el sistema operativo rechaza las que no caben. Es un resultado de concurrencia, no un error del plan. Te sirve para las preguntas 7 y 8.
- Cuando `seleccion` falla, `num_reserva` queda en `num_reserva_not_found` y el `pago` también falla (404). Por eso los errores se duplican: cuenta las fallas de `seleccion` por separado.

## Limitaciones para tu entrega

- La API y JMeter corren en el mismo computador: compiten por CPU y memoria (guía §13).
- JMeter mide la respuesta HTTP, no el navegador.
- Los resultados varían entre corridas: repite cada escenario al menos dos veces.
