| [← README](README_es_multi-doc.md) | [Technical Specs](01_Technical_Specs.md) | 📄 Business Logic | [User Journey](03_User_Journey.md) |
|:---:|:---:|:---:|:---:|

&nbsp;

# PEM - Business Logic Specifications
## Lógica de Negocio

> **Proyecto:** PurchaseEditorManager (PEM)  
> **Audiencia:** Business Analysts, Product Owners, Planificadores Senior  
> **Versión:** V 800.200514 Beta

---

## Índice

1. [Conceptos Fundamentales](#1-conceptos-fundamentales)
2. [Jerarquía de Datos de Negocio](#2-jerarquía-de-datos-de-negocio)
3. [Procesamiento de Propuestas](#3-procesamiento-de-propuestas)
4. [Netificación de Cantidades](#4-netificación-de-cantidades)
5. [Sistema de Sustitución y Compensación](#5-sistema-de-sustitución-y-compensación)
6. [Redes Logísticas](#6-redes-logísticas)
7. [Centralización por Lotes y por Compra](#7-centralización-por-lotes-y-por-compra)
8. [Estados de Necesidad: MANP3, MANP4, MANP5](#8-estados-de-necesidad-manp3-manp4-manp5)
9. [Reglas de Compensación (RuleEquivalences)](#9-reglas-de-compensación-ruleequivalences)
10. [Restricciones de Proveedor](#10-restricciones-de-proveedor)
11. [Proceso de Importación](#11-proceso-de-importación)
12. [Proceso de Exportación](#12-proceso-de-exportación)
13. [Glosario de Negocio](#13-glosario-de-negocio)

---

## 1. Conceptos Fundamentales

### ¿Qué problema resuelve PEM?

El sistema de planificación **SO99+** genera propuestas de compra automáticas basadas en previsiones de demanda. Sin embargo, estas propuestas no tienen en cuenta:

- **Stock real sobrante** en otros almacenes que podría usarse en lugar de comprar
- **Productos equivalentes o sustitutos** que podrían reemplazar al producto propuesto
- **Mínimos de compra/logística** exigidos por los proveedores
- **Reglas de negocio** específicas del cliente sobre qué productos pueden intercambiarse
- **Centralización** de pedidos para optimizar costes

**PEM** toma las propuestas de SO99, las enriquece con datos maestros del ERP, aplica reglas de negocio configurables, y produce órdenes de compra y traspaso optimizadas listas para enviar al ERP.

### Flujo de negocio de alto nivel

```
SO99 genera propuestas → PEM importa → PEM calcula (netifica, sustituye, centraliza)
→ Planificador revisa problemas → Planificador aprueba → PEM exporta al ERP
```

---

## 2. Jerarquía de Datos de Negocio

PEM organiza las propuestas en una jerarquía de 4 niveles que refleja la estructura organizativa de compras:

```
Plangroup (Grupo de Planificación)
│   Agrupa productos que se revisan con la misma frecuencia
│   Ej: "LÁCTEOS - DIARIO", "CONGELADOS - SEMANAL"
│
└── Supplier (Proveedor)
    │   Agrupa todas las propuestas de un mismo proveedor
    │   Ej: "DANONE", "NESTLÉ"
    │
    └── Warehouse (Almacén)
        │   Agrupa propuestas para un mismo almacén de destino
        │   Ej: "VAL4" (Valencia), "MAD1" (Madrid)
        │
        └── Proposal (Propuesta)
            Producto específico con cantidad, fecha, coste
            Cada propuesta tiene un IdxProp:
            • Idx=0: Propuesta principal
            • Idx=1: Alternativa 1
            • Idx=2: Alternativa 2
```

### ¿Qué es un Plangroup?

Un **Plangroup** (grupo de planificación) define:
- **Frecuencia de planificación**: cada cuántos días se revisa (diario, semanal, etc.)
- **Tipo de centralización**: ACCUM (centraliza) o NO_ACCUM (no centraliza)
- **Nivel de centralización**: REG (vía regional) o CEN (directo a central)

Ejemplo: El Plangroup "LÁCTEOS_DIARIO" se planifica cada día, centraliza vía regional (REG).

---

## 3. Procesamiento de Propuestas

### Flujo Completo de Cálculo

```
┌──────────────────────────────────────────────────────────────┐
│ FASE 1: TransferOrders SO99                                  │
│ • PEM importa TransferOrders generados por SO99              │
│ • Vincula cada TransferOrder con su propuesta de compra      │
│ • Netifica: descuenta la cantidad traspasada de la compra    │
└──────────────────────────────────────────────────────────────┘

→ [Flujos Detallados: ejemplo completo](Flujos_Detallados.md#sec1)

                              ↓
┌──────────────────────────────────────────────────────────────┐
│ FASE 2: Sustituciones Locales (mismo almacén)                │
│ • Para cada propuesta con cantidad pendiente > 0:            │
│   a) Buscar sustitutos (S) en el mismo almacén               │
│      → Items obsoletos que pueden reemplazar al pedido       │
│   b) Buscar equivalentes (D) en el mismo almacén             │
│      → Items diferentes pero intercambiables                 │
│ • Validar contra reglas de compensación                      │
│ • Netificar: descontar stock encontrado de la propuesta      │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ FASE 3: Sustituciones en Red (toda la red logística)         │
│ • Para cada propuesta con cantidad pendiente > 0:            │
│   a) Buscar sustitutos (S) en TODA la red logística          │
│   b) Buscar IGUALES (I) en TODA la red                       │
│      → Mismo producto en otro almacén con sobrestock         │
│   c) Buscar equivalentes (D) en TODA la red                  │
│ • Validar contra reglas de compensación                      │
│ • Netificar                                                   │
│ • Generar TransferOrders para los traspasos entre almacenes  │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ FASE 4: Mínimos de Entrega                                   │
│ • Evaluar cada propuesta contra:                             │
│   - Lote mínimo de movimiento (MANP3)                        │
│   - Compra mínima del proveedor (MANP4)                      │
│   - Portes mínimos (MANP5)                                   │
│ • Marcar propuestas que no cumplen para revisión manual      │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ FASE 5: Centralización                                       │
│ • Para proveedores tipo ACCUM:                               │
│   - Agrupar todas las propuestas en el almacén regional      │
│   - Generar TransferOrders del regional a cada destino       │
│ • Para proveedores tipo NO_ACCUM:                            │
│   - Mantener propuestas en sus almacenes originales          │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│ FASE 6: Promociones y Descuentos                             │
│ • Aplicar promociones vigentes de proveedores                │
│ • Ajustar costes según rappels y descuentos                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Netificación de Cantidades

### ¿Qué es la netificación?

La **netificación** es el proceso de reducir la cantidad a comprar de una propuesta utilizando el stock disponible de productos sustitutos, iguales o equivalentes en otros almacenes.

### Algoritmo

```
Para una propuesta con QtyFinal = 100 unidades:

1. Buscar candidatos (sustitutos/equivalentes con stock)
2. Ordenar candidatos por MAYOR stock equivalente disponible
3. Para cada candidato, en orden:

   Candidato A: 60 unidades en stock → EquivalentStock = 60
   QtyFinal (100) >= EquivalentStock (60)?
   → Sí → candidate.EquivalentStockUsed = 60
           QtyFinal = 100 - 60 = 40

   Candidato B: 50 unidades en stock → EquivalentStock = 50
   QtyFinal (40) >= EquivalentStock (50)?
   → No → candidate.EquivalentStockUsed = 40
           QtyFinal = 40 - 40 = 0

4. Resultado: QtyFinal = 0 (totalmente compensada)
   • Se compran 0 unidades
   • Se genera TransferOrder de 60 uds desde almacén A
   • Se genera TransferOrder de 40 uds desde almacén B
```

### ¿Qué stock se usa?

Depende de la regla de compensación aplicada:

| Tipo de Stock | Significado | Cuándo se usa |
|---|---|---|
| **TotalStock** | Todo el stock virtual del almacén | Cuando la regla permite usar todo el stock |
| **OverStock** | Solo el stock que excede la demanda proyectada | Cuando la regla es conservadora (solo stock "sobrante") |

El OverStock se calcula como: `min(OverStkQty_AvedemQty, OverStkQty_Projected)`

Además, el sistema verifica que el stock proyectado (a futuro, dentro del leadTime) esté por encima del MaxOutflow (demanda máxima esperada), para asegurar que no se está desabasteciendo el almacén de origen.

---

## 5. Sistema de Sustitución y Compensación

### 5.1 Tres Tipos de Equivalencia

| Tipo | Código | Significado | Fuente de datos | Ejemplo |
|---|---|---|---|---|
| **Sustituto** | **S** | Producto obsoleto que es reemplazado por uno nuevo | `IN_ITEM_SUPERSSESIONS` (SO99) | Item "123" (descatalogado) → Item "456" (nuevo) |
| **Igual** | **I** | El MISMO producto en otro almacén | `ItemLocationList` (stock en otros whouses) | Item "789" en VAL4 → Item "789" en MAD1 |
| **Equivalente** | **D** | Producto DIFERENTE pero funcionalmente intercambiable | `IN_ITEM_EQUIVALENCES` (ERP) | Leche Marca A → Leche Marca B |

### 5.2 Orden de Búsqueda

El sistema busca candidatos en este orden, pasando al siguiente tipo solo si la propuesta aún tiene cantidad pendiente:

```
LOCAL (mismo almacén):
  1º Sustitutos (S) → ¿QtyFinal > 0?
  2º Equivalentes (D) → ¿QtyFinal > 0?

NETWORK (toda la red logística):
  3º Sustitutos (S) → ¿QtyFinal > 0?
  4º Iguales (I) → ¿QtyFinal > 0?
  5º Equivalentes (D) → Fin
```

**Nota:** Los Iguales (I) solo se buscan en Network, no en Local (no tiene sentido traspasar el mismo producto dentro del mismo almacén).

### 5.3 Elegibilidad de un Candidato

Un candidato es elegible si cumple TODAS estas condiciones:

1. **Stock suficiente**: `stockTotal > 0`
2. **Lote mínimo**: `stockTotal >= minResLot` (el stock debe cubrir al menos el lote mínimo de traslado)
3. **Nivel de almacén válido**: El nivel (N0, N1, N2, N3) no es `NoLevel`
4. **Validación por regla de compensación**: Al menos una regla permite la operación
5. **Stock equivalente suficiente**: `EquivalentStock >= minResLot` (tras aplicar Total/OverStock)

---

## 6. Redes Logísticas

### 6.1 Estructura Jerárquica

Cada producto pertenece a una **red logística** que define la jerarquía de almacenes:

```
Central (CEN)
  │
  ├── Regional "ESTE" (REG)
  │   ├── VAL4 (Valencia)
  │   ├── ALI1 (Alicante)
  │   └── CAS1 (Castellón)
  │
  ├── Regional "CENTRO" (REG)
  │   ├── MAD1 (Madrid)
  │   └── TOL1 (Toledo)
  │
  └── Regional "SUR" (REG)
      ├── SEV1 (Sevilla)
      └── MAL1 (Málaga)
```

Los datos de la red logística vienen de `IN_RED_LOG` (importado desde ERP).

### 6.2 Niveles entre Almacenes

Cuando se evalúa si un producto puede moverse del almacén A al almacén B, se determina el **nivel**:

| Nivel | Significado | Ejemplo |
|---|---|---|
| **N0** | Mismo almacén | VAL4 → VAL4 |
| **N1** | Misma región | VAL4 → ALI1 (ambos en regional ESTE) |
| **N2** | Distinta región (destino regional) | MAD1 → VAL4 (regionales diferentes) |
| **N3** | Resto de almacenes | MAD1 → ALI1 (almacén no regional desde otra región) |
| **NoLevel** | Sin relación | No comparten red logística |

### 6.3 ScopeDistribution (Ámbito de Distribución)

Cada producto tiene un ámbito que limita hasta dónde puede moverse:

| ScopeDistribution | Significado | ¿Puede usar N2? | ¿Puede usar N3? |
|---|---|---|---|
| `_ALL` | Puede usar cualquier almacén | ✅ Sí | ✅ Sí |
| `_ALL_R` | Puede usar regionales + resto | ✅ Sí (si dest es regional) | ✅ Sí (si dest no es regional) |
| `_REG` | Solo su propia región | ❌ No | ❌ No |

Proviene del campo `FREEATTR6` en el MasterFile (importado de SO99).

---

## 7. Centralización por Lotes y por Compra

### 7.1 ¿Por qué centralizar?

Muchos proveedores exigen:
- **Pedido mínimo** por almacén (ej: mínimo 100 unidades)
- **No separar pedidos**: entregan todo en un solo almacén

Si un almacén necesita 30 unidades y otro 40, por separado no llegan al mínimo. **Centralizando** (30+40=70, más cerca del mínimo), se puede negociar mejor o al menos cumplir restricciones.

### 7.2 Tipos de Centralización

| Tipo | Comportamiento |
|---|---|
| **ACCUM** | El proveedor NO separa pedidos. Todo se entrega en el almacén regional/central. PEM genera TransferOrders automáticos del regional a cada almacén destino |
| **NO_ACCUM** | El proveedor SÍ acepta pedidos separados por almacén. No se centraliza |
| **ALL_WHOUSE** | El proveedor entrega directamente en cada almacén. Sin restricciones |

### 7.3 Niveles de Centralización

| Nivel | Ruta |
|---|---|
| **REG** | Proveedor → Regional → (TransferOrder) → Almacén destino |
| **CEN** | Proveedor → Central → (TransferOrder) → Regional → (TransferOrder) → Almacén destino |

### 7.4 Ejemplo

```
Proveedor DANONE, Plangroup LÁCTEOS, tipo ACCUM, nivel REG:

Sin centralizar:              Centralizado:
VAL4: 30 uds (no llega)       REG_ESTE: 110 uds (¡llega al mínimo de 100!)
ALI1: 40 uds (no llega)       → TransferOrder REG_ESTE→VAL4: 30 uds
CAS1: 40 uds (no llega)       → TransferOrder REG_ESTE→ALI1: 40 uds
                               → TransferOrder REG_ESTE→CAS1: 40 uds
```

                               → [Flujos Detallados: centralización](Flujos_Detallados.md#sec5)

---

## 8. Estados de Necesidad: MANP3, MANP4, MANP5

### 8.1 Tabla de Estados

| Estado | Código | Color | Significado de negocio |
|---|---|---|---|
| **NO_PROCESSED** | 0 | ⬜ Gris | Pendiente de calcular |
| **MANP3** | 30 | 🔴 Rojo | **Falta lote mínimo de movimiento** |
| **MANP3_V** | 31 | 🔴 | MANP3 revisado por el planificador |
| **MANP4** | 41 | 🟠 Naranja | **Falta compra mínima del proveedor** |
| **MANP4_V** | 42 | 🟠 | MANP4 revisado por el planificador |
| **MANP5** | 51 | 🟡 Amarillo | **Falta mínimo de portes** |
| **MANP5_V** | 52 | 🟡 | MANP5 revisado |
| **MANP5_C** | 53 | 🟡 | Centralizado (se gestiona en el central) |
| **TO_EXPORT** | 90 | 🔵 Azul | Aprobado, listo para exportar al ERP |
| **EXPORTED** | 95 | 🟣 Lila | Ya enviado al ERP |

### 8.2 MANP3 — Lote Mínimo de Movimiento

**Problema:** La cantidad neta necesaria no alcanza el **lote mínimo** definido para ese producto en ese almacén.

**Causa:** Cada producto/almacén tiene un `minResLot` (lote mínimo de reserva) y un `incresLot` (incremento de lote). Si `QtyFinal < minResLot`, no se puede cursar el pedido.

**Qué ve el planificador:**
- Producto, almacén, cantidad necesaria (ej: 15 uds)
- Lote mínimo requerido (ej: 50 uds)
- Opciones de acción:
  - **BUY**: Forzar compra de 50 uds (asume sobrecoste)
  - **NO_BUY**: No comprar (asume riesgo de rotura)
  - **TRANSFER**: Buscar traspaso desde otro almacén

### 8.3 MANP4 — Compra Mínima del Proveedor

**Problema:** La cantidad neta no alcanza el **mínimo de compra** que exige el proveedor para esa ubicación.

**Causa:** Restricción en `IN_SUPP_LOCATION_COMPRA` (mínimo de pedido por proveedor+ubicación).

**Qué ve el planificador:**
- Cantidad necesaria vs mínimo del proveedor
- Opciones:
  - **BUY**: Forzar compra del mínimo
  - **NO_BUY**: No comprar
  - **CEN_NO_BUY**: Centralizar (agrupar con otros almacenes) para llegar al mínimo

### 8.4 MANP5 — Portes Mínimos

**Problema:** La cantidad no alcanza el **mínimo de portes** (gastos de envío).

**Causa:** Restricción en `IN_SUPP_LOCATION_PORTES`.

**Qué ve el planificador:**
- Similar a MANP4 pero relacionado con costes de transporte
- Opciones:
  - **BUY**: Asumir el coste de portes
  - **NO_BUY**: No comprar
  - **CEN_NO_BUY**: Centralizar para compartir portes

### 8.5 Acciones del Planificador (Restriction_Action)

| Acción | Significado | Efecto |
|---|---|---|
| **ADVANCE** (0) | Adelantar | Mover cantidad a un período futuro (consolidar) |
| **BUY** (1) | Comprar | Forzar la compra ignorando la restricción |
| **NO_BUY** (2) | No comprar | Eliminar la necesidad |
| **TRANSFER** (3) | Traspasar | Cubrir con stock de otro almacén |
| **CEN_NO_BUY** (4) | Centralizar sin comprar | Agrupar en central para llegar al mínimo |

---

## 9. Reglas de Compensación (RuleEquivalences)

### 9.1 ¿Qué es una Regla de Compensación?

Una regla de compensación define **bajo qué condiciones** un producto (origen) puede usarse para cubrir la necesidad de otro producto (destino).

Las reglas se configuran en la tabla `IN_RULE_EQUIVALENCES` (importadas desde ERP) y son editables desde PEM mediante `RuleCompensaDialog`.

### 9.2 Estructura de una Regla

| Campo | Pregunta que responde | Ejemplo |
|---|---|---|
| `whouse_use` | ¿Mismo almacén (1) o distinto (2)? | 2 (entre almacenes) |
| `typeEquiv` | ¿Sustituto (S), Igual (I) o Equivalente (D)? | D |
| `whouseOri` | ¿Desde qué almacén se puede ceder stock? | "ALL" (cualquiera) |
| `whouseDest` | ¿A qué almacén se puede enviar? | "VAL4" (solo a Valencia) |
| `aplicationType` | ¿A qué nivel de producto aplica? | FAMI (familia) |
| `aplicationCode` | ¿Qué código concreto? | "LACTEOS" |
| `class_ABC_gen` | ¿Qué clasificación ABC general? | "A" (solo productos top) |
| `class_ABC_SKU` | ¿Qué clasificación ABC del SKU? | "" (sin restricción) |
| `expensive` | ¿Productos caros (S), baratos (N) o todos (ALL)? | "ALL" |
| `travel` | ¿El producto puede viajar entre almacenes? | "S" (sí) |
| `TotalStock` | ¿Usar todo el stock (true) o solo OverStock (false)? | false (solo sobrestock) |

### 9.3 Ejemplo de Regla

```
Regla "COMP_LACTEOS_REG":
  whouse_use = 2 (entre almacenes)
  typeEquiv = D (equivalentes)
  whouseOri = ALL
  whouseDest = ALL
  aplicationType = FAMI
  aplicationCode = LACTEOS
  class_ABC_gen = (vacío)
  class_ABC_SKU = (vacío)
  expensive = ALL
  travel = S
  TotalStock = false (solo OverStock)

Significado:
"Productos equivalentes (D) de la familia LÁCTEOS pueden compensarse
entre cualquier almacén, usando solo el sobrestock (no todo el stock),
siempre que el producto pueda viajar."
```

### 9.4 Algoritmo de Validación

```
Para una propuesta y un candidato:

1. Filtrar TODAS las reglas que:
   • Coinciden en whouse_use (1=local, 2=red)
   • Coinciden en typeEquiv (S, I, D)

2. Para cada regla, verificar en cascada:
   ✅ ¿El almacén de origen del candidato está permitido?
   ✅ ¿El almacén de destino de la propuesta está permitido?
   ✅ ¿La clasificación del producto coincide? (sección/línea/familia/marca...)
   ✅ ¿La clasificación ABC general del candidato coincide?
   ✅ ¿La clasificación ABC del SKU coincide?
   ✅ ¿La restricción de caro/barato coincide?

3. De las reglas que cumplen, seleccionar LA MÁS RESTRICTIVA
   (la que tiene más filtros específicos informados)

4. Con la regla ganadora:
   • Si es local (whouseUse=1) → OK
   • Si es red (whouseUse=2) → verificar que el producto "viaja"
   • Si TotalStock=true → OK
   • Si TotalStock=false (OverStock) → verificar que hay sobrestock real:
     - OverStkQty_AvedemQty > 0
     - Stock proyectado > MaxOutflow en el leadTime
```

### 9.5 ¿Cómo se mide la "restrictividad"?

Cada regla obtiene una puntuación según los filtros que aplica:

| Filtro informado | Puntos |
|---|---|
| `class_ABC_gen` no vacío | +1 |
| `class_ABC_SKU` no vacío | +2 |
| `aplicationCode` no vacío | + valor de aplicationType (1 a 8) |
| `whouseDest` específico (no "ALL") | +4 |
| `whouseOri` específico (no "ALL") | +4 |

**Ejemplo de puntuación:**
- Regla A: `aplicationCode="LACTEOS"` (FAMI=3), `whouseDest="VAL4"` (+4) → 7 puntos
- Regla B: `aplicationCode="LACTEOS"` (FAMI=3) → 3 puntos
- **Gana la Regla A** por ser más específica

→ [Flujos Detallados: algoritmo de validación](Flujos_Detallados.md#sec3)

---

## 10. Restricciones de Proveedor

### 10.1 Tipos de Restricciones

| Tipo | Tabla | Significado |
|---|---|---|
| **Movimiento (MANP3)** | `IN_SUPP_ITEM_MOV` | Lote mínimo, incremento, stock de seguridad por proveedor+item |
| **Compra (MANP4)** | `IN_SUPP_LOCATION_COMPRA` | Pedido mínimo (valor y unidades) por proveedor+ubicación |
| **Portes (MANP5)** | `IN_SUPP_LOCATION_PORTES` | Mínimo de portes por proveedor+ubicación |
| **Promociones** | `IN_SUPP_LOCATION_PROMOTIONS` | Descuentos, rappels, bonificaciones |

### 10.2 Búsqueda Jerárquica

Para compras mínimas, el sistema busca en este orden:

```
1. Buscar restricción para Supplier + Location específicos
   Ej: "DANONE_VAL4"
   
2. Si no existe, buscar restricción para Supplier + ALL
   Ej: "DANONE_ALL" (aplica a todas las ubicaciones del proveedor)

3. Si existe para Supplier+Location, seleccionar la mejor restricción
   que aplique al item concreto (según segmentación, familia, etc.)
```

---

## 11. Proceso de Importación

### ¿Qué se importa y de dónde?

| Origen | Datos | Frecuencia |
|---|---|---|
| **SO99+** | Propuestas, MasterFile, ItemLocations, Suppliers, Warehouses, SuperSessions, Plangroups | Cada día de planificación |
| **ERP** | Equivalencias, Segmentaciones, Stock Implantación, Reglas, Red Logística, Restricciones de Proveedor | Bajo demanda ("Cargar Info ERP") |

### Flujo de Importación desde SO99

```
1. Determinar qué Plangroups toca planificar hoy
   → Según frecuencia de planificación de cada Plangroup

2. Backup de seguridad
   → Propuestas actuales → BACK_IN_PROPOSALS1_DATE
   → Logs → BACK_LOGS

3. Limpieza de datos obsoletos
   → Propuestas ya procesadas (STATUS='P')
   → TransferOrders antiguos
   → Logs con más de N días

4. Importar datos maestros
   → Productos, ubicaciones, proveedores, almacenes

5. Importar propuestas del día
   → Solo de los Plangroups que tocan hoy
   → Solo propuestas no procesadas

6. Importar TransferOrders
   → Traspasos entre almacenes planificados por SO99
```

---

## 12. Proceso de Exportación

### ¿Qué se exporta?

| Tabla | Contenido | Destino |
|---|---|---|
| `OUT_PURCHASE_ORDERS` | Órdenes de compra finales | ERP (INTER_SO99) |
| `OUT_TRANSFER_ORDERS` | Órdenes de traspaso entre almacenes | ERP |
| `OUT_SUBSTITUTIONS` | Sustituciones aplicadas | ERP |

### Flujo de Exportación

```
1. Planificador marca Plangroups como "To Export"
   → Las propuestas pasan a estado TO_EXPORT (90)

2. Planificador pulsa "Exportar"

3. PEM:
   a) Limpia tablas OUT_* (TRUNCATE)
   b) Inserta en OUT_PURCHASE_ORDERS:
      - Solo propuestas con QtyFinal > 0
      - Solo propuestas en estado TO_EXPORT
      - Incluye: item, descripción, almacén, proveedor, fechas, cantidades
   c) Inserta en OUT_TRANSFER_ORDERS:
      - TransferOrders generados por PEM
   d) Inserta en OUT_SUBSTITUTIONS:
      - Registro de qué stock se usó para compensar cada propuesta

4. Ejecuta Stored Procedure de exportación
   → Mueve datos de tablas OUT a tablas equivalentes en INTER_SO99

5. Las propuestas pasan a estado EXPORTED (95)

6. El ERP recoge los datos de INTER_SO99 y genera:
   - Pedidos de compra reales
   - Órdenes de traspaso reales
```

---

## 13. Glosario de Negocio

| Término | Definición |
|---|---|
| **SO99+** | Sistema de planificación de demanda que genera propuestas de compra automáticas |
| **PEM** | Purchase Editor Manager — esta aplicación |
| **ERP** | Sistema de gestión empresarial destino (SAP, Navision, etc.) |
| **Plangroup** | Grupo de planificación: conjunto de productos revisados con la misma frecuencia |
| **Propuesta** | Sugerencia de compra para un producto en un almacén, generada por SO99 |
| **Netificación** | Reducir la cantidad a comprar usando stock disponible de otras fuentes |
| **Sustituto (S)** | Producto obsoleto/descatalogado que es reemplazado por el producto de la propuesta |
| **Igual (I)** | El mismo producto en otro almacén con stock sobrante |
| **Equivalente (D)** | Producto diferente pero funcionalmente intercambiable |
| **Candidato** | Producto+almacén que tiene stock y podría usarse para compensar una propuesta |
| **MANP3** | Necesidad que no alcanza el lote mínimo de movimiento |
| **MANP4** | Necesidad que no alcanza la compra mínima del proveedor |
| **MANP5** | Necesidad que no alcanza el mínimo de portes |
| **Centralización** | Agrupar pedidos de varios almacenes en uno superior para cumplir mínimos |
| **ACCUM** | Tipo de Plangroup que requiere centralización (proveedor no separa pedidos) |
| **Red Logística** | Jerarquía de almacenes: Central → Regional → Local |
| **TransferOrder** | Orden de traspaso de mercancía entre dos almacenes |
| **OverStock** | Stock que excede la demanda proyectada — "stock sobrante" disponible para traspasar |
| **TotalStock** | Todo el stock virtual del almacén, incluyendo el necesario para demanda propia |
| **ScopeDistribution** | Ámbito geográfico hasta dónde puede distribuirse un producto |
| **ABC** | Clasificación de productos por importancia: A (top), B (medio), C (bajo) |
| **SKUL** | SKU + Location: identificador compuesto de producto en un almacén concreto |
| **LeadTime** | Tiempo que tarda en llegar un pedido desde que se emite |
| **BulkInsert** | Inserción masiva de datos usando `SqlBulkCopy` para máximo rendimiento |