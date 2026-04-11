# PRESENTACIÓN PRUEBA TÉCNICA
## Análisis de Datos — Caso Familia Miranda
### Diego Ballesteros | 2026

---

# 1. RESUMEN EJECUTIVO

## ¿Qué es este proyecto?
Es un **pipeline de análisis de datos** que procesa información financiera de una familia colombiana para responder preguntas de negocio sobre sus gastos, presupuesto y hábitos.

## ¿Qué problema resuelve?
La familia Miranda necesita entender:
- ¿Cuánto gana y gasta cada mes?
- ¿Están gastando más de lo que ganan?
- ¿En qué rubros se pasan del presupuesto?
- ¿Qué gastos no estaban presupuestados?

## Solución implementada
Un sistema de 4 notebooks que:
1. **Limpia** los datos crudos
2. **Modela** en base de datos relacional (3FN)
3. **Analiza** y genera reportes visuales
4. **Detecta** patrones con word cloud

---

# 2. ARQUITECTURA DEL PROYECTO

## Flujo de datos (Medallón Architecture)

```
┌─────────────────────────────────────────────────────────────┐
│  BRONZE — Datos crudos                                      │
│  Carpeta: data/raw/                                         │
│  - Gastos_Papa_202308.txt                                    │
│  - Gastos_Mama_202308.txt                                    │
│  - presupuesto.xlsx                                          │
│  (Archivos fuente sin modificar)                             │
├─────────────────────────────────────────────────────────────┤
│  SILVER — Datos procesados                                   │
│  Carpeta: data/processed/                                    │
│  - gastos_clean.csv (372 registros limpios)                  │
│  - presupuesto_clean.csv (74 registros)                       │
│  - mapeo_categorias.csv (reglas de equivalencia)             │
├─────────────────────────────────────────────────────────────┤
│  GOLD — Modelo analítico                                     │
│  Archivo: data/familia_miranda.db (SQLite)                   │
│  Tablas: miembros, rubros, presupuesto, gastos               │
└─────────────────────────────────────────────────────────────┘
```

## ¿Por qué esta arquitectura?
- **Bronze**: Guarda los archivos originales sin tocar (trazabilidad)
- **Silver**: Datos limpios y estandarizados, listos para cualquier análisis
- **Gold**: Modelo optimizado para consultas y reportes

---

# 3. ESTRUCTURA DEL CÓDIGO (OOP)

## Notebooks y Clases

### Notebook 01 — Ingesta y Limpieza
**Propósito**: Cargar y limpiar los archivos fuente

| Clase | Responsabilidad | Métodos principales |
|-------|----------------|---------------------|
| `DataLoader` | Cargar archivos | `load_gasto_txt()`, `load_gasto_xlsx()`, `load_presupuesto_hoja()` |
| `DataValidator` | Diagnosticar problemas | `summary()`, `check_non_numeric()`, `check_duplicate_headers()` |
| `DataCleaner` | Limpiar datos | `clean()`, `apply_mapeo()` |
| `DataExporter` | Guardar CSV | `export_gastos()`, `export_presupuesto()` |

**Problemas encontrados y corregidos**:
- Valor "8MIL PESOS" → convertido a 8000
- Fechas en formato string → convertidas a datetime
- Categorías de gastos distintas a rubros → tabla de mapeo con 35 reglas

---

### Notebook 02 — Modelo Relacional
**Propósito**: Crear la base de datos en 3ra Forma Normal (3FN)

| Clase | Tabla | Descripción |
|-------|-------|-------------|
| `Miembro` | miembros | papa, mama, hijo |
| `Rubro` | rubros | 44 categorías de gastos |
| `Presupuesto` | presupuesto | Valor planeado por rubro y mes |
| `Gasto` | gastos | 372 transacciones |

**Clase `DatabaseLoader`**: Carga los datos en el orden correcto respetando las dependencias.

---

### Notebook 03 — Análisis y Reportes
**Propósito**: Responder las 10 preguntas de negocio

| Clase | Responsabilidad |
|-------|----------------|
| `FamilyAnalyzer` | Ejecutar consultas SQL y cálculos |
| `FamilyVisualizer` | Generar gráficos con Matplotlib |

---

### Notebook 04 — Word Cloud (Bonus)
**Propósito**: Detectar el deporte favorito de cada miembro

| Clase | Responsabilidad |
|-------|----------------|
| `WordCloudAnalyzer` | Construir corpus, generar nubes de palabras, detectar deportes |

---

# 4. MODELO DE DATOS (3FN)

