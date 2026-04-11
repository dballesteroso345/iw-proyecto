# EXPLICACIÓN COMPLETA DEL CÓDIGO
## Por qué se usó cada clase, método y librería

---

# NOTEBOOK 01: INGESTA Y LIMPIEZA

## ¿Por qué este notebook existe?

Los datos originales (archivos .txt y .xlsx) tienen problemas:
- Formatos inconsistentes (CSV con `;` en lugar de `,`)
- Valores no numéricos ("8MIL PESOS")
- Columnas con nombres con espacios y mayúsculas mezcladas
- Categorías que no coinciden con los rubros del presupuesto

Este notebook soluciona esos problemas y produce archivos limpios.

---

## Librerías importadas

```python
import pandas as pd
import numpy as np
from pathlib import Path
import warnings
```

| Librería | ¿Para qué se usa? |
|----------|-------------------|
| `pandas` | Manipular datos en tablas (DataFrames). Es el estándar de la industria para análisis de datos en Python |
| `numpy` | Operaciones numéricas eficientes. Pandas lo usa por debajo |
| `Path` | Manejar rutas de archivos de forma segura (funciona en Windows, Mac, Linux) |
| `warnings` | Suprimir avisos que no afectan el funcionamiento |

---

## Constantes definidas

```python
_FILAS_EXCLUIR = {
    'Ingresos', 'Egresos Colombia', 'Varios:', 'TOTAL INGRESOS',
    'TOTAL EGRESOS', 'Ahorro Esperado', 'total ahorros'
}
```

**¿Por qué?** 

El archivo de presupuesto tiene filas que son **encabezados y totales**, no datos reales. Por ejemplo:

| Rubro | Valor |
|-------|-------|
| Ingresos | (encabezado) |
| Salario | 5000000 |
| TOTAL INGRESOS | 5000000 | ← Esta fila no debe contarse |

Si no las excluimos, el código contaría "Ingresos" como un rubro más y arruinaría el análisis.

---

## CLASE 1: DataLoader

### ¿Qué hace?

Carga los archivos de datos desde la carpeta `data/raw/`.

### ¿Por qué una clase y no funciones sueltas?

Porque todos los métodos comparten la misma configuración (`self.raw_dir`). Si después queremos cambiar dónde buscar los archivos, solo cambiamos un lugar.

### Código explicado línea por línea

```python
class DataLoader:
    """Carga los archivos fuente del caso Familia Miranda."""

    def __init__(self, raw_dir: Path):
        self.raw_dir = raw_dir
```

**¿Qué hace `__init__`?**
- Es el **constructor**: se ejecuta cuando creas un objeto de esta clase
- `raw_dir` es la carpeta donde están los archivos originales
- Se guarda en `self.raw_dir` para que todos los métodos lo usen

---

```python
def load_gasto_txt(self, filename: str, miembro: str, mes: str) -> pd.DataFrame:
    """Carga un archivo TXT de gastos con separador ';' y comillas simples."""
    df = pd.read_csv(
        self.raw_dir / filename,
        sep=';',
        quotechar="'",
        encoding='utf-8'
    )
    df.columns = df.columns.str.strip().str.replace("'", '', regex=False)
    df['miembro']    = miembro
    df['mes_origen'] = mes
    return df
```

**Línea por línea:**

| Línea | Explicación |
|-------|-------------|
| `pd.read_csv(...)` | Lee el archivo CSV. Los archivos de la familia usan `;` como separador (formato europeo/latino) |
| `sep=';'` | Indica que las columnas se separan con punto y coma, no con coma |
| `quotechar="'"` | Los textos vienen entre comillas simples `'`, esto las quita automáticamente |
| `encoding='utf-8'` | Para que lea correctamente caracteres especiales como ñ y tildes |
| `df.columns.str.strip()` | Quita espacios en blanco de los nombres de columnas |
| `.str.replace("'", '', regex=False)` | Quita las comillas simples que quedaron en los nombres de columna |
| `df['miembro'] = miembro` | Agrega una columna que dice si es papá, mamá o hijo |
| `df['mes_origen'] = mes` | Agrega una columna que dice si es agosto o septiembre |
| `return df` | Devuelve el DataFrame limpio |

**¿Por qué agregar `miembro` y `mes_origen`?**

Porque los archivos vienen separados por persona y mes. Sin estas columnas, después no podríamos saber de quién es cada gasto.

---

```python
def load_gasto_xlsx(self, filename: str, miembro: str, mes: str) -> pd.DataFrame:
    """Carga un archivo Excel de gastos."""
    df = pd.read_excel(self.raw_dir / filename, sheet_name='Sheet1')
    df['miembro']    = miembro
    df['mes_origen'] = mes
    return df
```

