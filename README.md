# Tiquetmania: pruebas de rendimiento con JMeter

Monorepo con la API de Tiquetmania, el plan de JMeter del despliegue y el plan de experimentación.

```
tiquetmania/
├── api/
│   └── ticketmania_simple.py        # API (Python, puerto 8080)
├── pruebas/
│   ├── jmeter/                      # escenario fijo del pipeline
│   │   ├── Ticketmania.jmx
│   │   └── datos.csv
│   └── experimentacion/             # 10 preguntas, local (el pipeline no lo ejecuta)
│       ├── Experimentacion.jmx
│       ├── datos.csv
│       └── README.md
├── .github/
│   └── workflows/
│       └── rendimiento.yml          # pipeline de GitHub Actions
└── README.md
```

## Ejecutar en local

```bash
python api/ticketmania_simple.py
```

En otra terminal, desde `pruebas/jmeter/`:

```bash
jmeter -n -t Ticketmania.jmx -l resultados.jtl -e -o reporte
```

## Escenario del despliegue

| Parámetro | Valor |
|---|---|
| Grupo de Hilos | estándar, 200 hilos |
| Periodo de subida | 300 s |
| Contador del bucle | 1 |
| Temporizador | Aleatorio Uniforme, 500 a 1500 ms |
| CSV | `datos.csv` (ruta relativa, misma carpeta que el `.jmx`) |

## Pipeline

`.github/workflows/rendimiento.yml` corre con cada push a `main` o a mano (workflow_dispatch): instala Java 17 y JMeter 5.6.3, levanta la API, espera `/api/salud`, ejecuta el plan en modo no-GUI, falla si el `.jtl` tiene peticiones fallidas y sube el reporte como artefacto `reporte-rendimiento` con `if: always()`.