## ¿Qué es la Tercera Forma Normal?
Es una regla de diseño de bases de datos que:
- Elimina duplicados
- Evita inconsistencias
- Facilita consultas complejas

## Diagrama

```
┌──────────────────┐         ┌───────────────────────────────────────┐
│    miembros      │         │               gastos                  │
│──────────────────│         │───────────────────────────────────────│
│ PK id_miembro    │──┐      │ PK id_gasto (auto)                    │
│    nombre        │  └─────>│ FK id_miembro (obligatorio)            │
│    tipo          │         │ FK id_rubro (nullable)                │
└──────────────────┘         │    fecha, descripcion, valor...       │
                             └───────────────────────────────────────┘
                                      ▲
┌──────────────────┐                 │
│     rubros       │                 │
│──────────────────│         ┌───────┴───────────────────────────────┐
│ PK id_rubro      │──┐      │           presupuesto                 │
│    nombre_rubro  │  └─────>│───────────────────────────────────────│
└──────────────────┘         │ FK id_rubro + mes = UNIQUE           │
                             │    valor_planeado                     │
                             └───────────────────────────────────────┘
```

## Decisiones clave
- `gastos.id_rubro` es **nullable** (permite gastos sin rubro presupuestado)
- `UNIQUE(id_rubro, mes)` en presupuesto (un solo valor planeado por mes)

---

# 5. PREGUNTAS DE NEGOCIO RESPONDIDAS

## P1. Ejecución Presupuestal (Planeado vs Real)
**Hallazgo**: En septiembre, varios rubros superaron el presupuesto.

| Rubro | Planeado | Real | Exceso |
|-------|----------|------|--------|
| VIAJES | $30,000 | $407,000 | +1,256% |
| PAGO TARJETA NUBE | $1,500,000 | $2,840,000 | +89.6% |
| CUOTA CASA | $3,700,000 | $4,200,000 | +13.5% |

---

## P2. Ingreso Mensual Total
| Mes | Ingresos |
|-----|----------|
| Agosto 2023 | $19.1M COP |
| Septiembre 2023 | $21.6M COP |

---

## P3. Gasto Mensual Total
| Mes | Gastos |
|-----|--------|
| Agosto 2023 | $93.2M COP |
| Septiembre 2023 | $21.5M COP |

**Nota**: Agosto tiene gastos atípicamente altos por una inversión en CDT ($78M).

---

## P4. Capacidad de Ahorro
| Mes | Ingresos | Gastos | Resultado |
|-----|----------|--------|-----------|
| Agosto | $19.1M | $93.2M | **-$74.2M (déficit)** |
| Septiembre | $21.6M | $21.5M | **+$75K (superávit)** |

**Conclusión**: En agosto gastaron más de lo que ganaron debido a una inversión no recurrente (CDT para carro). Sin esa inversión, el flujo sería positivo.

---

## P5. Top 3 Rubros con Mayor Sobreejercicio
1. **VIAJES**: +1,256% (planeado $30K, real $407K)
2. **PAGO TARJETA NUBE**: +89.6%
3. **CUOTA CASA**: +13.5%

---

## P6. Medio de Pago Preferido
| Miembro | Medio Preferido |
|---------|----------------|
| Papá | Efectivo |
| Mamá | Débito |
| Hijo | Efectivo |

---

## P7. Gastos No Presupuestados
Son gastos que no tienen rubro asignado en el presupuesto:

| Categoría | Transacciones | Total |
|-----------|--------------|-------|
| inversiones (CDT) | 4 | $78.3M COP |
| PRESTAMO | 2 | $4.0M COP |
| RETIRO CAJERO | 4 | $2.0M COP |
| CUOTAS DE MANEJO | 5 | $211K COP |

---

## P8. Rubros Sin Gasto
12 rubros del presupuesto no tuvieron ningún gasto registrado (ej: algunos servicios específicos).

---

## P9. Comparativo Agosto vs Septiembre
- **Agosto**: Gasto explosivo por CDT ($78M)
- **Septiembre**: Comportamiento normal, flujo casi equilibrado

---

## P10. Deporte Favorito (Word Cloud)
| Miembro | Deporte | Evidencia |
|---------|---------|-----------|
| Hijo | **Ciclismo** | "carrera de bici", "Carrera go rigo", "Salidas en bici" |
| Papá | **Ciclismo + Fútbol** | "Pastillas frenos bici", "ayuda futbol thomas" |
| Mamá | **Caminata** | "Cafe y torta caminata", sin equipamiento deportivo |

---

# 6. TECNOLOGÍAS UTILIZADAS