**Diferencia con TXT:**
- No necesita `sep` ni `quotechar` porque Excel ya tiene formato definido
- `sheet_name='Sheet1'` indica qué hoja del Excel leer

---

```python
def load_presupuesto_hoja(self, filename: str, hoja: str, mes: str) -> pd.DataFrame:
    """Carga una hoja del presupuesto y retorna solo los rubros validos."""
    df = pd.read_excel(
        self.raw_dir / filename,
        sheet_name=hoja,
        usecols='A:C',
        header=0,
        engine='openpyxl'
    )
    df.columns = ['rubro', 'valor_planeado', 'valor_real']
    df = df[
        df['rubro'].notna() &
        ~df['rubro'].isin(_FILAS_EXCLUIR) &
        df['valor_planeado'].apply(lambda x: isinstance(x, (int, float)))
    ].copy()
    df['mes'] = mes
    return df[['mes', 'rubro', 'valor_planeado']].reset_index(drop=True)
```

**Explicación detallada:**

| Línea | Explicación |
|-------|-------------|
| `usecols='A:C'` | Solo lee las columnas A, B y C del Excel (ignora el resto) |
| `header=0` | La primera fila es el encabezado |
| `engine='openpyxl'` | Motor para leer archivos .xlsx |
| `df.columns = [...]` | Renombra las columnas a nombres más simples |
| `df['rubro'].notna()` | Filtra filas donde el rubro no está vacío |
| `~df['rubro'].isin(_FILAS_EXCLUIR)` | Excluye las filas que son encabezados/totales |
| `isinstance(x, (int, float))` | Solo acepta valores numéricos en valor_planeado |
| `.copy()` | Crea una copia para evitar warnings de Pandas |
| `reset_index(drop=True)` | Reinicia el índice para que sea continuo (0, 1, 2...) |

---

```python
def load_all_gastos(self) -> pd.DataFrame:
    """Carga y consolida todos los archivos de gastos de todos los miembros."""
    frames = [
        self.load_gasto_txt('Gastos_Papa_202308.txt', 'papa', '2023-08'),
        self.load_gasto_txt('Gastos_Papa_202309.txt', 'papa', '2023-09'),
        self.load_gasto_txt('Gastos_Mama_202308.txt', 'mama', '2023-08'),
        self.load_gasto_txt('Gastos_Mama_202309.txt', 'mama', '2023-09'),
        self.load_gasto_xlsx('Gastos_Hijo_202309.xlsx', 'hijo', '2023-09'),
    ]
    return pd.concat(frames, ignore_index=True)
```

**¿Qué hace `pd.concat`?**

Une varios DataFrames en uno solo. Es como pegar tablas verticalmente.

| Parámetro | Explicación |
|-----------|-------------|
| `frames` | Lista de 5 DataFrames (cada archivo) |
| `ignore_index=True` | Reinicia los índices para que sean 0, 1, 2... en lugar de 0,0,0,1,1,1... |

**¿Por qué no hay archivo del hijo en agosto?**
- Porque el enunciado dice que solo existe septiembre para el hijo

---

## CLASE 2: DataValidator

### ¿Qué hace?

Diagnostica problemas en los datos antes de limpiar.

### ¿Por qué separarlo de DataCleaner?

Porque **validar** y **limpiar** son responsabilidades diferentes:
- Validar: Solo detecta y reporta problemas
- Limpiar: Modifica los datos

Si los mezclamos en una clase, el código se vuelve difícil de mantener.

---

```python
class DataValidator:
    """Ejecuta diagnosticos de calidad sobre el DataFrame de gastos."""

    def __init__(self, df: pd.DataFrame):
        self.df = df
```

**¿Por qué guardar el DataFrame en `self`?**
- Para no pasar `df` como parámetro en cada método
- Todos los métodos operan sobre el mismo DataFrame

---

```python
def summary(self) -> None:
    """Imprime resumen general: total de registros, columnas, nulos y tipos."""
    print('DIAGNOSTICO DE CALIDAD — GASTOS CONSOLIDADOS')
    print('=' * 55)
    print(f'Total registros : {len(self.df)}')
    print(f'Columnas        : {list(self.df.columns)}')
    print()
    print('Valores nulos por columna:')
    print(self.df.isnull().sum().to_string())
    print()
    print('Tipos de datos:')
    print(self.df.dtypes.to_string())
```

**¿Qué hace cada parte?**

