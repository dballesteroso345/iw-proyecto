# IW Resource Management — Caso Familia Miranda

**Prueba Tecnica de Analisis de Datos** — Procesamiento, modelado relacional y analisis financiero familiar

---

## Descripcion del Proyecto

Pipeline completo de analisis de datos para el caso "Familia Miranda". Cubre desde la ingesta y limpieza de archivos fuente hasta el modelado relacional en 3FN, la generacion de reportes analiticos con visualizaciones y un analisis de texto libre (word cloud) para identificar el deporte favorito de cada miembro.

El codigo esta estructurado con **Programacion Orientada a Objetos** — cada notebook expone una o varias clases con responsabilidades bien delimitadas, eliminando codigo procedural repetido.

### Preguntas de Negocio Respondidas

1. Ejecucion presupuestal: planeado vs. real por rubro y miembro
2. Ingreso mensual total de la familia
3. Gasto mensual total de la familia
4. Capacidad de ahorro: se esta gastando mas de lo que se gana?
5. Top 3 rubros con mayor sobreejercicio presupuestal
6. Medio de pago preferido por cada miembro
7. Gastos no registrados en ningun rubro presupuestado
8. Rubros presupuestados que no registraron ningun gasto
9. Comparativo Agosto vs. Septiembre (analisis senior)
10. Deporte favorito de cada miembro via word cloud (bonus senior)

---

## Arquitectura del Sistema

El proyecto sigue la arquitectura **Medallon** (Bronze → Silver → Gold), estandar en ingenieria de datos moderna:

```
┌─────────────────────────────────────────────────────────────┐
│  BRONZE — Datos crudos                                      │
│  data/raw/                                                  │
│  Gastos_Papa_202308.txt   Gastos_Mama_202308.txt            │
│  Gastos_Papa_202309.txt   Gastos_Mama_202309.txt            │
│  Gastos_Hijo_202309.xlsx  presupuesto.xlsx                  │
├─────────────────────────────────────────────────────────────┤
│  SILVER — Datos procesados y estandarizados                 │
│  data/processed/                                            │
│  gastos_clean.csv         presupuesto_clean.csv             │
│  mapeo_categorias.csv                                       │
├─────────────────────────────────────────────────────────────┤
│  GOLD — Modelo relacional listo para analisis               │
│  data/familia_miranda.db  (SQLite via SQLAlchemy)           │
│  Tablas: miembros / rubros / presupuesto / gastos           │
└─────────────────────────────────────────────────────────────┘
```

Esta separacion garantiza trazabilidad completa desde el dato crudo hasta el reporte final. Si el sistema escalara a multiples familias, la migracion de SQLite a PostgreSQL o Azure SQL Database requeriria cambiar unicamente la cadena de conexion en SQLAlchemy.

---

## Diseño de Clases (OOP)

Cada notebook encapsula su logica en clases con responsabilidades unicas:

```
Notebook 01 — Ingesta y Limpieza
├── DataLoader        load_all_gastos() / load_all_presupuesto()
│                     load_gasto_txt() / load_gasto_xlsx() / load_presupuesto_hoja()
├── DataValidator     summary() / check_non_numeric() / check_duplicate_headers()
├── DataCleaner       clean() → _remove_header_rows / _fix_8mil / _convert_types ...
│                     apply_mapeo()
└── DataExporter      export_gastos() / export_presupuesto() / export_mapeo()

Notebook 02 — Modelo Relacional
├── Miembro           ORM — tabla miembros
├── Rubro             ORM — tabla rubros
├── Presupuesto       ORM — tabla presupuesto (UNIQUE rubro+mes)
├── Gasto             ORM — tabla gastos (id_rubro nullable)
└── DatabaseLoader    load_all() → _load_miembros / _load_rubros /
                      _load_presupuesto / _load_gastos / validate()

Notebook 03 — Analisis y Reportes
├── FamilyAnalyzer    ejecucion_presupuestal() / ingresos() / gastos_totales()
│                     flujo_caja() / top_sobreejecucion() / medio_pago_preferido()
│                     gastos_sin_rubro() / rubros_sin_gasto()
└── FamilyVisualizer  plot_ejecucion() / plot_flujo_caja() / plot_top_sobreejecucion()
                      plot_medios_pago() / plot_comparativo()

Notebook 04 — Word Cloud (Bonus Senior)
└── WordCloudAnalyzer build_corpus() / generate_wordclouds() / detect_sports()
                      print_sports_detection() / print_evidence()
                      plot_sports_frequency()
```

