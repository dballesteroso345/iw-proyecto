# Inconsistencias Encontradas en los Datos

## Fecha de Revisión: 2026-04-08

---

## 1. PROBLEMAS CRÍTICOS DE DATOS

### 1.1 Registros Duplicados
**Causa:** Transacciones que aparecen en múltiples archivos fuente (Papa y Mama registran los mismos gastos compartidos)

**Impacto:**
- 113 registros duplicados
- Valor inflado: $45,126,795 COP

**Ejemplos de duplicados:**
| Descripción | Fecha | Valor | Veces |
|-------------|-------|-------|-------|
| compra carro | 2023-08-28 | $39,000,000 | 2 |
| Pago hipoteca y abono | 2023-08-04 | $1,600,000 | 2 |
| Techai aeropuerto | 2023-08-03 | $68,800 | 2 |

### 1.2 Contratos Clasificados como Gastos
**Problema:** Los contratos (ingresos) están en la tabla `gastos` en lugar de ser ingresos.

**Impacto:**
- Agosto: $19.1M contados como gastos
- Septiembre: $21.6M contados como gastos
- Total: $40.6M mal clasificado

---

## 2. DISCREPANCIAS README vs DATOS REALES

| Métrica | README | Datos Reales | Diferencia |
|---------|--------|--------------|------------|
| Gastos Agosto | $93.2M | $51.3M | -$41.9M |
| Gastos Septiembre | $21.5M | $18.3M | -$3.2M |
| Ahorro Agosto | -$74M | -$32.2M | +$41.8M |
| Ahorro Septiembre | +$75K | +$3.3M | +$3.2M |

---

## 3. ANÁLISIS CORRECTO (Sin Duplicados)

### AGOSTO 2023
```
Ingresos (Contratos):     $19,074,000
Gastos Operativos:        $12,162,586
Inversiones (CDT Carro):  $39,157,432
-----------------------------------------
Flujo Neto:              -$32,245,018  (déficit)
```

### SEPTIEMBRE 2023
```
Ingresos (Contratos):     $21,574,000
Gastos Operativos:        $18,291,628
-----------------------------------------
Flujo Neto:              +$3,282,372  (superávit)
```

---

## 4. CAUSA RAÍZ

Los archivos fuente tienen gastos compartidos registrados en múltiples archivos:
- `Gastos_Papa_202308.txt` y `Gastos_Mama_202308.txt` contienen las mismas transacciones de gastos familiares compartidos (viajes, comida, hipoteca, etc.)

La lógica de ingesta no detectó ni eliminó estos duplicados.

---

## 5. RECOMENDACIONES

1. **Eliminar duplicados:** Crear clave única (fecha + descripción + valor) para detectar y eliminar duplicados
2. **Reclasificar contratos:** Mover contratos de `gastos` a una tabla de ingresos o separarlos en el análisis
3. **Actualizar README:** Corregir los valores de gastos y flujo de caja
4. **Regenerar visualizaciones:** Los gráficos actuales muestran valores inflados

---

## 6. ARCHIVOS AFECTADOS

- `data/processed/gastos_clean.csv` - Contiene duplicados
- `data/familia_miranda.db` - Cargada con duplicados
- `README.md` - Valores incorrectos
- `PRESENTACION.md` - Valores incorrectos
- `RESUMEN_RAPIDO.md` - Valores incorrectos
- Gráficos PNG en `data/processed/` - Datos incorrectos