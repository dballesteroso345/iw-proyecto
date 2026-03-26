# IW Resource Management — Caso Familia Miranda

> **Prueba técnica de Análisis de Datos** · Procesamiento, modelado y visualización de datos financieros familiares

---

## Descripción del Proyecto

Este proyecto implementa un pipeline completo de análisis de datos para el caso "Familia Miranda", que incluye la limpieza de datos financieros de 3 miembros de familia (Padre, Madre, Hijo), el modelado de un esquema relacional normalizado en 3FN, y la generación de reportes analíticos con visualizaciones.

### Objetivos de Negocio

- Analizar la ejecución presupuestal (planeado vs. real)
- Identificar rubros con mayor sobreejercicio
- Determinar el flujo de caja familiar (ingresos vs. gastos)
- Detectar medios de pago preferidos por miembro
- Identificar gastos no presupuestados y rubros sin utilizar

---

## Estructura del Proyecto

```
iw-proyecto/
├── data/
│   ├── raw/                        # Archivos fuente originales
│   │   ├── Gastos_Hijo_202309.xlsx
│   │   ├── Gastos_Mama_202308.txt
│   │   ├── Gastos_Mama_202309.txt
│   │   ├── Gastos_Papa_202308.txt
│   │   ├── Gastos_Papa_202309.txt
│   │   └── presupuesto.xlsx
│   └── processed/                  # Datos limpios y resultados
│       ├── gastos_clean.csv
│       ├── presupuesto_clean.csv
│       ├── mapeo_categorias.csv
│       ├── familia_miranda.db      # Base de datos SQLite
│       ├── ejecucion_presupuestal.png
│       ├── flujo_caja.png
│       ├── medios_pago.png
│       └── top3_sobreejercicio.png
├── notebooks/
│   ├── 01_ingestion_cleaning.ipynb  # Limpieza y estandarización
│   ├── 02_data_model.ipynb          # Modelo relacional 3FN
│   └── 03_analysis.ipynb            # Reportes y visualizaciones
└── README.md
```

---

## Tecnologías Utilizadas

| Capa | Tecnología | Uso |
|------|------------|-----|
| **Procesamiento** | Python 3.11 + Pandas | Limpieza y transformación de datos |
| **Modelado** | SQLAlchemy + SQLite | Base de datos relacional normalizada |
| **Análisis** | SQL + Pandas | Consultas y agregaciones |
| **Visualización** | Matplotlib + Seaborn | Gráficos estadísticos |
| **Entorno** | Jupyter Notebook | Desarrollo interactivo |

---

## Modelo de Datos (3FN)

El modelo relacional implementa la **Tercera Forma Normal** para garantizar integridad y eliminar redundancia:

```
┌──────────────────┐         ┌───────────────────────────────────────┐
│    miembros      │         │               gastos                  │
│──────────────────│         │───────────────────────────────────────│
│ PK id_miembro    │──┐      │ PK id_gasto         INT  AUTOINCREMENT│
│    nombre        │  └─────>│ FK id_miembro       INT  NOT NULL     │
│    tipo          │         │ FK id_rubro         INT               │
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

### Decisiones de Diseño Clave

- **Integridad referencial**: Todos los gastos están vinculados a un miembro válido mediante FK
- **Rubros nulos permitidos**: Permite registrar gastos sin categoría presupuestal
- **Constraint UNIQUE**: Garantiza un solo presupuesto por rubro-mes
- **Trazabilidad**: Se preserva la categoría original de cada transacción

---

## Cómo Ejecutar

### Requisitos

```bash
# Crear entorno virtual (opcional)
conda create -n iw_project python=3.11
conda activate iw_project

# Instalar dependencias
pip install pandas sqlalchemy matplotlib seaborn openpyxl jupyter
```

### Ejecución

```bash
# Ejecutar notebooks en orden
jupyter nbconvert --to notebook --execute notebooks/01_ingestion_cleaning.ipynb
jupyter nbconvert --to notebook --execute notebooks/02_data_model.ipynb
jupyter nbconvert --to notebook --execute notebooks/03_analysis.ipynb
```

---

## Hallazgos Principales

### 1. Ejecución Presupuestal (Septiembre 2023)

| Métrica | Valor |
|---------|-------|
| **Ingresos totales** | ~$21.5M COP |
| **Gastos totales** | ~$24M COP |
| **Déficit** | ~$2.5M COP |

### 2. Top 3 Rubros con Mayor Sobreejercicio

| Rubro | Planeado | Real | Exceso |
|-------|----------|------|--------|
| PAGO SALUD Y PENSIONES | $7.8M | $9.1M | +17% |
| COMIDAS AFUERA | $500K | $850K | +70% |
| TRANSPORTE | $1.2M | $1.8M | +50% |

### 3. Gastos No Presupuestados

- **PRESTAMO**: $4.0M COP
- **inversiones**: $78.3M COP (CDT compra carro)
- **RETIRO CAJERO**: $2.0M COP

### 4. Medios de Pago Preferidos

| Miembro | Medio Preferido |
|---------|-----------------|
| Papá | Débito |
| Mamá | Débito |
| Hijo | Efectivo |

---

## Visualizaciones Generadas

| Gráfico | Descripción |
|---------|-------------|
| `ejecucion_presupuestal.png` | Comparativo planeado vs. real por rubro |
| `flujo_caja.png` | Ingresos vs. gastos vs. ahorro por mes |
| `top3_sobreejercicio.png` | Top 3 rubros con mayor exceso |
| `medios_pago.png` | Distribución de medios de pago por miembro |

---

## Notas Técnicas

### Proceso de Limpieza

1. **Normalización de formatos**: Se estandarizó el separador `;` y comillas `'` de los archivos TXT
2. **Imputación de valores**: El valor `"8MIL PESOS"` se convirtió a `8000`
3. **Mapeo de categorías**: 37 categorías mapeadas a 37 rubros del presupuesto
4. **Conversión de tipos**: Fechas a `datetime`, valores a `float`

### Arquitectura por Capas (Medallón)

```
Bronze (RAW)    → Archivos fuente sin modificar
Silver (PROCESSED) → Datos limpios en CSV
Gold (ANALÍTICA)   → Modelo relacional SQLite
```

---

## Autor

**Diego Ballesteros** — Análisis de Datos y BI

---

## Licencia

Proyecto desarrollado como prueba técnica. Uso académico y profesional.