---

## Modelo de Datos (3FN)

El esquema relacional cumple la **Tercera Forma Normal**:

| Forma Normal | Regla | Como se cumple |
|---|---|---|
| 1FN | Valores atomicos, sin grupos repetidos | Cada fila tiene un valor por columna; gastos en filas separadas |
| 2FN | Sin dependencias parciales | Todas las tablas usan surrogate key (INT autoincrement) |
| 3FN | Sin dependencias transitivas | nombre_rubro depende solo de id_rubro; ningun atributo no-clave depende de otro |

```
┌──────────────────┐         ┌───────────────────────────────────────┐
│    miembros      │         │               gastos                  │
│──────────────────│         │───────────────────────────────────────│
│ PK id_miembro    │──┐      │ PK id_gasto         INT  AUTOINCREMENT│
│    nombre        │  └─────>│ FK id_miembro       INT  NOT NULL     │
│    tipo          │         │ FK id_rubro         INT  (nullable)   │
└──────────────────┘         │    fecha            DATE NOT NULL     │
                             │    descripcion      TEXT              │
┌──────────────────┐         │    valor            REAL NOT NULL     │
│     rubros       │         │    forma_pago       TEXT              │
│──────────────────│         │    categoria_origen TEXT              │
│ PK id_rubro      │──┐      │    mes              TEXT NOT NULL     │
│    nombre_rubro  │  └─────>│                                       │
└──────────────────┘         └───────────────────────────────────────┘
         │
         │              ┌────────────────────────────────┐
         │              │          presupuesto           │
         │              │────────────────────────────────│
         │              │ PK id_presupuesto  INT         │
         └─────────────>│ FK id_rubro        INT NOT NULL│
                        │    mes             TEXT NOT NULL│
                        │    valor_planeado  REAL NOT NULL│
                        │    UNIQUE(id_rubro, mes)        │
                        └────────────────────────────────┘
```

**Decisiones de diseño:**
- `gastos.id_rubro` es nullable — permite registrar gastos sin rubro presupuestado sin violar integridad referencial
- `UNIQUE(id_rubro, mes)` en presupuesto — garantiza un solo valor planeado por rubro por mes
- `categoria_origen` preserva la categoria original del archivo fuente para trazabilidad

---

## Estructura del Proyecto

```
iw-proyecto/
├── data/
│   ├── raw/                          # Bronze — archivos fuente sin modificar
│   │   ├── Gastos_Papa_202308.txt
│   │   ├── Gastos_Papa_202309.txt
│   │   ├── Gastos_Mama_202308.txt
│   │   ├── Gastos_Mama_202309.txt
│   │   ├── Gastos_Hijo_202309.xlsx
│   │   └── presupuesto.xlsx
│   ├── processed/                    # Silver — datos limpios y visualizaciones
│   │   ├── gastos_clean.csv
│   │   ├── presupuesto_clean.csv
│   │   ├── mapeo_categorias.csv
│   │   ├── ejecucion_presupuestal.png
│   │   ├── flujo_caja.png
│   │   ├── top3_sobreejercicio.png
│   │   ├── medios_pago.png
│   │   ├── comparativo_meses.png
│   │   ├── wordcloud_familia.png
│   │   └── deportes_familia.png
│   └── familia_miranda.db            # Gold — modelo relacional SQLite
├── notebooks/
│   ├── 01_ingestion_cleaning.ipynb   # Ingesta, diagnostico y limpieza
│   ├── 02_data_model.ipynb           # Modelo relacional 3FN y carga
│   ├── 03_analysis.ipynb             # Reportes analiticos y visualizaciones
│   └── 04_wordcloud_bonus.ipynb      # Word cloud y deteccion de deportes
└── README.md
```

---

## Tecnologias Utilizadas

| Capa | Tecnologia | Uso |
|------|-----------|-----|
| Procesamiento | Python 3.11 + Pandas | Limpieza, transformacion y mapeo de datos |
| Modelado | SQLAlchemy + SQLite | ORM declarativo, esquema relacional 3FN |
| Analisis | SQL + Pandas | Consultas analiticas y agregaciones |
| Visualizacion | Matplotlib + Seaborn | Graficos estadisticos |
| Texto | WordCloud | Nube de palabras sobre descripciones de gastos |
| Entorno | Jupyter Notebook | Desarrollo interactivo reproducible |