| Método | Explicación |
|--------|-------------|
| `len(self.df)` | Cuenta cuántas filas hay |
| `self.df.columns` | Lista los nombres de las columnas |
| `self.df.isnull().sum()` | Cuenta valores vacíos (nulos) por columna |
| `self.df.dtypes` | Muestra el tipo de cada columna (texto, número, fecha) |
| `.to_string()` | Convierte el resultado a texto legible |

---

```python
def check_non_numeric(self) -> pd.DataFrame:
    """Retorna filas con valores no numericos en la columna valor."""
    return self.df[
        pd.to_numeric(self.df['valor'], errors='coerce').isna() &
        self.df['valor'].notna()
    ]
```

**Explicación detallada:**

| Parte | Explicación |
|-------|-------------|
| `pd.to_numeric(..., errors='coerce')` | Intenta convertir a número. Si falla, pone `NaN` |
| `.isna()` | Detecta cuáles fallaron (quedaron como NaN) |
| `self.df['valor'].notna()` | Excluye los que ya eran nulos originalmente |
| El resultado | Filas donde `valor` NO es un número pero tampoco está vacío |

**Ejemplo del problema encontrado:**

| fecha | valor | problema |
|-------|-------|----------|
| 2023-08-12 | 8MIL PESOS | ← Esta fila se detecta como no numérica |

---

```python
def check_duplicate_headers(self) -> pd.DataFrame:
    """Retorna filas que parecen encabezados repetidos del CSV."""
    return self.df[
        self.df['fecha'].astype(str).str.contains(
            'fecha|idCategoria', case=False, na=False
        )
    ]
```

**¿Por qué busca esto?**

A veces los CSV tienen encabezados repetidos en medio del archivo:

```
fecha,valor,descripcion
2023-08-01,5000,comida
fecha,valor,descripcion    ← ¡Esta línea no debería estar!
2023-08-02,3000,transporte
```

El código detecta esas filas problemáticas.

---

## CLASE 3: DataCleaner

### ¿Qué hace?

Aplica las correcciones a los datos.

### ¿Por qué tiene un atributo `self.log`?

Para llevar un registro de qué cambios hizo. Al final, el usuario puede ver un resumen de la limpieza.

---

```python
class DataCleaner:
    """Limpia y estandariza el DataFrame de gastos."""

    def __init__(self):
        self.log = []
```

**¿Qué es `self.log = []`?**
- Una lista vacía donde se guardarán mensajes
- Cada vez que se hace una corrección, se agrega un mensaje

---

```python
def clean(self, df: pd.DataFrame) -> pd.DataFrame:
    """Aplica todas las transformaciones de limpieza en secuencia."""
    df = df.copy()
    df = self._remove_header_rows(df)
    df = self._remove_efectivo_errors(df)
    df = self._fix_8mil(df)
    df = self._convert_types(df)
    df = self._strip_strings(df)
    df = self._rename_columns(df)
    return df
```

**Patrón de diseño: Pipeline**

Cada método modifica el DataFrame y pasa al siguiente:
```
DataFrame original
    → _remove_header_rows (quita encabezados repetidos)
    → _remove_efectivo_errors (quita filas con error)
    → _fix_8mil (corrige "8MIL PESOS")
    → _convert_types (convierte tipos de datos)
    → _strip_strings (quita espacios)
    → _rename_columns (renombrar columnas)
    → DataFrame limpio
```

**¿Por qué métodos privados (con `_`)?**

Los métodos que empiezan con `_` son "privados" por convención. Solo se usan internamente desde `clean()`. El usuario nunca los llama directamente.

---

```python
def _fix_8mil(self, df: pd.DataFrame) -> pd.DataFrame:
    mask = df['valor'].astype(str).str.upper().str.contains('8MIL', na=False)
    df.loc[mask, 'valor'] = 8000
    self.log.append(f'[IMPUTADO] "8MIL PESOS" -> 8000: {mask.sum()} registro(s)')
    return df
```

**Explicación:**

| Línea | Explicación |
|-------|-------------|
| `df['valor'].astype(str)` | Convierte la columna a texto |
| `.str.upper()` | Pone todo en mayúsculas (para buscar sin importar mayúsculas/minúsculas) |
| `.str.contains('8MIL', na=False)` | Busca filas que contienen "8MIL" |
| `df.loc[mask, 'valor'] = 8000` | Reemplaza esos valores por 8000 |
| `self.log.append(...)` | Registra el cambio para el reporte final |

**¿Por qué 8000 y no otro valor?**
- "8MIL PESOS" claramente significa 8,000 pesos colombianos
- Es una decisión de imputación razonable basada en el contexto

