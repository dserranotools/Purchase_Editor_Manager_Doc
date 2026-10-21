| [← README](README_es_multi-doc.md) | [Technical Specs](01_Technical_Specs.md) | [Business Logic](02_Business_Logic.md) | 📄 User Journey |
|:---:|:---:|:---:|:---:|

&nbsp;

# PEM - User Journey
## Flujo del Planificador

> **Proyecto:** PurchaseEditorManager (PEM)  
> **Audiencia:** Planificadores, Jefes de Compras, Formadores, Product Owners  
> **Versión:** V 800.200514 Beta

---

## Índice

1. [Resumen del día a día del Planificador](#1-resumen-del-día-a-día-del-planificador)
2. [Paso 1: Inicio de PEM](#2-paso-1-inicio-de-pem)
3. [Paso 2: Importación desde SO99](#3-paso-2-importación-desde-so99)
4. [Paso 3: Cálculo Automático](#4-paso-3-cálculo-automático)
5. [Paso 4: Visualización de Resultados](#5-paso-4-visualización-de-resultados)
6. [Paso 5: Revisión y Resolución de Problemas](#6-paso-5-revisión-y-resolución-de-problemas)
7. [Paso 6: Operaciones Manuales](#7-paso-6-operaciones-manuales)
8. [Paso 7: Aprobación y Exportación al ERP](#8-paso-7-aprobación-y-exportación-al-erp)
9. [Escenarios Típicos](#9-escenarios-típicos)
10. [Atajos y Consejos](#10-atajos-y-consejos)
11. [Preguntas Frecuentes](#11-preguntas-frecuentes)

---

## 1. Resumen del día a día del Planificador

```
┌──────────────────────────────────────────────────────┐
│                   DÍA DEL PLANIFICADOR               │
├──────────────────────────────────────────────────────┤
│  08:00  Abrir PEM                                    │
│  08:05  Importar propuestas de SO99                  │
│  08:10  Cargar Info ERP (si hay cambios)             │
│  08:15  Ejecutar cálculos automáticos                │
│  08:20  Revisar el árbol → Ver colores               │
│  08:30  Resolver MANP3 (rojos)                       │
│  09:00  Resolver MANP4 (naranjas)                    │
│  09:30  Resolver MANP5 (amarillos)                   │
│  10:00  Revisar propuestas OK (verdes)               │
│  10:15  Recalcular si hubo cambios manuales          │
│  10:30  Marcar para exportar                         │
│  10:35  Exportar al ERP                              │
│  10:40  Revisar Logs → Confirmar exportación OK      │
└──────────────────────────────────────────────────────┘
```

> 💡 **Nota:** Los tiempos son orientativos. Un planificador experimentado con pocos Plangroups puede completar el ciclo en 30-45 minutos.

---

## 2. Paso 1: Inicio de PEM

### Lo que ve el planificador

1. **Arranca la aplicación** desde el acceso directo
2. La aplicación carga la configuración y prueba la conexión a las 3 bases de datos
3. **Si la conexión falla**, aparece una ventana de configuración:
   - Permite cambiar servidor, base de datos, usuario y contraseña
   - Soporta autenticación Windows (campos usuario/contraseña vacíos)
4. Una vez conectado, se abre la **pantalla principal**

### Pantalla Principal (MainFormNEW)

```
┌─────────────────────────────────────────────────────────────┐
│ [Toolbar: Importar | Calcular | Exportar | Config | Logs]   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  🌐 Árbol Jerárquico           │ 📋 Grid de Propuestas      │
│  ┌─────────────────────────┐   │ ┌─────────────────────────┐│
│  │ 🟢 LÁCTEOS_DIARIO       │   │ │Item │RefFabr│Qty│Estado ││
│  │  └─ 🟢 DANONE           │   │ │1234 │DAN001 │50 │OK     ││
│  │     └─ 🟢 VAL4          │   │ │5678 │DAN002 │30 │MANP3  ││
│  │     └─ 🔴 ALI1          │   │ │9012 │DAN003 │20 │MANP4  ││
│  │  └─ 🟠 NESTLÉ           │   │ │...                      ││
│  │     └─ 🔵 MAD1          │   │ │                         ││
│  │ 🔴 CONGELADOS_SEMANAL   │   │ │                         ││
│  │  └─ ...                 │   │ │                         ││
│  └─────────────────────────┘   │ └─────────────────────────┘│
│                                                             │
│ [Panel de Restricciones / Sustituciones / Movimientos]      │
└─────────────────────────────────────────────────────────────┘
```

**Código de colores del árbol:**

| Color | Estado | Significado para el planificador |
|---|---|---|
| ⬜ **Gris** | Sin procesar | No se ha calculado todavía |
| 🟢 **Verde** | OK | Sin problemas. Se puede exportar |
| 🔴 **Rojo** | MANP3 | ¡Atención! Problema de lote mínimo |
| 🟠 **Naranja** | MANP4 | ¡Atención! Problema de compra mínima |
| 🟡 **Amarillo** | MANP5 | Atención: problema de portes |
| 🔵 **Azul** | To Export | Marcado para exportar |
| 🟣 **Lila** | Exportado | Ya enviado al ERP |

El color de cada nodo refleja el **estado más grave** de sus hijos.

---

## 3. Paso 2: Importación desde SO99

### Cuándo importar

- Al inicio de cada jornada de planificación
- Cuando SO99 ha generado nuevas propuestas
- La frecuencia de importación depende de los Plangroups (diario, semanal...)

### Cómo importar

1. Pulsar el botón **"Importar"** en la toolbar
2. PEM automáticamente:
   - Determina qué Plangroups toca planificar hoy
   - Importa las propuestas de SO99 para esos Plangroups
   - Importa datos maestros actualizados (productos, stocks, proveedores)
   - Hace backup de los datos anteriores
3. Barra de progreso muestra el avance
4. Al finalizar, mensaje de confirmación con:
   - Nº de propuestas importadas
   - Nº de registros limpiados
   - Tiempo total

### Opcional: Cargar Info ERP

Botón **"Cargar Info ERP"** — Solo necesario si ha habido cambios en:
- Reglas de compensación
- Red logística
- Restricciones de proveedores
- Equivalencias entre productos

---

## 4. Paso 3: Cálculo Automático

### Qué hace PEM automáticamente

El planificador pulsa **"Calcular"** y PEM ejecuta en secuencia:

```
✅ 1. TransferOrders SO99
   └── Vincula traspasos planificados por SO99 con propuestas de compra
   └── Descuenta cantidades traspasadas

✅ 2. Sustituciones Locales
   └── Busca en el mismo almacén productos que puedan sustituir/ser equivalentes
   └── Aplica reglas de compensación
   └── Netifica (descuenta stock encontrado)

✅ 3. Sustituciones en Red
   └── Busca en TODA la red logística: mismo producto en otros almacenes
   └── Busca equivalentes en otros almacenes
   └── Genera TransferOrders para los traspasos necesarios

✅ 4. Cálculo de Mínimos
   └── Evalúa lote mínimo → marca MANP3 si no cumple
   └── Evalúa compra mínima → marca MANP4 si no cumple
   └── Evalúa portes mínimos → marca MANP5 si no cumple

✅ 5. Centralización
   └── Agrupa pedidos de proveedores ACCUM
   └── Genera TransferOrders del central a los destinos

✅ 6. Promociones
   └── Aplica descuentos y rappels
```

### Modo Multithreading

Si `multiThreading=true` (por defecto), PEM usa procesamiento paralelo:
- Varios Plangroups se calculan simultáneamente
- Dentro de cada Plangroup, varios Warehouses en paralelo
- **Más rápido** en máquinas con múltiples núcleos

---

## 5. Paso 4: Visualización de Resultados

### Navegación por el Árbol

```
🌐 Plangroup "LÁCTEOS_DIARIO" 🟢
│   → Expandir para ver proveedores
│
├── 🏭 DANONE 🟢
│   │  → Expandir para ver almacenes
│   │
│   ├── 🏬 VAL4 (Valencia) 🟢
│   │   └── Click para ver propuestas en el grid
│   │
│   ├── 🏬 ALI1 (Alicante) 🔴
│   │   └── ¡Atención! Tiene propuestas con MANP3
│   │
│   └── 🏬 CAS1 (Castellón) 🟠
│       └── ¡Atención! Tiene propuestas con MANP4
│
└── 🏭 NESTLÉ 🟢
    ├── 🏬 MAD1 (Madrid) 🔵
    └── 🏬 SEV1 (Sevilla) 🟣
```

### El Grid de Propuestas

Al hacer clic en un almacén, el grid muestra todas sus propuestas con las columnas configuradas:

| ItemID | RefFabr | Descripción | QProp | QtyFinal | Estado |
|---|---|---|---|---|---|
| 123456 | REF001 | Leche entera 1L | 100 | 100 | 🟢 OK |
| 789012 | REF002 | Leche semi 1L | 50 | 15 | 🔴 MANP3 |
| 345678 | REF003 | Leche desnatada 1L | 30 | 30 | 🟠 MANP4 |

### Columnas visibles

Las columnas mostradas dependen del tipo de vista y se configuran en `columns.conf`:
- Vista General: item, referencia, descripción, marca, familia, fechas, cantidades, costes
- Vista SO99: añade columnas de centralización y traspasos
- Vista Sustituciones: columnas de candidatos, stock, reglas
- Vista Centralización: columnas de agrupación

---

## 6. Paso 5: Revisión y Resolución de Problemas

### 6.1 Identificar Problemas

El planificador recorre el árbol buscando nodos con colores de alerta:
- 🔴 **Rojo** = MANP3 → Problemas graves de lote
- 🟠 **Naranja** = MANP4 → Problemas de compra mínima
- 🟡 **Amarillo** = MANP5 → Problemas de portes

### 6.2 Panel de Restricciones (Restriction_NecessitiesForm)

Al hacer doble clic en un nodo con problemas, se abre el panel de restricciones:

```
┌─────────────────────────────────────────────────────────────┐
│ RESTRICCIONES - ALI1 (Alicante) - DANONE                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Producto: 789012 - Leche semi 1L                           │
│  Cantidad necesaria: 15 uds                                 │
│  Lote mínimo: 50 uds                                        │
│  Estado actual: MANP3                                       │
│                                                             │
│  ¿Qué acción tomar?                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ( ) ADVANCE  - Adelantar a siguiente período         │   │
│  │ ( ) BUY       - Forzar compra de 50 uds              │   │
│  │ ( ) NO_BUY    - No comprar                           │   │
│  │ ( ) TRANSFER  - Traspasar desde otro almacén         │   │
│  │ ( ) CEN_NO_BUY- Centralizar sin comprar              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  [Aplicar]  [Cancelar]                                      │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 Estrategia de Resolución

**Para MANP3 (lote mínimo):**

| Situación | Acción recomendada |
|---|---|
| Producto crítico, rotura inaceptable | **BUY** — asumir el coste del lote completo |
| Producto no crítico, hay stock en otra ubicación | **TRANSFER** — buscar en la red logística |
| Producto de baja rotación, puede esperar | **ADVANCE** — acumular para el siguiente período |
| Producto descatalogado o sin demanda real | **NO_BUY** — no comprar |

**Para MANP4 (compra mínima):**

| Situación | Acción recomendada |
|---|---|
| El proveedor es el único, no hay alternativa | **BUY** — asumir el mínimo |
| Se puede agrupar con otros almacenes | **CEN_NO_BUY** — centralizar para llegar al mínimo |

**Para MANP5 (portes):**
| Situación | Acción recomendada |
|---|---|
| El coste de portes es asumible | **BUY** — pagar los portes |
| Se puede agrupar con otros pedidos | **CEN_NO_BUY** — compartir portes |

### 6.4 Ver Candidatos a Sustitución

Botón **"Ver Sustituciones"** → Abre `SubsProposalsForm`:

```
┌─────────────────────────────────────────────────────────────┐
│ CANDIDATOS A SUSTITUCIÓN - Propuesta 789012                 │
├─────────────────────────────────────────────────────────────┤
│ ItemOrig│LocOrig│ItemDest│Stock │Tipo│Regla│Qty a traspasar │
│ 789012  │VAL4   │789012  │60 uds│I   │R01  │15 uds          │
│ 890123  │MAD1   │789012  │40 uds│D   │R02  │0 uds           │
└─────────────────────────────────────────────────────────────┘
```

- Muestra qué productos/almacenes pueden compensar la propuesta
- Indica la regla que lo permite
- Muestra la cantidad que se va a traspasar

### 6.5 Ver Movimientos

Botón **"Ver Movimientos"** → Abre `MovimentsPorposalsForm`:
- TransferOrders generados
- Centralizaciones creadas
- Origen, destino, cantidad, fecha

---

## 7. Paso 6: Operaciones Manuales

### Editar Cantidades

En el grid de propuestas, el planificador puede editar manualmente:
- `QtyFinal`: cantidad final a comprar
- `ReleaseDate`: fecha de lanzamiento del pedido
- `DueDate`: fecha de entrega

Las celdas editadas se marcan y el estado de la propuesta se actualiza automáticamente.

### Editar Reglas de Compensación

Botón **"Reglas"** → Abre `RuleCompensaDialog`:
- Lista de todas las reglas de equivalencia
- Crear, editar, eliminar reglas
- Campos: tipo de equivalencia, almacenes, aplicación, ABC, caro/barato, viaja, stock

### Gestionar Plangroups

Menú **"Plangroups"** → Abre `PlangroupsForm`:
- Ver/editar frecuencia de planificación
- Cambiar tipo de centralización (ACCUM/NO_ACCUM)
- Cambiar nivel de centralización (REG/CEN)

---

## 8. Paso 7: Aprobación y Exportación al ERP

### 8.1 Marcar para Exportar

1. Seleccionar uno o varios Plangroups en el árbol
2. Botón **"Marcar para Exportar"**
3. Las propuestas del Plangroup pasan a estado **TO_EXPORT** (🔵 azul)
4. Los nodos cambian a color azul

### 8.2 Exportar

1. Botón **"Exportar"**
2. PEM ejecuta:
   - Limpia las tablas de salida
   - Inserta las órdenes de compra en `OUT_PURCHASE_ORDERS`
   - Inserta los traspasos en `OUT_TRANSFER_ORDERS`
   - Ejecuta el procedimiento almacenado de exportación al ERP
3. Las propuestas pasan a estado **EXPORTED** (🟣 lila)
4. Mensaje de confirmación con:
   - Nº de órdenes de compra exportadas
   - Nº de TransferOrders exportados
   - Nº de sustituciones aplicadas

### 8.3 Verificar Exportación

- Consultar el **Log** (botón "Logs") para ver el historial de exportaciones
- Verificar en el ERP que los pedidos han llegado correctamente

---

## 9. Escenarios Típicos

### Escenario 1: Día normal, sin problemas

```
1. Importar → 120 propuestas
2. Calcular → Todo verde ✅
3. Marcar para exportar
4. Exportar → 120 órdenes enviadas al ERP
⏱️ Tiempo total: ~5 minutos
```

### Escenario 2: Varios MANP3 en productos críticos

```
1. Importar → 150 propuestas
2. Calcular → 12 en MANP3 🔴 (productos top)
3. Revisar cada MANP3:
   • 8 → BUY (forzar compra, son críticos)
   • 3 → TRANSFER (hay stock en otros almacenes)
   • 1 → ADVANCE (baja rotación, puede esperar)
4. Recalcular → Todo verde menos el ADVANCE
5. Marcar para exportar (excepto ADVANCE)
6. Exportar → 149 órdenes
⏱️ Tiempo total: ~20-30 minutos
```

### Escenario 3: Problemas con proveedor que no separa pedidos (ACCUM)

```
1. Importar → 80 propuestas del proveedor DANONE
2. Calcular → Varias MANP4 🟠 (no llegan al mínimo de compra)
3. El planificador revisa:
   • DANONE es ACCUM → centralizar en regional
   • Todas las propuestas de DANONE se agrupan en REG_ESTE
   • PEM genera TransferOrders: REG_ESTE → VAL4, ALI1, CAS1
4. Cantidad total en REG_ESTE: 250 uds → ¡Supera el mínimo de 200!
5. Marcar para exportar
6. Exportar
⏱️ Tiempo total: ~15 minutos
```

### Escenario 4: Carga de nuevas reglas desde ERP

```
1. El equipo de compras ha definido nuevas reglas de compensación
2. Planificador pulsa "Cargar Info ERP"
3. Se actualizan reglas, equivalencias, red logística
4. Recalcular → Las nuevas reglas permiten más sustituciones
5. Algunos MANP3/4/5 se resuelven automáticamente
⏱️ Tiempo total: ~5-10 minutos
```

---

## 10. Atajos y Consejos

### Consejos de Productividad

| Consejo | Explicación |
|---|---|
| **Prioriza los rojos** | Los MANP3 suelen ser los más críticos. Resuélvelos primero |
| **Usa el filtro ABC** | Si hay restricción ABC configurada, los productos C pueden esperar |
| **Recalcula tras cambios** | Si editas cantidades o resuelves restricciones, recalcula para ver el impacto |
| **Verifica las reglas** | Si un candidato esperado no aparece, revisa las reglas de compensación |
| **Consulta el Log** | Si algo falla, el Log muestra exactamente qué ocurrió y dónde |
| **Centraliza cuando puedas** | Agrupar pedidos suele resolver múltiples MANP4/MANP5 de golpe |

### Atajos de Teclado

| Atajo | Acción |
|---|---|
| F5 | Refrescar/Recalcular |
| Ctrl+E | Exportar |
| Ctrl+I | Importar |
| Ctrl+L | Ver Logs |

---

## 11. Preguntas Frecuentes

### ❓ ¿Por qué una propuesta sigue en MANP3 si ya busqué sustitutos?

Posibles causas:
- No hay sustitutos/equivalentes definidos para ese producto
- Las reglas de compensación no permiten la operación (revisar reglas)
- El stock disponible no alcanza el lote mínimo de traslado
- El producto no "viaja" (restricción de travel = NO)

### ❓ ¿Qué diferencia hay entre un candidato Igual (I) y uno Equivalente (D)?

- **Igual (I)**: Es EXACTAMENTE el mismo producto, solo cambia el almacén. Ej: Leche DANONE 1L en VAL4 → Leche DANONE 1L en MAD1
- **Equivalente (D)**: Es un producto DIFERENTE pero intercambiable. Ej: Leche DANONE 1L → Leche PASCUAL 1L

### ❓ ¿Puedo exportar solo algunos Plangroups y dejar otros para más tarde?

Sí. Marca para exportar solo los Plangroups que quieras enviar al ERP. Los demás quedan pendientes.

### ❓ ¿Qué pasa si exporto dos veces el mismo Plangroup?

Las tablas OUT_* se limpian (TRUNCATE) antes de cada exportación, así que solo se exporta el estado actual. Las propuestas ya exportadas (estado EXPORTED) no se vuelven a enviar.

### ❓ ¿Puedo deshacer una exportación?

No directamente desde PEM. Contacta con el equipo de ERP para anular los pedidos generados. En PEM, las propuestas quedan en estado EXPORTED (95).

### ❓ ¿Cada cuánto debo importar desde SO99?

Depende de la frecuencia de planificación de tus Plangroups. Lo habitual es una vez al día para Plangroups diarios, y el día correspondiente para los semanales/quincenales.

### ❓ El cálculo va muy lento, ¿qué puedo hacer?

- Verifica que `multiThreading=true` en `settings.conf`
- Reduce el horizonte de visualización (`HorizontVisualizationTimeDays`)
- Comprueba que la conexión a las BBDD es rápida
- Si usas un servidor SQL remoto, la latencia de red afecta mucho

---

## Resumen Visual del Flujo

```
INICIO
  │
  ├─ [1] ABRIR PEM ────────────────────────────────────── 1 min
  │      • Conexión a SO99, PEM, ERP
  │
  ├─ [2] IMPORTAR ─────────────────────────────────────── 2-5 min
  │      • Carga propuestas del día desde SO99
  │      • (Opcional) Cargar Info ERP
  │
  ├─ [3] CALCULAR ─────────────────────────────────────── 1-5 min
  │      • TransferOrders, Sustituciones, Mínimos, Centralización
  │
  ├─ [4] REVISAR ──────────────────────────────────────── 5-30 min
  │      • Árbol de colores: identificar rojos/naranjas
  │      • Panel de restricciones: resolver MANP3/4/5
  │      • Editar cantidades si es necesario
  │
  ├─ [5] RECALCULAR (opcional) ────────────────────────── 1-3 min
  │      • Reflejar cambios manuales
  │
  ├─ [6] APROBAR ──────────────────────────────────────── 1 min
  │      • Marcar Plangroups para exportar
  │
  └─ [7] EXPORTAR ─────────────────────────────────────── 1-2 min
         • Enviar órdenes al ERP
         • Confirmar en Logs
```