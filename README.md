# Análisis de sensores industriales

Examen práctico del primer parcial — Manejo Masivo de Datos.

## Objetivo

Analizar con Python las mediciones de temperatura y vibración de sensores instalados en cuatro plantas industriales, identificar las lecturas con alerta de temperatura y documentar los resultados en un proyecto reproducible.

## Aviso: datos simulados

Los datos de `data/sensores_industriales.csv` son **simulados**. No provienen de sensores ni de plantas reales. El umbral de alerta (temperatura mayor que 85 °C) es una regla didáctica del examen.

## Descripción de los datos

El archivo contiene 100,000 mediciones de 40 sensores (10 por planta), con una lectura por minuto, del 01/09/26 0:00 al 02/09/26 17:39. Las fechas están en formato día/mes/año.

| Columna | Significado |
|---|---|
| `id_registro` | Identificador de la medición |
| `fecha_hora` | Fecha y hora de la lectura |
| `id_sensor` | Identificador del sensor |
| `planta` | Planta donde está instalado |
| `temperatura_c` | Temperatura en grados Celsius |
| `vibracion_mm_s` | Vibración en milímetros por segundo |

## Estructura del repositorio

```
.
├── data/sensores_industriales.csv
├── analisis.ipynb
├── resultados/alertas.csv
├── evidencias/
├── informe.md
├── requirements.txt
├── .gitignore
└── README.md
```

## Instalación (Ubuntu)

Requiere `git`, `python3` y `python3-venv`.

```bash
git clone https://github.com/TU_USUARIO/TU_REPOSITORIO.git
cd TU_REPOSITORIO
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Ejecución

Desde la raíz del repositorio y con el entorno virtual activado:

```bash
jupyter nbconvert --to notebook --execute --inplace analisis.ipynb
```

Esto ejecuta todas las celdas de `analisis.ipynb` y genera `resultados/alertas.csv`. Para verlo de forma interactiva:

```bash
jupyter notebook analisis.ipynb
```

y elige **Run → Run All Cells**. El notebook usa rutas relativas, por lo que debe abrirse desde la raíz del repositorio.

## Resultados que calcula el análisis

- Cantidad de registros y de sensores distintos.
- Temperatura promedio de cada planta.
- Temperatura máxima, con su sensor y fecha (se muestran todos los empates).
- Número de lecturas con temperatura mayor que 85 °C.
- Planta con más alertas (se muestran todos los empates).
- Exportación de todas las lecturas con alerta a `resultados/alertas.csv`, con las columnas originales.

## Informe

Las respuestas de la parte de Big Data (5 V, tipos de datos, batch y streaming, Lambda y Kappa, tipos de analítica) están en [`informe.md`](informe.md).