---

```python
def _convert_types(self, df: pd.DataFrame) -> pd.DataFrame:
    df['fecha'] = pd.to_datetime(df['fecha'], errors='coerce')
    df['valor'] = pd.to_numeric(df['valor'], errors='coerce')
    return df
```

**¿Por qué convertir tipos?**

Pandas lee todo como texto por defecto. Para análisis necesitamos:
- `fecha` como tipo `datetime` (para hacer comparaciones de fechas)
- `valor` como tipo `float` (para sumar, promediar, etc.)

---

```python
def _strip_strings(self, df: pd.DataFrame) -> pd.DataFrame:
    for col in ['flujo casa mes', 'Forma de Pago', 'idCategoria']:
        if col in df.columns:
            df[col] = df[col].astype(str).str.strip()
    return df
```

**¿Qué hace `.str.strip()`?**

Quita espacios al inicio y final:

| Original | Después de strip |
|----------|------------------|
| `" Mercado "` | `"Mercado"` |
| `"Debito"` | `"Debito"` (sin cambio) |

**¿Por qué es importante?**
- `" Mercado"` y `"Mercado"` son diferentes para la computadora
- Sin esto, el mapeo de categorías fallaría

---

```python
def _rename_columns(self, df: pd.DataFrame) -> pd.DataFrame:
    return df.rename(columns={
        'flujo casa mes' : 'descripcion',
        'Forma de Pago'  : 'forma_pago',
        'idCategoria'    : 'id_categoria',
    })
```

**¿Por qué renombrar?**

| Nombre original | Nombre nuevo | Por qué |
|-----------------|--------------|---------|
| `flujo casa mes` | `descripcion` | Más claro, describe qué contiene |
| `Forma de Pago` | `forma_pago` | Minúsculas y sin espacios (buena práctica) |
| `idCategoria` | `id_categoria` | Convención snake_case |

---

## MAPEO DE CATEGORÍAS

```python
MAPEO_CATEGORIAS = {
    'Contrato papa': 'Contrato papa',
    'CUOTA CASA': 'CUOTA  CASA',
    'MERCADO': 'MERCADO',
    # ... 35 reglas más
}
```

**¿Qué es esto?**

Los archivos de gastos usan nombres diferentes a los del presupuesto:

| Archivo de gastos | Presupuesto |
|-------------------|-------------|
| "CUOTA CASA" | "CUOTA  CASA" (con dos espacios) |
| "TRANSPORTE CARRO / MOTO" | "TRANSPORTE CARRO/MOTO/GASOLINA" |

El diccionario `MAPEO_CATEGORIAS` traduce de un nombre al otro.

**¿Por qué algunos valores son `None`?**

```python
'CUOTAS DE MANEJO': None,   # gasto no presupuestado
'PRESTAMO': None,           # gasto no presupuestado
'RETIRO CAJERO': None,     # gasto no presupuestado
```

Estos gastos no tienen equivalente en el presupuesto. Al poner `None`, el código los marca como "sin rubro" para el análisis posterior.

---

## CLASE 4: DataExporter

```python
class DataExporter:
    """Exporta los datos procesados a data/processed/."""

    def __init__(self, proc_dir: Path):
        self.proc_dir = proc_dir
```

**¿Por qué una clase solo para exportar?**

Porque:
1. Todos los métodos comparten `self.proc_dir`
2. Si después queremos exportar a otros formatos (Parquet, JSON), solo agregamos métodos aquí
3. Mantiene la responsabilidad única: solo exporta, no limpia ni analiza

---

```python
def export_gastos(self, df: pd.DataFrame) -> None:
    cols = ['fecha', 'descripcion', 'valor', 'forma_pago',
            'id_categoria', 'rubro_presupuesto', 'miembro', 'mes_origen']
    df[cols].to_csv(self.proc_dir / 'gastos_clean.csv', index=False)
    print(f'gastos_clean.csv      -> {len(df)} registros')
```

**Explicación:**

| Línea | Explicación |
|-------|-------------|
| `cols = [...]` | Lista de columnas a exportar (en orden específico) |
| `df[cols]` | Selecciona solo esas columnas |
| `.to_csv(..., index=False)` | Guarda como CSV sin incluir el índice (0, 1, 2...) |

**¿Por qué `index=False`?**

Sin esto, el CSV tendría una columna extra:
```
,fecha,descripcion,valor
0,2023-08-01,Mercado,50000
1,2023-08-02,Taxi,20000
```

