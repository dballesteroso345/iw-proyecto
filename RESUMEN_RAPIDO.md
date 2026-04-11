# RESUMEN RÁPIDO PARA LA PRESENTACIÓN
## (1 hoja para estudiar antes de entrar)

---

## QUÉ ES EL PROYECTO
Un sistema que analiza los gastos de una familia colombiana (Papá, Mamá, Hijo) durante Agosto y Septiembre 2023.

---

## CÓMO FUNCIONA (en una frase)
```
Archivos crudos → Limpieza → Base de datos → Análisis → Gráficos
```

---

## LAS 4 CAPAS

| Capa | Qué contiene | Dónde está |
|------|-------------|------------|
| Bronze | Archivos originales sin modificar | `data/raw/` |
| Silver | Datos limpios en CSV | `data/processed/` |
| Gold | Base de datos lista para análisis | `data/familia_miranda.db` |

---

## LOS 4 NOTEBOOKS

| Notebook | Qué hace |
|----------|---------|
| 01_ingestion_cleaning | Carga y limpia los archivos, exporta CSV |
| 02_data_model | Crea la base de datos en 3FN |
| 03_analysis | Responde las 10 preguntas de negocio |
| 04_wordcloud_bonus | Detecta el deporte favorito |

---

## RESULTADOS CLAVE (para responder rápido)

| Pregunta | Respuesta |
|----------|-----------|
| ¿Cuánto ganan? | Ago: $19.1M / Sep: $21.6M |
| ¿Cuánto gastan? | Ago: $93.2M / Sep: $21.5M |
| ¿Ahorran? | Ago: NO (déficit $74M) / Sep: SÍ (superávit $75K) |
| ¿Por qué déficit en agosto? | Inversión CDT carro $78M (no recurrente) |
| Top 3 sobreejercicio | VIAJES (+1,256%), TARJETA NUBE (+89%), CUOTA CASA (+13%) |
| Medio pago preferido | Papá: Efectivo / Mamá: Débito / Hijo: Efectivo |
| Deporte favorito | Hijo: Ciclismo / Papá: Ciclismo+Fútbol / Mamá: Caminata |

---

## QUÉ ES 3FN (Tercera Forma Normal)

Es una regla de diseño de bases de datos que:
1. **1FN**: Cada celda tiene un solo valor (no listas)
2. **2FN**: Todo depende de la llave primaria completa
3. **3FN**: Nada depende de otros campos que no sean la llave

**Ejemplo**: El nombre del rubro depende solo del `id_rubro`, no de otras columnas.

---

## QUÉ ES OOP (Programación Orientada a Objetos)

Cada notebook tiene **clases** con una responsabilidad específica:

```
DataLoader    → Solo cargar archivos
DataValidator → Solo diagnosticar problemas
DataCleaner   → Solo limpiar datos
DataExporter  → Solo guardar archivos
```

**Ventaja**: Si hay un error en limpieza, solo se arregla en `DataCleaner`.

---

## QUÉ ES SQLAlchemy

Es un **ORM** (Object-Relational Mapping) que permite:
- Escribir Python en lugar de SQL
- Cambiar de SQLite a PostgreSQL sin modificar código
- Evitar inyección SQL automáticamente

---

## QUÉ ES WORD CLOUD

Una **nube de palabras** que muestra las palabras más frecuentes más grandes. Se usó para detectar intereses de cada persona buscando palabras como:
- "bici", "carrera" → ciclismo
- "futbol" → fútbol

---

## TECNOLOGÍAS (para memorizar)

| Tecnología | Para qué se usó |
|------------|----------------|
| Python | Lenguaje principal |
| Pandas | Manipular datos como tablas |
| SQLAlchemy | Crear y consultar base de datos |
| SQLite | Base de datos sin servidor |
| Matplotlib | Hacer gráficos |
| Jupyter | Notebooks interactivos |

---

## POSIBLES PREGUNTAS DEL ENTREVISTADOR

**P: ¿Por qué nullable en id_rubro?**
R: Porque algunos gastos como el CDT no tenían rubro en el presupuesto. Si obligamos a tener rubro, perderíamos esos registros.

**P: ¿Por qué SQLite y no PostgreSQL?**
R: SQLite no requiere servidor, es portable y suficiente para este caso. Si escalamos, SQLAlchemy permite cambiar fácilmente.

**P: ¿Cómo escalaría esto para más familias?**
R: Agregar una tabla `familias` con `id_familia` y relacionar todos los miembros a una familia.

**P: ¿Qué haría si los archivos tuvieran errores nuevos?**
R: Los errores se manejan en `DataCleaner` con reglas específicas. Agregaría nuevos métodos para nuevos tipos de errores.

---

## ARCHIVOS FINALES

```
data/processed/
├── gastos_clean.csv         (372 registros limpios)
├── presupuesto_clean.csv    (74 registros)
├── mapeo_categorias.csv     (35 reglas de mapeo)
├── ejecucion_presupuestal.png
├── flujo_caja.png
├── top3_sobreejercicio.png
├── medios_pago.png
├── wordcloud_familia.png
└── deportes_familia.png

data/
└── familia_miranda.db       (Base de datos SQLite)
```

---

## FRASES PARA LA PRESENTACIÓN

- "Implementé una arquitectura medallón Bronze-Silver-Gold estándar en ingeniería de datos"
- "El modelo está en Tercera Forma Normal para garantizar integridad referencial"
- "Cada notebook encapsula su lógica en clases con responsabilidad única"
- "Todas las preguntas de negocio fueron respondidas con consultas SQL"
- "El sistema está listo para escalar y conectarse a Power BI"

---

¡ÉXITO EN LA PRESENTACIÓN!