| Tecnología | Uso |
|------------|-----|
| **Python 3.11** | Lenguaje principal |
| **Pandas** | Manipulación de datos |
| **SQLAlchemy** | ORM para base de datos |
| **SQLite** | Base de datos embebida |
| **Matplotlib + Seaborn** | Visualizaciones |
| **WordCloud** | Nubes de palabras |
| **Jupyter Notebook** | Desarrollo interactivo |

---

# 7. CÓMO EJECUTAR EL PROYECTO

## Requisitos
```bash
pip install pandas sqlalchemy matplotlib seaborn openpyxl wordcloud jupyter
```

## Orden de ejecución
Los notebooks deben correrse en secuencia:

```
01_ingestion_cleaning → genera gastos_clean.csv, presupuesto_clean.csv
02_data_model         → genera familia_miranda.db
03_analysis           → genera gráficos PNG
04_wordcloud_bonus    → genera wordcloud_familia.png
```

## Comando rápido
```bash
jupyter nbconvert --to notebook --execute notebooks/01_ingestion_cleaning.ipynb
jupyter nbconvert --to notebook --execute notebooks/02_data_model.ipynb
jupyter nbconvert --to notebook --execute notebooks/03_analysis.ipynb
jupyter nbconvert --to notebook --execute notebooks/04_wordcloud_bonus.ipynb
```

---

# 8. GUÍA PARA LA PRESENTACIÓN ORAL

## Estructura sugerida (30 minutos)

### Introducción (3 min)
- "Este proyecto analiza los datos financieros de la familia Miranda"
- "Procesamos 372 transacciones de 3 miembros en 2 meses"
- "Respondimos 10 preguntas de negocio"

### Arquitectura (5 min)
- Explicar las 3 capas: Bronze → Silver → Gold
- "Los datos crudos se guardan sin modificar"
- "Los datos limpios van a CSV"
- "El modelo relacional permite consultas complejas"

### Demo de código (10 min)
- Mostrar notebook 01: cómo se limpian los datos
- Mostrar notebook 02: cómo se crea la base de datos
- Mostrar notebook 03: cómo se resuelven las preguntas

### Resultados (8 min)
- Mostrar las gráficas generadas (ejecución_presupuestal.png, flujo_caja.png)
- Explicar los hallazgos principales (déficit de agosto, deportes)

### Conclusiones (4 min)
- "El sistema está listo para escalar a más familias"
- "Todas las preguntas de negocio fueron respondidas"
- "El modelo 3FN garantiza consistencia de datos"

---

# 9. PREGUNTAS FRECUENTES

## ¿Por qué SQLAlchemy y no SQL directo?
Porque SQLAlchemy es un ORM (Object-Relational Mapping) que:
- Permite cambiar de SQLite a PostgreSQL sin modificar código
- Evita inyección SQL
- Es más mantenible para proyectos grandes

## ¿Por qué nullable en id_rubro?
Porque algunos gastos (como el CDT) no tenían un rubro presupuestado. Si obligamos a tener un rubro, esos registros se perderían.

## ¿Cómo se detectó el deporte favorito?
Buscando palabras clave en las descripciones de gastos:
- "bici", "carrera", "rigo" → ciclismo
- "futbol", "cancha" → fútbol
- Se contó la frecuencia de estas palabras por miembro

---

# 10. ARCHIVOS ENTREGABLES

```
iw-proyecto/
├── data/
│   ├── raw/                    # Archivos originales
│   ├── processed/              # Datos limpios y gráficos
│   └── familia_miranda.db      # Base de datos SQLite
├── notebooks/
│   ├── 01_ingestion_cleaning.ipynb
│   ├── 02_data_model.ipynb
│   ├── 03_analysis.ipynb
│   └── 04_wordcloud_bonus.ipynb
└── README.md                   # Documentación técnica
```

---

# CONCLUSIÓN FINAL

Este proyecto demuestra:

1. **Ingeniería de datos**: Pipeline Bronze-Silver-Gold
2. **Modelado relacional**: Diseño en 3ra Forma Normal
3. **Programación orientada a objetos**: Clases con responsabilidad única
4. **Análisis financiero**: Respuesta a 10 preguntas de negocio
5. **Visualización**: Gráficos claros para tomadores de decisiones

El código está listo para escalar a múltiples familias y puede conectarse a Power BI o Tableau para dashboards interactivos.

---

**Autor**: Diego Ballesteros  
**Tecnologías**: Python, Pandas, SQLAlchemy, SQLite, Matplotlib, WordCloud  
**Fecha**: 2026