Con `index=False`:
```
fecha,descripcion,valor
2023-08-01,Mercado,50000
2023-08-02,Taxi,20000
```

---

# NOTEBOOK 02: MODELO RELACIONAL

## ¿Por qué este notebook existe?

Para convertir los CSV limpios en una base de datos relacional que permita:
- Consultas complejas con SQL
- Integridad referencial (no puedes tener un gasto sin miembro)
- Escalar a múltiples familias sin cambiar la estructura

---

## Librerías importadas

```python
from sqlalchemy import (
    create_engine, text,
    Column, Integer, String, Float, Date, UniqueConstraint, ForeignKey
)
from sqlalchemy.orm import declarative_base, Session
```

| Importación | ¿Para qué? |
|-------------|-----------|
| `create_engine` | Crea la conexión a la base de datos |
| `text` | Ejecuta consultas SQL directas |
| `Column, Integer, String...` | Define columnas de las tablas |
| `UniqueConstraint` | Crea restricciones de unicidad |
| `ForeignKey` | Crea relaciones entre tablas |
| `declarative_base` | Base para definir modelos ORM |
| `Session` | Maneja transacciones (insertar, actualizar, etc.) |

---

## ¿Qué es SQLAlchemy ORM?

**ORM** = Object-Relational Mapping

Es una forma de definir tablas como **clases de Python**:

```python
# En lugar de escribir SQL:
# CREATE TABLE miembros (
#     id_miembro INTEGER PRIMARY KEY,
#     nombre TEXT NOT NULL
# );

# Escribes una clase:
class Miembro(Base):
    __tablename__ = 'miembros'
    id_miembro = Column(Integer, primary_key=True)
    nombre = Column(String(50), nullable=False)
```

**Ventajas:**
- Código más legible
- Funciona con SQLite, PostgreSQL, MySQL sin cambios
- Evita inyección SQL automáticamente

---

## CLASE 1: Miembro

```python
class Miembro(Base):
    """Miembros de la familia Miranda."""
    __tablename__ = 'miembros'

    id_miembro = Column(Integer, primary_key=True, autoincrement=True)
    nombre     = Column(String(50),  nullable=False, unique=True)
    tipo       = Column(String(20),  nullable=False)
```

**Explicación de cada columna:**

| Columna | Tipo | Restricciones | Explicación |
|---------|------|---------------|-------------|
| `id_miembro` | Integer | PRIMARY KEY, AUTO | Identificador único (1, 2, 3...) |
| `nombre` | String(50) | NOT NULL, UNIQUE | "papa", "mama", "hijo" |
| `tipo` | String(20) | NOT NULL | Rol en la familia |

**¿Por qué `primary_key=True`?**

La llave primaria identifica cada fila de forma única. Sin ella:
- No podrías relacionar gastos con miembros
- Pandas no sabría cuál registro actualizar

**¿Por qué `autoincrement=True`?**

Para que SQLite asigne automáticamente:
- Primer miembro → id_miembro = 1
- Segundo miembro → id_miembro = 2
- etc.

---

## CLASE 2: Rubro

```python
class Rubro(Base):
    """Rubros del presupuesto familiar."""
    __tablename__ = 'rubros'

    id_rubro     = Column(Integer, primary_key=True, autoincrement=True)
    nombre_rubro = Column(String(100), nullable=False, unique=True)
```

**¿Qué es un rubro?**

Categorías del presupuesto: "MERCADO", "TRANSPORTE", "SERVICIOS", etc.

**¿Por qué tabla separada?**

En lugar de repetir el nombre en cada gasto:

| Sin normalizar | Con normalización |
|----------------|-------------------|
| id_rubro=1, nombre="MERCADO" | id_rubro=1, nombre="MERCADO" (una vez) |
| id_rubro=1, nombre="MERCADO" | gasto con id_rubro=1 |
| id_rubro=1, nombre="MERCADO" | gasto con id_rubro=1 |

Esto ahorra espacio y evita inconsistencias.

---

## CLASE 3: Presupuesto

```python
class Presupuesto(Base):
    """
    Valor planeado por rubro y mes.
    UNIQUE(id_rubro, mes) garantiza un solo valor planeado por rubro por mes.
    """
    __tablename__  = 'presupuesto'
    __table_args__ = (UniqueConstraint('id_rubro', 'mes', name='uq_rubro_mes'),)

    id_presupuesto = Column(Integer, primary_key=True, autoincrement=True)
    id_rubro       = Column(Integer, ForeignKey('rubros.id_rubro'), nullable=False)
    mes            = Column(String(7),  nullable=False)
    valor_planeado = Column(Float,      nullable=False)
```