---

## Hallazgos Principales

### Flujo de Caja (Agosto vs. Septiembre 2023)

| Mes | Ingresos | Gastos | Ahorro | Tasa |
|-----|----------|--------|--------|------|
| Agosto 2023 | $19.1M COP | $93.2M COP | -$74.2M | negativo* |
| Septiembre 2023 | $21.6M COP | $21.5M COP | +$75K | 0.3% |

*El deficit de agosto se explica por una inversion no recurrente (CDT compra carro ~$78M) sin rubro presupuestado.

### Top 3 Rubros con Mayor Sobreejercicio (Septiembre)

| # | Rubro | Planeado | Real | Exceso |
|---|-------|----------|------|--------|
| 1 | VIAJES | $30K | $407K | +1,256% |
| 2 | PAGO TARJETA NUBE | $1.5M | $2.84M | +89.6% |
| 3 | CUOTA CASA | $3.7M | $4.2M | +13.5% |

### Gastos No Presupuestados

| Categoria | Transacciones | Total |
|-----------|--------------|-------|
| inversiones (CDT carro) | 4 | $78.3M COP |
| PRESTAMO | 2 | $4.0M COP |
| RETIRO CAJERO | 4 | $2.0M COP |
| CUOTAS DE MANEJO | 5 | $211K COP |

### Medios de Pago Preferidos (Septiembre)

| Miembro | Medio Preferido |
|---------|----------------|
| Papa | Efectivo |
| Mama | Debito |
| Hijo | Efectivo |

### Deporte Favorito por Miembro

| Miembro | Deporte | Evidencia |
|---------|---------|-----------|
| Hijo | Ciclismo | "carrera de bici", "salida en bici", "Carrera go rigo", "Carrera de ponal" |
| Papa | Ciclismo + Futbol | "Pastillas frenos bici", "porta bicicletas", "cera piernas", "ayuda futbol thomas" |
| Mama | Caminata / Actividad casual | "Cafe y torta caminata", sin gastos en equipamiento deportivo |

---

## Como Ejecutar

### Requisitos

```bash
pip install pandas sqlalchemy matplotlib seaborn openpyxl wordcloud jupyter
```

### Orden de ejecucion

Los notebooks deben ejecutarse en secuencia ya que cada uno produce los archivos que consume el siguiente:

```
01_ingestion_cleaning  →  gastos_clean.csv / presupuesto_clean.csv / mapeo_categorias.csv
02_data_model          →  familia_miranda.db
03_analysis            →  graficos en data/processed/
04_wordcloud_bonus     →  wordcloud_familia.png / deportes_familia.png
```

```bash
jupyter nbconvert --to notebook --execute notebooks/01_ingestion_cleaning.ipynb
jupyter nbconvert --to notebook --execute notebooks/02_data_model.ipynb
jupyter nbconvert --to notebook --execute notebooks/03_analysis.ipynb
jupyter nbconvert --to notebook --execute notebooks/04_wordcloud_bonus.ipynb
```

O abrir cada notebook en Jupyter y ejecutar celdas en orden.

---

## Proceso de Limpieza (Notebook 01)

| Problema detectado | Accion tomada |
|--------------------|--------------|
| Valor `"8MIL PESOS"` no numerico | Imputado a `8000` (Papa, Ago 2023, Mercado Origen) |
| Columna `fecha` en formato string | Convertida a `datetime` |
| Espacios y mayusculas inconsistentes | Normalizados con `str.strip()` |
| Categorias de gastos distintas a rubros del presupuesto | Tabla de mapeo de 37 reglas |
| Gastos sin rubro equivalente (inversiones, prestamos) | Registrados con `id_rubro = NULL` |

Por cuestiones de tiempo y para garantizar la exactitud de los números, he priorizado deliberadamente la robustez de la arquitectura de datos, la limpieza programática y la normalización del modelo sobre la creación del tablero en Power BI.

Al entregar una base de datos limpia y estructurada en Tercera Forma Normal (Capa Gold), el modelo ya está completamente listo para ser conectado a Power BI. Todas las preguntas de negocio fueron resueltas y demostradas directamente con consultas analíticas en el código.

## Autor

Diego Ballesteros — Analisis de Datos y Business Intelligence