**Explicación:**

| Columna | Explicación |
|---------|-------------|
| `id_presupuesto` | Llave primaria |
| `id_rubro` | Llave foránea → referencia a `rubros.id_rubro` |
| `mes` | "2023-08", "2023-09" |
| `valor_planeado` | Cuánto se planeó gastar |

**¿Qué es `UniqueConstraint('id_rubro', 'mes')`?**

Garantiza que NO puedas tener:

| id_rubro | mes | valor |
|----------|-----|-------|
| 1 | 2023-08 | 100000 |
| 1 | 2023-08 | 150000 | ← ERROR: Ya existe |

Sin esto, podrías tener dos presupuestos para el mismo rubro y mes, y no sabrías cuál usar.

---

## CLASE 4: Gasto

```python
class Gasto(Base):
    """
    Transacciones de gasto de cada miembro.
    id_rubro es nullable para permitir gastos sin rubro presupuestado.
    """
    __tablename__ = 'gastos'

    id_gasto         = Column(Integer, primary_key=True, autoincrement=True)
    id_miembro       = Column(Integer, ForeignKey('miembros.id_miembro'), nullable=False)
    id_rubro         = Column(Integer, ForeignKey('rubros.id_rubro'),     nullable=True)
    fecha            = Column(Date,    nullable=False)
    descripcion      = Column(String(200))
    valor            = Column(Float,   nullable=False)
    forma_pago       = Column(String(30))
    categoria_origen = Column(String(100))
    mes              = Column(String(7),  nullable=False)
```

**¿Por qué `nullable=True` en `id_rubro`?**

Porque algunos gastos no tienen rubro en el presupuesto:
- CDT (inversión)
- Préstamos
- Retiros de cajero

Si `id_rubro` fuera `NOT NULL`, esos registros **no se podrían insertar**.

---

## CLASE 5: DatabaseLoader

```python
class DatabaseLoader:
    """Carga los datos procesados en las tablas del modelo relacional."""

    def __init__(self, engine, df_gastos: pd.DataFrame, df_ppto: pd.DataFrame):
        self.engine      = engine
        self.df_gastos   = df_gastos
        self.df_ppto     = df_ppto
        self.miembro_ids: dict = {}
        self.rubro_ids:   dict = {}
```

**¿Por qué guardar los IDs en diccionarios?**

Porque SQLAlchemy usa IDs numéricos, pero los DataFrames tienen nombres:

```python
self.miembro_ids = {'papa': 1, 'mama': 2, 'hijo': 3}
self.rubro_ids = {'MERCADO': 1, 'TRANSPORTE': 2, ...}
```

Así podemos buscar rápidamente:
```python
id_miembro = self.miembro_ids['papa']  # → 1
```

---

```python
def _load_miembros(self) -> None:
    data = [
        {'nombre': 'papa', 'tipo': 'papa'},
        {'nombre': 'mama', 'tipo': 'mama'},
        {'nombre': 'hijo', 'tipo': 'hijo'},
    ]
    with Session(self.engine) as session:
        session.bulk_insert_mappings(Miembro, data)
        session.commit()
        self.miembro_ids = {m.nombre: m.id_miembro for m in session.query(Miembro).all()}
    print(f'Miembros insertados : {self.miembro_ids}')
```

**Explicación:**

| Línea | Explicación |
|-------|-------------|
| `data = [...]` | Lista de diccionarios con los datos a insertar |
| `Session(self.engine)` | Abre una sesión de base de datos |
| `bulk_insert_mappings()` | Inserta múltiples filas eficientemente |
| `session.commit()` | Guarda los cambios |
| `session.query(Miembro).all()` | Lee todos los miembros para obtener sus IDs |

---

```python
def validate(self) -> None:
    """Verifica conteos e integridad referencial despues de la carga."""
    # ... código de validación ...
```

**¿Por qué validar después de cargar?**

Para asegurarse de que:
1. Se insertaron todos los registros esperados
2. No hay gastos sin miembro válido (integridad referencial)
3. Los conteos coinciden con los CSV originales

---

# NOTEBOOK 03: ANÁLISIS Y REPORTES

## CLASE 1: FamilyAnalyzer

```python
class FamilyAnalyzer:
    """Ejecuta consultas SQL y cálculos financieros sobre la familia."""

    def __init__(self, engine):
        self.engine = engine
```

**¿Qué hace esta clase?**

Ejecuta todas las consultas SQL necesarias para responder las 10 preguntas de negocio.

---

### Ejemplo de método: ejecución presupuestal

```python
def ejecucion_presupuestal(self, mes: str) -> pd.DataFrame:
    """Retorna planeado vs. real por rubro para un mes específico."""
    query = text("""
        SELECT 
            r.nombre_rubro,
            p.valor_planeado,
            COALESCE(SUM(g.valor), 0) AS valor_real,
            COALESCE(SUM(g.valor), 0) - p.valor_planeado AS diferencia,
            ROUND((COALESCE(SUM(g.valor), 0) - p.valor_planeado) * 100.0 
                  / NULLIF(p.valor_planeado, 0), 2) AS porcentaje_exceso
        FROM presupuesto p
        JOIN rubros r ON p.id_rubro = r.id_rubro
        LEFT JOIN gastos g ON p.id_rubro = g.id_rubro AND p.mes = g.mes
        WHERE p.mes = :mes
        GROUP BY r.nombre_rubro, p.valor_planeado
        ORDER BY porcentaje_exceso DESC
    """)
    with self.engine.connect() as conn:
        return pd.read_sql(query, conn, params={'mes': mes})
```

**Explicación del SQL:**

| Cláusula | Explicación |
|----------|-------------|
| `SELECT` | Elige columnas: rubro, planeado, real, diferencia, % exceso |
| `COALESCE(SUM(g.valor), 0)` | Si no hay gastos, pone 0 (en lugar de NULL) |
| `LEFT JOIN gastos` | Incluye rubros sin gastos |
| `GROUP BY` | Agrupa por rubro para sumar |
| `ORDER BY porcentaje_exceso DESC` | Ordena del mayor al menor exceso |

**¿Por qué `COALESCE`?**

`LEFT JOIN` puede devolver NULL cuando no hay gastos:

| rubro | valor_real |
|-------|------------|
| MERCADO | 50000 |
| VIAJES | NULL ← Sin gastos |

`COALESCE(NULL, 0)` convierte NULL a 0.

---

### Ejemplo de método: flujo de caja

```python
def flujo_caja(self) -> pd.DataFrame:
    """Retorna ingresos, gastos y ahorro por mes."""
    query = text("""
        SELECT 
            mes,
            SUM(CASE WHEN id_rubro IN (
                SELECT id_rubro FROM rubros 
                WHERE nombre_rubro LIKE '%Contrato%'
            ) THEN valor ELSE 0 END) AS ingresos,
            SUM(CASE WHEN id_rubro NOT IN (
                SELECT id_rubro FROM rubros 
                WHERE nombre_rubro LIKE '%Contrato%'
            ) THEN valor ELSE 0 END) AS gastos
        FROM gastos
        GROUP BY mes
        ORDER BY mes
    """)
    # ... resto del código ...
```

**¿Qué hace `CASE WHEN`?**

Es como un IF en SQL:
- Si el rubro es "Contrato", suma a ingresos
- Si no, suma a gastos

---

## CLASE 2: FamilyVisualizer

```python
class FamilyVisualizer:
    """Genera gráficos con Matplotlib y Seaborn."""

    def __init__(self, output_dir: Path):
        self.output_dir = output_dir
        plt.style.use('seaborn-v0_8-whitegrid')
```

**¿Qué hace esta clase?**

Toma los DataFrames de `FamilyAnalyzer` y genera gráficos PNG.

---

```python
def plot_ejecucion(self, df: pd.DataFrame, mes: str) -> None:
    """Gráfico de barras horizontales: planeado vs real."""
    fig, ax = plt.subplots(figsize=(12, len(df) * 0.4 + 2))
    
    # Eje Y: nombres de rubros
    y_pos = range(len(df))
    
    # Barras de valor real
    ax.barh(y_pos, df['valor_real'], color='steelblue', label='Real')
    
    # Barras de valor planeado (más pequeñas, superpuestas)
    ax.barh(y_pos, df['valor_planeado'], color='lightgray', label='Planeado')
    
    ax.set_yticks(y_pos)
    ax.set_yticklabels(df['nombre_rubro'])
    ax.set_xlabel('Valor (COP)')
    ax.legend()
    ax.set_title(f'Ejecución Presupuestal - {mes}')
    
    plt.tight_layout()
    plt.savefig(self.output_dir / f'ejecucion_{mes}.png', dpi=150)
    plt.close()
```

**Explicación de Matplotlib:**

| Línea | Explicación |
|-------|-------------|
| `fig, ax = plt.subplots()` | Crea una figura y un eje |
| `figsize=(12, ...)` | Tamaño del gráfico en pulgadas |
| `ax.barh()` | Dibuja barras horizontales |
| `ax.set_xlabel()` | Etiqueta del eje X |
| `ax.legend()` | Muestra la leyenda |
| `plt.savefig()` | Guarda el gráfico como imagen |
| `plt.close()` | Cierra la figura (libera memoria) |

---

# NOTEBOOK 04: WORD CLOUD

## CLASE: WordCloudAnalyzer

```python
class WordCloudAnalyzer:
    """Analiza las descripciones de gastos para identificar el deporte favorito."""

    COLORMAPS = {'papa': 'Blues', 'mama': 'Oranges', 'hijo': 'Greens'}

    SPORTS_KEYWORDS = {
        'ciclismo' : ['bici', 'bicicleta', 'carrera', 'rigo', 'rigoberto',
                      'ponal', 'porta', 'frenos', 'pastillas', 'ciclismo'],
        'futbol'   : ['futbol', 'thomas', 'galacticos', 'cancha'],
        # ... más deportes
    }
```

**¿Qué es `COLORMAPS`?**

Un diccionario que asigna un color a cada miembro:
- Papá → Azul
- Mamá → Naranja
- Hijo → Verde

**¿Qué es `SPORTS_KEYWORDS`?**

Un diccionario que asocia cada deporte con palabras clave:
- Si aparece "bici", "carrera", "rigo" → ciclismo
- Si aparece "futbol", "cancha" → fútbol

---

```python
def _clean_text(self, descripciones: pd.Series) -> str:
    texto = ' '.join(descripciones.astype(str).str.lower().tolist())
    texto = re.sub(r'[^a-záéíóúüñ\s]', ' ', texto)
    return re.sub(r'\s+', ' ', texto).strip()
```

**¿Qué hace esta función?**

| Línea | Explicación |
|-------|-------------|
| `' '.join(...)` | Une todas las descripciones en un solo texto |
| `.str.lower()` | Convierte todo a minúsculas |
| `re.sub(r'[^a-záéíóúüñ\s]', ' ', texto)` | Quita todo lo que NO sea letra o espacio |
| `re.sub(r'\s+', ' ', texto)` | Reemplaza múltiples espacios por uno solo |

**Ejemplo:**

| Entrada | Salida |
|---------|--------|
| `"Mercado! Compras... 3 items"` | `"mercado compras items"` |

---

```python
def detect_sports(self, miembro: str) -> dict:
    """Cuenta menciones de palabras deportivas."""
    texto = ' '.join(
        self.df[self.df['miembro'] == miembro]['descripcion'].str.lower().tolist()
    )
    result = {}
    for deporte, palabras in self.SPORTS_KEYWORDS.items():
        n = sum(texto.count(p) for p in palabras)
        if n > 0:
            result[deporte] = n
    return result
```

**Explicación:**

| Línea | Explicación |
|-------|-------------|
| Filtrar por miembro | Solo busca en las descripciones de ese miembro |
| `sum(texto.count(p) for p in palabras)` | Cuenta cuántas veces aparece cada palabra clave |
| `if n > 0` | Solo guarda deportes con menciones |

---

# RESUMEN DE PATRONES DE DISEÑO USADOS

## 1. Separación de responsabilidades

Cada clase hace UNA cosa:
- `DataLoader` → Solo carga archivos
- `DataValidator` → Solo diagnostica problemas
- `DataCleaner` → Solo limpia datos
- `DataExporter` → Solo exporta archivos

## 2. Patrón Pipeline

```python
df = cleaner.clean(df)
df = cleaner.apply_mapeo(df, mapeo)
exporter.export_gastos(df)
```

Cada paso transforma los datos y pasa al siguiente.

## 3. Inyección de dependencias

```python
loader = DataLoader(RAW_DIR)  # Se pasa la carpeta como parámetro
```

Esto permite cambiar la configuración sin modificar el código.

## 4. Log de cambios

```python
self.log.append('[IMPUTADO] "8MIL PESOS" -> 8000')
```

Para tener trazabilidad de qué se modificó.

---

# ¿POR QUÉ ESTE DISEÑO?

| Alternativa | Problema | Solución aplicada |
|-------------|----------|-------------------|
| Todo en un archivo | Código difícil de mantener | Separar en 4 notebooks |
| Funciones sueltas | Variables globales, difícil testear | Clases con estado |
| Código procedural | Repetición de código | Métodos reutilizables |
| Sin validación | Errores no detectados | `DataValidator` separado |
| Sin log de cambios | No se sabe qué se modificó | `self.log` en `DataCleaner` |

---

**Fin de la explicación del código**