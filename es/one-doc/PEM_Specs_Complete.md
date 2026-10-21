[← README](./README.md)

&nbsp;

# PEM - Purchase Editor Manager
## Documento de Especificaciones Técnicas y de Negocio

> **Versión:** 1.0  
> **Proyecto:** PurchaseEditorManager (PEM)  
> **Tecnología:** VB.NET WinForms (.NET Framework 4.8) + SQL Server  
> **Componentes UI:** DevExpress WinForms (Skin Office 2016 Colorful)

---

## Índice

1. [Visión General del Proyecto](#1-visión-general-del-proyecto)
2. [Arquitectura del Sistema](#2-arquitectura-del-sistema)
3. [Modelo de Clases – Capa BEC (Business Entity Classes)](#3-modelo-de-clases--capa-bec)
4. [Modelo de Clases – Capa DAL (Data Access Layer)](#4-modelo-de-clases--capa-dal)
5. [Modelo de Clases – Capa BCC (Business Control Classes)](#5-modelo-de-clases--capa-bcc)
6. [Modelo de Datos – Esquema SQL Server](#6-modelo-de-datos--esquema-sql-server)
7. [Flujo de Datos entre Bases de Datos](#7-flujo-de-datos-entre-bases-de-datos)
8. [Configuración del Sistema](#8-configuración-del-sistema)
9. [Lógica de Negocio – Procesamiento de Propuestas](#9-lógica-de-negocio--procesamiento-de-propuestas)
10. [Sistema de Sustitución, Equivalencias y Compensación](#10-sistema-de-sustitución-equivalencias-y-compensación)
11. [Redes Logísticas y Niveles de Almacén](#11-redes-logísticas-y-niveles-de-almacén)
12. [Centralización por Lotes y por Compra](#12-centralización-por-lotes-y-por-compra)
13. [Estatus de Necesidad (MANP3, MANP4, MANP5)](#13-estatus-de-necesidad-manp3-manp4-manp5)
14. [Reglas de Compensación (RuleEquivalences)](#14-reglas-de-compensación-ruleequivalences)
15. [Proceso de Importación desde SO99](#15-proceso-de-importación-desde-so99)
16. [Proceso de Exportación hacia ERP](#16-proceso-de-exportación-hacia-erp)
17. [User Journey – Flujo del Planificador](#17-user-journey--flujo-del-planificador)
18. [Sistema de Logging](#18-sistema-de-logging)
19. [Parámetros de Configuración (ParamGen)](#19-parámetros-de-configuración-paramgen)
20. [Estructura de UI y Formularios](#20-estructura-de-ui-y-formularios)

---

## 1. Visión General del Proyecto

### ¿Qué es PEM?

**Purchase Editor Manager (PEM)** es una aplicación de escritorio Windows (WinForms) que actúa como **intermediario inteligente entre el sistema de planificación SO99+ y el ERP del cliente**. Su función principal es:

1. **Importar propuestas de compra** desde SO99+ (sistema de planificación de demanda)
2. **Calcular y optimizar** dichas propuestas aplicando reglas de negocio:
   - Netificación de cantidades (ajustar Qty según stock real)
   - Sustitución de productos (items a extinguir → items equivalentes)
   - Compensación entre almacenes (traspasos de stock sobrante)
   - Ajuste a mínimos de entrega (lote mínimo, compra mínima, portes mínimos)
   - Centralización de pedidos (por lote y por compra)
3. **Presentar al planificador** una vista jerárquica (Plangroup → Supplier → Warehouse → Propuesta) con indicadores visuales de estado
4. **Permitir la revisión manual** de problemas (MANP3, MANP4, MANP5) y su resolución
5. **Exportar las órdenes de compra y traspaso** al ERP (base de datos intermedia INTER_SO99)

### Bases de Datos implicadas

| Base de Datos | Rol | Ejemplo |
|---|---|---|
| **SO99+** | Origen de datos de planificación (propuestas, masterfile, stocks) | `20244_DS_EmmaSleep` |
| **PEM** | Base de datos propia de trabajo (importación, cálculo, resultados) | `BK_PEM_Silvestre_863_19259` |
| **ERP (INTER)** | Base intermedia para exportar resultados al ERP | `800_DS_Silvestre_INTER_SO99_11052018` |

### Flujo de Alto Nivel

```
SO99+ → [Import] → PEM (cálculos y revisión) → [Export] → INTER_SO99 → ERP
```

---

## 2. Arquitectura del Sistema

### Arquitectura en Capas (3-Tier lógico en misma máquina)

```
┌─────────────────────────────────────────────────────┐
│                    UI Layer                         │
│  MainFormNEW, ProposalsForm, Dialogs, SummaryNodes  │
│  (DevExpress WinForms + ColorStatus)                │
├─────────────────────────────────────────────────────┤
│                  BCC Layer                          │
│  Business Control Classes (PurchaseEditorBCC,       │
│  ProposalsBCC, ItemBCC, SupplierBCC, etc.)          │
│  → Orquestan la lógica de negocio                   │
├─────────────────────────────────────────────────────┤
│                  DAL Layer                          │
│  Data Access Layer (SQL Server, SqlClient)          │
│  → Acceso directo a las 3 BBDD                      │
├─────────────────────────────────────────────────────┤
│                  BEC Layer                          │
│  Business Entity Classes (POCOs/DTOs)               │
│  → Transportan datos entre capas                    │
└─────────────────────────────────────────────────────┘
```

### Dependencias entre capas

- **UI → BCC**: Los formularios crean y usan `PurchaseEditorBCC` como fachada principal
- **BCC → DAL**: Las clases BCC instancian DAL para leer/escribir en BD
- **BCC → BEC**: Usan entidades BEC como contenedores de datos
- **DAL → BEC**: Construyen entidades BEC a partir de resultados SQL
- **Global (AppConf)**: Contexto compartido con conexiones, logging, idioma, parámetros

### Entry Point

[PurchaseEditorMain.vb](/PurchaseEditor/PurchaseEditorMain.vb) → `Main()`:
1. Crea `AppConf` (contexto global)
2. Carga `connection.conf`
3. Prueba conexión SQL; si falla, muestra `configSQLConnexionForm`
4. Carga `settings.conf`, `language_XX.conf`, `columns.conf`, `ParamGen`
5. Inicia la UI según argumentos de línea de comandos:
   - Sin args → `MainFormNEW`
   - `\Logs` → `LogsForm`
   - `\Console` → `ConsoleForm` (modo servidor/consola)

---

## 3. Modelo de Clases – Capa BEC

Las **Business Entity Classes** son objetos planos que representan los datos del dominio. Muchos heredan de `BaseEntity` que proporciona tracking de modificaciones y estado de edición.

### 3.1 Entidades Principales

#### `ItemMasterFileEntity` — Ficha Maestra de Producto
| Campo | Descripción |
|---|---|
| `ItemID` | Código único del producto |
| `RefFabr` | Referencia del fabricante |
| `Descr` | Descripción del producto |
| `Mark` | Marca (FREEATTR1) |
| `Section` | Sección (FREEATTR2) |
| `Line` | Línea (FREEATTR3) |
| `Family` | Familia (MACROFAM) |
| `SubFamily` | SubFamilia (PRODFAM) |
| `FamilyMark` | Familia + Marca combinado |
| `Plangroup` | Grupo de planificación asignado |
| `StatusFabr` | Estado fabricante: ACTIVO, HASTA FIN, Unknown |
| `Supplier` | Proveedor principal (FREEATTR5) |
| `ScopeDistribution` | Ámbito de distribución: _ALL, _ALL_R, _REG (FREEATTR6) |
| `RedLog` | Red logística asignada (FREEATTR7) |
| `ABC_General` | Clasificación ABC general (FREEATTR8) |
| `Segmentation` | Lista de segmentaciones del producto |
| `Unit`, `UnitVol`, `UnitCube`, `UnitWeight`, `PalletQty` | Datos logísticos |

#### `ItemLocationEntity` — Producto en un Almacén
| Campo | Descripción |
|---|---|
| `ItemID` + `Location` | Clave compuesta (SKUL) |
| `LevelWhouse` | Nivel en la red logística (N0, N1, N2, N3, NoLevel) |
| `Whouse` | Almacén |
| `Supplier` | Proveedor |
| `stockTotal` | Stock virtual total (VIRTSTK) |
| `stockTotal_Ini` | Stock virtual inicial (antes de operaciones) |
| `AveDemQty` | Demanda media desestacionalizada |
| `coverage` | Días de cobertura mínima deseada |
| `OverStkQty_Projected` | Cantidad disponible para intercambiar (proyectada) |
| `OverStkQty_AvedemQty` | Cantidad disponible para intercambiar (avg demand) |
| `UnitCost` | Coste unitario |
| `UnitCost_WithOutRapel` | Coste real sin rappel |
| `minResLot` | Lote mínimo de reserva/traslado |
| `incresLot` | Incremento de lote |
| `qtyMinOverStok` | Stock mínimo congelado en el almacén |
| `ABC_SKU` | Clasificación ABC del SKU |
| `expensive` | ¿Es caro? (S/N/ALL) |
| `travel` | ¿Puede viajar? (S/N/NONE) |
| `leadTime` | Lead time del producto en el almacén |
| `isProjSTK_OverMaxOutflow` | ¿Stock proyectado > MaxOutflow? |

#### `ProposalEntity` — Propuesta de Compra
| Campo | Descripción |
|---|---|
| `getKey` | Clave: Plangroup_Supplier_Location_Item_IdxFinal |
| `SKUL` | Clave: Plangroup_Supplier_Location_Item |
| `ItemID`, `WhouseID`, `LocationID`, `SupplierID`, `PlangroupID` | Identificadores |
| `IndProp` | Índice de propuesta SO99 |
| `DueDate`, `ReleaseDate` | Fechas de entrega y pedido |
| `StockActual` | Stock actual en el almacén |
| `QPropSO99` | Cantidad propuesta original de SO99 |
| `QProp`, `QPropCost` | Cantidad propuesta y su coste |
| `QtySust`, `QtySustCost` | Cantidad sustituida y su coste |
| `QtyAdv` | Cantidad adelantada |
| `QtyCent` | Cantidad centralizada |
| `Qty_MinItem` | Cantidad mínima por ítem (lote) |
| `Qty_MinCompr` | Cantidad mínima de compra |
| `Qty_MinPort` | Cantidad mínima de portes |
| `QtyFinal`, `QtyFinalCost` | Cantidad final neta |
| `UnitCost` | Coste unitario aplicado |
| `IdxProp` | Índice de propuesta |
| `IdxMinDel` | Índice de mínima entrega |
| `IdxFinal` | Índice final |
| `tratar` | Status de necesidad (enum `Necessity_Status`) |
| `CandidateListAux` | Lista de candidatos a sustitución para esta propuesta |
| `SubstituteList` | Lista final de sustituciones aplicadas |
| `SRC_Whouse`, `SRC_LocationID` | Origen para TransferOrders SO99 |
| `ProposalLink`, `TransferLink` | Vinculación entre propuestas y traspasos |

#### `SubstituteProposal` — Candidato a Sustitución
| Campo | Descripción |
|---|---|
| `ItemID_Ori`, `Location_Orig` | Producto y ubicación de origen (el que cede stock) |
| `ItemID_Dest`, `Location_Dest` | Producto y ubicación de destino (el que recibe) |
| `RefFabr`, `Descr`, `Mark` | Datos del producto candidato |
| `TotalStockInitial`, `TotalStock` | Stock total inicial y actual |
| `Coverage` | Días de cobertura |
| `OverStockInitial`, `OverStock` | Sobre-stock inicial y actual |
| `EquivalentStock` | Stock equivalente disponible para el traspaso |
| `EquivalentStockInitial` | Stock equivalente inicial |
| `EquivalentStockUsed` | Stock equivalente ya usado |
| `Level` | Nivel de almacén (N0=mismo, N1=misma región, N2=otra región, N3=resto) |
| `TypeEquivalence` | Tipo: S=Sustituto, I=Igual, D=Diferente (Equivalente) |
| `minResLot` | Lote mínimo de traslado |
| `qtyMinOverStok` | Stock mínimo congelado |
| `ruleID` | ID de la regla de compensación que validó este candidato |
| `Qty` | Cantidad a traspasar |
| `isOverStock` | Si se está usando sobre-stock (vs. total stock) |

#### `NecessityEntity` — Necesidad de Compra
Representa una necesidad neta calculada que debe ser cubierta.

| Campo | Descripción |
|---|---|
| `W0` | Ubicación de la propuesta original |
| `W1` | Ubicación intermedia (regional) |
| `W2` | Ubicación final de entrega |
| `Item` | Código del producto |
| `Supplier` | Proveedor |
| `P0` | Cantidad propuesta original (SO99) |
| `N0` | Necesidad neta (QtyFinal de la propuesta) |
| `NM` | Necesidad neta modificada (tras ajustes manuales) |
| `NM_MOV` | Necesidad neta modificada por ítem-ubicación |
| `NM_COMP` | Necesidad neta modificada por compra supplier-ubicación |
| `Tratar` | Estado de la necesidad |

#### Entidades de Soporte
- `TransferOrderEntity` — Orden de traspaso entre almacenes
- `CentralizationEntity` — Centralización de pedidos
- `RedLogEntity` — Nodo de red logística (Location → Regional → Central)
- `SKU_Equiv` — Representación de SKU para reglas de equivalencia
- `SupplierEntity`, `SuppItem_MovEntity`, `SuppLocation_CompraEntity`, `SuppLocation_PortesEntity`, `SuppLocation_PromoEntity` — Datos de proveedores

### 3.2 Colecciones (Dictionary-based)
- `ProposalPlangroupList` — `Dictionary(Of PlangroupID, ProposalPlangroupEntity)`
- `ProposalPlangroupEntity` → `supplierList` → `whouseList` → `idxProposalList(Idx)`
- `ItemMasterFileList` — `Dictionary(Of ItemID, ItemMasterFileEntity)`
- `ItemLocationList` — `Dictionary(Of "ItemID_Location", ItemLocationEntity)`
- `SubstitutesList` — `Dictionary(Of "ItemID@Location", List(Of SubstituteEntity))`
- `SubstituteProposal_List` — `Dictionary(Of Key, SubstituteProposal)`
- `EquivalentList` — `Dictionary(Of ItemID, List(Of EquivalentItemID))`

---

## 4. Modelo de Clases – Capa DAL

Los Data Access Layer classes encapsulan todo el acceso a SQL Server usando `SqlClient` y `SQLServerHelper`.

| Clase DAL | Responsabilidad | Conexiones |
|---|---|---|
| `ItemDAL` | Lectura de Items, ItemLocations, Sustitutos, Equivalentes, Stock Implantación, MasterFile | SO99, PEM |
| `ProposalsDAL` | Lectura/escritura de propuestas (IN_PROPOSALS, TransferOrders) | SO99, PEM |
| `SupplierDAL` | Lectura de datos de proveedores: movimientos, compras mínimas, portes, promociones | PEM |
| `PlangroupsDAL` | Lectura/escritura de Plangroups y su frecuencia de planificación | SO99, PEM |
| `RedLogisticaDAL` | Lectura de la red logística (IN_RED_LOG) | PEM |
| `ImportDAL` | Proceso de importación: backup, limpieza e inserción de propuestas y TransferOrders | SO99, PEM |
| `ExportDAL` | Exportación a ERP: PurchaseOrders, TransferOrders (PEM + SO99), Sustituciones, limpieza de tablas OUT | PEM, ERP |
| `MovimentsDAL` | Lectura de movimientos (TransferOrders, Centralizaciones) | PEM |
| `ConfigDAL` | Lectura de configuración: ParamGen, tablas de mapeo ERP | PEM |
| `LogDAL` | Escritura de logs en BD (LOGS) | PEM |
| `IOFiles` | Lectura de archivos de configuración (.conf) del sistema de archivos | N/A |
| `RuleEquivalencesDAL` | Lectura de reglas de equivalencia/compensación | PEM |
| `SKU_EquivDAL` | Lectura de equivalencias entre SKUs | PEM |

---

## 5. Modelo de Clases – Capa BCC

### 5.1 Orquestador Principal: `PurchaseEditorBCC`

Fachada principal usada por la UI. Coordina todos los subsistemas:

```
PurchaseEditorBCC
├── ProposalsBCC      → Cálculos, sustituciones, mínimos, exportación
├── SupplierBCC       → Promociones, costes
└── RedLogisticaBCC   → Red logística
```

### 5.2 Motor de Cálculo: `ProposalsBCC`

Es la clase más grande y compleja del sistema (~1161 líneas). Responsabilidades:

- Carga inicial de propuestas y TransferOrders
- Cálculo de TransferOrders SO99 (linking propuestas → traspasos)
- Netificación de propuestas con sustitutos/equivalentes
- Búsqueda de candidatos a sustitución (Local + Network)
- Validación de candidatos contra reglas de compensación
- Cálculo de mínimos de entrega (MinLot, MinCompra, MinPortes)
- Generación de TransferOrders y Centralizaciones
- Marcado para exportación
- Exportación de PurchaseOrders y TransferOrders

**Métodos clave:**
| Método | Descripción |
|---|---|
| `loadInit()` | Carga todos los DALs, BCCs dependientes, TransferOrders y propuestas |
| `calcTrasnferOrdersSO99()` | Vincula TransferOrders SO99 con propuestas de compra |
| `calcSustAll_Local()` | Calcula sustituciones dentro del mismo almacén |
| `calcSustAll_Network()` | Calcula sustituciones a través de la red logística |
| `CalcMinDel()` | Calcula mínimos de entrega (lote, compra, portes) |
| `markToExport_*()` | Marca propuestas para exportar |
| `saveExportProposals()` | Inserta en OUT_PURCHASE_ORDERS |
| `saveExportTransferOrders_PEM()` | Inserta en OUT_TRANSFER_ORDERS |

### 5.3 Soporte de Productos: `ItemBCC`

- Carga y gestiona `ItemMasterFileList`, `ItemLocationList`, `SubstitutesList`, `EquivalentList`
- Consulta stock de implantación (restricciones `IN_ITEM_STOCK_IMPL`)
- Filtrado de ItemLocations por stock > 0 y exclusiones
- Verificación de lotes mínimos y Qty mínima de traslado

### 5.4 Soporte de Proveedores: `SupplierBCC`

- Gestiona `SuppItem_MovList` (mínimos por ítem-proveedor)
- Gestiona `SuppLocation_ComprasList` (mínimos de compra por proveedor-ubicación)
- Gestiona `SuppLocation_PortesList` (mínimos de portes)
- Gestiona `SuppLocation_PromoList` (promociones por proveedor-ubicación)
- Búsqueda jerárquica: primero Supplier_Location, luego Supplier_ALL

### 5.5 Red Logística: `RedLogisticaBCC`

- Carga `RedLogMap`: red jerárquica Location → Regional → Central
- Determina niveles entre almacenes (N0, N1, N2, N3)
- Consulta almacenes regionales y centrales
- Auto-registro de almacenes no definidos (con warning)

### 5.6 Movimientos: `MovimentsBCC`

- Gestiona `TransferOrdersList` (órdenes de traspaso generadas)
- Gestiona `CentralizationList` (centralizaciones generadas)
- Permite añadir/quitar TransferOrders y Centralizaciones

### 5.7 Plangroups: `PlangroupsBCC`

- Carga la lista de Plangroups desde SO99
- Consulta frecuencia de planificación
- Determina si un Plangroup centraliza a REG (Regional → Central) o CEN (directo a Central)

### 5.8 Reglas de Equivalencia: `RuleEquivalencesBCC`

- Carga y gestiona las reglas de compensación
- `validateRule()`: evalúa cada candidato contra todas las reglas aplicables
- Selecciona la regla más restrictiva que cumple
- Verifica: WhouseOri, WhouseDest, Aplicación, ABC_gen, ABC_SKU, Caro, Viaja, TotalStock/OverStock

### 5.9 Importación: `ImportBCC`

- Orquesta el proceso de importación desde SO99:
  1. Backup de tablas actuales
  2. Limpieza de datos obsoletos
  3. Importación de Masterdata
  4. Importación de tablas de soporte
  5. Importación de propuestas del día
  6. Importación de TransferOrders

### 5.10 Logging: `LogBCC`

- Sistema de logging con cola en memoria y volcado a BD
- Niveles: Info, Warning, Error, User, UserDetail, Debug
- Soporte para logging específico de propuestas

---

## 6. Modelo de Datos – Esquema SQL Server

### Tablas del Sistema (33 tablas en total)

#### Tablas de Configuración
| Tabla | Descripción |
|---|---|
| `CONFIG_Paramgen` | Parámetros de configuración runtime (idioma, flags, fechas) |
| `CONFIG_Tables_SO99` | Mapeo de nombres de tablas entre PEM e INTER_SO99 |

#### Tablas de Entrada (IN_*)
| Tabla | Origen | Descripción |
|---|---|---|
| `IN_T_PROPOSALS1_DATE` | SO99 | Propuestas de compra importadas |
| `IN_C_MASTERFILE` | SO99 | Ficha maestra de productos |
| `IN_C_ITEMLOCATION` | SO99 | Productos por ubicación/almacén |
| `IN_C_ITEMSUPPLIER` | SO99 | Relación producto-proveedor |
| `IN_C_SUPPLIERS` | SO99 | Proveedores |
| `IN_C_PLANGROUPS` | SO99 | Grupos de planificación |
| `IN_C_WAREHOUSES` | SO99 | Almacenes |
| `IN_ITEM_SUPERSSESIONS` | SO99 | Sustituciones (productos obsoletos → nuevos) |
| `IN_ITEM_EQUIVALENCES` | ERP | Equivalencias entre productos |
| `IN_ITEM_SEGMENTATION` | ERP | Segmentación de productos |
| `IN_ITEM_STOCK_IMPL` | ERP | Restricciones de stock por implantación |
| `IN_RULE_EQUIVALENCES` | ERP | Reglas de compensación/equivalencia |
| `IN_SUPP_ITEM_MOV` | ERP | Mínimos de movimiento por ítem-proveedor |
| `IN_SUPP_LOCATION_COMPRA` | ERP | Mínimos de compra por proveedor-ubicación |
| `IN_SUPP_LOCATION_PORTES` | ERP | Mínimos de portes por proveedor-ubicación |
| `IN_SUPP_LOCATION_PROMOTIONS` | ERP | Promociones por proveedor-ubicación |
| `IN_RED_LOG` | ERP | Red logística (Location → Regional → Central) |

#### Tablas de Trabajo (WRK_*)
| Tabla | Descripción |
|---|---|
| `WRK_NECESSITIES` | Necesidades calculadas (W0, W1, W2, NM, NM_MOV, NM_COMP) |
| `WRK_RESTRICTIONS` | Restricciones aplicadas a las necesidades |
| `WRK_SUBSTITUTIONS` | Sustituciones calculadas |
| `WRK_CENTRALIZATION` | Centralizaciones generadas |

#### Tablas de Salida (OUT_*)
| Tabla | Descripción |
|---|---|
| `OUT_PURCHASE_ORDERS` | Órdenes de compra listas para exportar al ERP |
| `OUT_TRANSFER_ORDERS` | Órdenes de traspaso listas para exportar al ERP |

#### Tablas de Backup (BACK_*)
| Tabla | Descripción |
|---|---|
| `BACK_IN_PROPOSALS1_DATE` | Backup de propuestas antes de limpiar |
| `BACK_LOGS` | Backup de logs antiguos |
| `BACK_OUT_PURCHASE_ORDERS` | Backup de órdenes de compra exportadas |
| `BACK_OUT_TRANSFER_ORDERS` | Backup de órdenes de traspaso exportadas |
| `BACK_WRK_CENTRALIZATION` | Backup de centralizaciones |
| `BACK_WRK_SUBSTITUTIONS` | Backup de sustituciones |

#### Tablas de Logging
| Tabla | Descripción |
|---|---|
| `LOGS` | Log principal de operaciones |
| `LOG_TYPE_ACCIONS` | Tipos de acción para el log |
| `LOG_ACTIONS_MESSAGE` | Mensajes predefinidos de log |

---

## 7. Flujo de Datos entre Bases de Datos

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  SO99+   │     │   PEM    │     │ERP/INTER │
│ (Origen) │     │ (Trabajo)│     │ (Destino)│
└────┬─────┘     └─────┬────┘     └──────┬───┘
     │                 │                 │
     │  ① IMPORT       │                 │
     │────────────────►│                 │
     │  Propuestas,    │                 │
     │  MasterFile,    │                 │
     │  ItemLocation,  │                 │
     │  SuperSessions, │                 │
     │  Suppliers,     │                 │
     │  Plangroups,    │                 │
     │  Warehouses     │                 │
     │                 │                 │
     │                 │  ② IMPORT ERP   │
     │                 │◄────────────────│
     │                 │  Equivalences,  │
     │                 │  Segmentation,  │
     │                 │  StockImpl,     │
     │                 │  Rules,         │
     │                 │  Supp_Mov/Comp/Port/Promo,│
     │                 │  RedLog         │
     │                 │                 │
     │                 │  ③ EXPORT       │
     │                 │────────────────►│
     │                 │  OUT_PURCHASE_  │
     │                 │  ORDERS,        │
     │                 │  OUT_TRANSFER_  │
     │                 │  ORDERS,        │
     │                 │  OUT_SUBSTITUTIONS│
     │                 │                 │
```

### ① Import desde SO99
- `ImportDAL.proposalsImport()`: Lee `IN_PROPOSALS1_DATE` y filtra por Plangroups activos
- `ImportDAL.import_IN_Masterdata_Table()`: Carga masterfiles, itemlocations, etc.
- `ImportDAL.import_IN_Table()`: Carga suppliers, warehouses, segmentations, superSessions
- `ImportDAL.transferOrdersImport()`: Carga TransferOrders desde SO99

### ② Import desde ERP
- Se ejecuta bajo demanda (botón "Cargar Info ERP")
- Carga equivalencias, reglas, segmentaciones, restricciones de stock, red logística
- Método: `ProposalsBCC.load_Info_ERP()`

### ③ Export a ERP
- `ExportDAL.exportProposals()`: Ejecuta procedimiento almacenado que mueve `OUT_PURCHASE_ORDERS` → INTER_SO99
- `ExportDAL.exportTransferOrders_PEM()`: Similar para `OUT_TRANSFER_ORDERS`
- `ExportDAL.exportTransferOrders_SO99()`: Exporta TransferOrders específicos de SO99
- `ExportDAL.exportSubstitutions()`: Exporta `OUT_SUBSTITUTIONS`

---

## 8. Configuración del Sistema

### Archivos de Configuración

| Archivo | Propósito |
|---|---|
| `Config/connection.conf` | Cadenas de conexión a las 3 BBDD |
| `Config/settings.conf` | Parámetros generales (idioma, multithreading, horizonte, etc.) |
| `Config/columns.conf` | Columnas visibles y orden en los grids de la UI |
| `Config/Languages/language_ES.conf` | Traducciones al español |
| `Config/Languages/language_EN.conf` | Traducciones al inglés |
| `app.config` | Configuración .NET (DevExpress, Framework, logging) |

### Formato connection.conf
```
DataBaseName_SO99: <nombre>
DatabaseServer_SO99: <servidor>
SQLUser_SO99: <usuario>       # Vacío = Windows Auth
SQLPwd_SO99: <password>

DataBaseName_PEM: <nombre>
DatabaseServer_PEM: <servidor>
SQLUser_PEM: <usuario>
SQLPwd_PEM: <password>

DataBaseName_ERP: <nombre>
DatabaseServer_ERP: <servidor>
SQLUser_ERP: <usuario>
SQLPwd_ERP: <password>
```

---

## 9. Lógica de Negocio – Procesamiento de Propuestas

### 9.1 Jerarquía de Datos

Las propuestas se organizan en una jerarquía de 4 niveles:

```
Plangroup (Grupo de Planificación)
  └── Supplier (Proveedor)
       └── Warehouse/Whouse (Almacén)
            └── Proposal (Propuesta, indexada por IdxProp)
```

Cada propuesta tiene un `IdxProp` (0=principal, 1=alternativa, 2=segunda alternativa, etc.).

### 9.2 Flujo de Cálculo Principal

```
1. Carga de datos (loadInit)
   ├── Carga TransferOrders SO99
   ├── Carga propuestas desde BD
   ├── Carga Items, ItemLocations
   ├── Carga Red Logística
   ├── Carga Reglas de Equivalencia
   ├── Carga Movimientos
   ├── Carga Plangroups
   └── Carga Suppliers

2. Cálculo de TransferOrders SO99 (calcTrasnferOrdersSO99)
   └── Vincula propuestas con TransferOrders del SO99
       └── Netifica cantidades

3. Sustituciones Locales (calcSustAll_Local)
   └── Para cada propuesta con QtySust > 0:
       ├── resetProposal()
       ├── Buscar sustitutos (tipo S) en el mismo almacén → validateCandidates
       ├── Buscar equivalentes (tipo D) en el mismo almacén → validateCandidates
       └── netificaProposals()

4. Sustituciones en Red (calcSustAll_Network)
   └── Para cada propuesta con QtySust > 0:
       ├── Buscar sustitutos (tipo S) en toda la red → validateCandidates
       ├── Buscar iguales (tipo I) en toda la red → validateCandidates
       ├── Buscar equivalentes (tipo D) en toda la red → validateCandidates
       ├── netificaProposals()
       └── Generar TransferOrders para los traspasos

5. Cálculo de Mínimos de Entrega (CalcMinDel)
   └── Evalúa restricciones de lote mínimo, compra mínima y portes mínimos

6. Cálculo de Promociones (CalcPromo)
   └── Aplica descuentos y promociones de proveedores

7. Exportación
   └── Marcar para exportar → Insertar en OUT_* → Ejecutar SP de export
```

### 9.3 Netificación de Propuestas

La netificación es el proceso de ajustar la `QtyFinal` de una propuesta descontando el stock de los candidatos sustitutos/equivalentes seleccionados.

```
Para cada candidato validado (ordenado por EquivalentStock descendente):
    Si QtyFinal de la propuesta > 0:
        Si EquivalentStock del candidato >= QtyFinal:
            candidate.EquivalentStockUsed = QtyFinal
            QtyFinal = 0
        Sino:
            candidate.EquivalentStockUsed = EquivalentStock
            QtyFinal -= EquivalentStock
```

---

## 10. Sistema de Sustitución, Equivalencias y Compensación

### 10.1 Tipos de Equivalencia

| Tipo | Código | Descripción | Ejemplo |
|---|---|---|---|
| **Sustituto (S)** | S | Producto a extinguir que es reemplazado por otro | Item viejo → Item nuevo (SuperSession) |
| **Igual (I)** | I | Mismo producto en otro almacén | Mismo SKU en almacén regional |
| **Equivalente/Diferente (D)** | D | Producto diferente pero equivalente funcionalmente | Marca A → Marca B equivalente |

### 10.2 Flujo de Búsqueda de Candidatos

```
Propuesta con QtySust > 0
│
├── Sustitución LOCAL (mismo almacén)
│   ├── 1º Buscar Sustitutos (S)
│   │   └── Buscar en SubstitutesList items que sustituyen al item de la propuesta
│   │       └── Para cada warehouse del item sustituto:
│   │           └── Si stock > 0 y cumple minResLot → Añadir candidato
│   ├── Validar candidatos contra reglas (whouseUse=1)
│   ├── Netificar propuesta con candidatos validados
│   │
│   ├── Si QtyFinal > 0: 2º Buscar Equivalentes (D)
│   │   └── Buscar en EquivalentList items equivalentes
│   │       └── Para cada warehouse del item equivalente:
│   │           └── Si stock > 0 y cumple minResLot → Añadir candidato
│   ├── Validar candidatos contra reglas (whouseUse=1)
│   └── Netificar propuesta
│
└── Sustitución en RED (todos los almacenes de la red logística)
    ├── 1º Buscar Sustitutos (S) en toda la red
    ├── Validar candidatos contra reglas (whouseUse=2)
    ├── Netificar
    ├── 2º Buscar Iguales (I) en toda la red (mismo item, otro almacén)
    ├── Validar y netificar
    ├── 3º Buscar Equivalentes (D) en toda la red
    ├── Validar y netificar
    └── Generar TransferOrders para traspasos entre almacenes
```

### 10.3 Orden de Prioridad

Los candidatos se procesan en orden de **mayor stock equivalente disponible** (`OrderByDescending(EquivalentStockInitial)`), lo que maximiza la compensación con el menor número de traspasos.

---

## 11. Redes Logísticas y Niveles de Almacén

### 11.1 Estructura de la Red Logística

La red logística se define en `IN_RED_LOG` y tiene una jerarquía de 3 niveles:

```
Central (CEN)
  ├── Regional 1 (REG)
  │   ├── Almacén 1.1
  │   ├── Almacén 1.2
  │   └── Almacén 1.3
  ├── Regional 2 (REG)
  │   ├── Almacén 2.1
  │   └── Almacén 2.2
  └── ...
```

### 11.2 Niveles entre Almacenes (ItemLocation_LevelWhouse)

| Nivel | Significado | Condición |
|---|---|---|
| **N0** | Mismo almacén | `location1 = location2` |
| **N1** | Misma región | `regional(loc1) = regional(loc2)` |
| **N2** | Distinta región (regional de destino) | ScopeDistribution = `_ALL` o (`_ALL_R` y loc2 es regional) |
| **N3** | Resto de almacenes | ScopeDistribution = `_ALL_R` y loc2 NO es regional |
| **NoLevel** | Sin relación válida | No comparten red logística |

### 11.3 ScopeDistribution (Ámbito de Distribución)

Proviene del campo `FREEATTR6` del MasterFile:
- `_ALL`: Puede usar cualquier almacén de la red
- `_ALL_R`: Puede usar regionales de otras regiones + resto
- `_REG`: Solo puede usar almacenes de su propia región

---

## 12. Centralización por Lotes y por Compra

### 12.1 Tipos de Centralización (Plangroup_TypeAccum)

| Tipo | Código | Descripción |
|---|---|---|
| **ACCUM** | ACCUM | El proveedor no separa pedidos. Obligatorio entregar en almacén superior y crear TransferOrders aguas abajo |
| **NO_ACCUM** | NO_ACCUM | El proveedor acepta pedidos separados en un mismo almacén |
| **ALL_WHOUSE** | (vacío) | El proveedor entrega en todos los almacenes. Pedidos y traspasos según propuestas originales |

### 12.2 Niveles de Centralización (Plangroup_LevelCenter)

| Nivel | Descripción |
|---|---|
| **REG** | Centraliza primero al Regional, luego del Regional al Central |
| **CEN** | Centraliza directamente al Central (sin pasar por Regional) |

### 12.3 Flujo de Centralización

```
Para cada Supplier → Whouse → Propuesta (solo Idx=0):
    Si la propuesta requiere centralización:
        1. Agrupar cantidades por Plangroup/Supplier
        2. Determinar almacén de centralización (REG o CEN)
        3. Crear CentralizationEntity
        4. Generar TransferOrders desde el central a los almacenes destino
        5. Las propuestas originales se marcan como MANP5 o MANP5_C
```

---

## 13. Estatus de Necesidad (MANP3, MANP4, MANP5)

### 13.1 Estados de Necesidad (Necessity_Status)

| Estado | Código | Color | Significado |
|---|---|---|---|
| `NO_PROCESSED` | 0 | Gris | Sin procesar |
| `SI` | 100 | - | OK (necesidad cubierta) |
| `NO` | 200 | - | No hay necesidad |
| **MANP3** | 30 | 🔴 Rojo | **Restricción de lote mínimo no cumplida** |
| **MANP3_V** | 31 | 🔴 Rojo | MANP3 validado/revisado manualmente |
| **MANP4** | 41 | 🟠 Naranja | **Restricción de compra mínima no cumplida** |
| **MANP4_V** | 42 | 🟠 Naranja | MANP4 validado/revisado manualmente |
| **MANP5** | 51 | 🟡 Amarillo | **Restricción de portes mínimos no cumplida** |
| **MANP5_V** | 52 | 🟡 Amarillo | MANP5 validado |
| **MANP5_C** | 53 | 🟡 | Centralizado |
| `WAIT` | 60 | - | En espera |
| `TRANSFER_SO99` | 70 | Verde oscuro | TransferOrder desde SO99 |
| `TRANSFER_Link` | 75 | - | TransferOrder vinculado a propuesta |
| `TRANSFER_CEN` | 76 | - | TransferOrder de centralización |
| `TRANS_Netifica` | 77 | - | TransferOrder netificado |
| `NOT_BUY` | 80 | - | No comprar |
| **TO_EXPORT** | 90 | 🔵 Azul | **Marcado para exportar** |
| **EXPORTED** | 95 | 🟣 Lila | **Ya exportado al ERP** |
| `ANULAR` | -1 | - | Anulado |

### 13.2 Significado de Negocio de MANP3, MANP4, MANP5

- **MANP3 (Min Lot)**: La cantidad necesaria no alcanza el **lote mínimo de movimiento** del producto en ese almacén. El planificador debe decidir si fuerza la compra igualmente (asumiendo el sobre-coste del lote mínimo) o si busca alternativas (sustitutos, centralización).

- **MANP4 (Min Compra)**: La cantidad necesaria no alcanza el **mínimo de compra** exigido por el proveedor para esa ubicación. El planificador debe decidir si incrementa la cantidad al mínimo del proveedor o si centraliza/agrupa pedidos.

- **MANP5 (Min Portes)**: La cantidad necesaria no alcanza el **mínimo de portes** (gastos de envío). Similar a MANP4 pero relacionado con costes de transporte.

### 13.3 Visualización en UI

El sistema `ColorStatus` asigna colores visuales a cada estado:
- Gris: Sin procesar
- Verde: OK
- Rojo: MANP3 (problema grave de lote)
- Naranja: MANP4 (problema de compra mínima)
- Amarillo: MANP5 (problema de portes)
- Azul: Listo para exportar
- Lila: Exportado

Los nodos del árbol jerárquico (Plangroup → Supplier → Warehouse) muestran el color del estado más grave de sus hijos.

---

## 14. Reglas de Compensación (RuleEquivalences)

### 14.1 Estructura de una Regla

Definidas en la tabla `IN_RULE_EQUIVALENCES`. Cada regla especifica condiciones para permitir que un SKU de origen compense a un SKU de destino.

| Campo | Descripción | Valores |
|---|---|---|
| `RuleID` | Identificador único de la regla | String |
| `whouse_use` | Ámbito: 1 = mismo almacén, 2 = otros almacenes | 1/2 |
| `typeEquiv` | Tipo de equivalencia | I, D, S |
| `whouseOri` | Almacén de origen permitido | Código o "ALL" |
| `whouseDest` | Almacén de destino permitido | Código o "ALL" |
| `aplicationType` | Tipo de aplicación | ALL, SECC, LINE, FAMI, SUBF, MARC, SEGM, FAMA, ITEM |
| `aplicationCode` | Código específico de aplicación | Código de sección, línea, familia, etc. |
| `class_ABC_gen` | Clasificación ABC general | A, B, C |
| `class_ABC_SKU` | Clasificación ABC del SKU | A, B, C |
| `expensive` | Restricción de precio | S=Solo caros, N=Solo baratos, ALL=Todos |
| `travel` | Puede viajar entre almacenes | S=Sí, N=No, NONE=Sin restricción |
| `TotalStock` | Usa todo el stock o solo sobre-stock | True=TotalStock, False=OverStock |

### 14.2 Niveles de Aplicación (Item_Aplication)

Ordenados de menor a mayor restricción:

| Tipo | Significado | Ejemplo |
|---|---|---|
| ALL | Todos los productos | Sin filtro |
| SECC (1) | Sección | Todos los productos de la sección "LÁCTEOS" |
| LINE (2) | Línea | Línea "YOGURES" |
| FAMI (3) | Familia | Familia "YOGUR NATURAL" |
| SUBF (4) | SubFamilia | Subfamilia "YOGUR NATURAL DESNATADO" |
| MARC (5) | Marca | Marca "DANONE" |
| SEGM (6) | Segmentación | Segmento "BIO" |
| FAMA (7) | Familia + Marca | Familia "YOGUR" + Marca "DANONE" |
| ITEM (8) | Producto específico | ItemID "123456" |

### 14.3 Algoritmo de Validación de Reglas

```
validateRule(proposal, whouseUse, candidate, SKU_Ori, SKU_Equi, itemLocation):
    1. Filtrar reglas por whouse_use y typeEquiv
    2. Para cada regla candidata, verificar:
       a. whouseOri = SKU_Equi.location O "ALL"
       b. whouseDest = SKU_Ori.location O "ALL"
       c. Aplicación coincide (según tipo: sección, línea, familia, etc.)
       d. ABC_gen coincide
       e. ABC_SKU coincide
       f. Caro/Barato coincide
    3. De las que cumplen, seleccionar la MÁS RESTRICTIVA (mayor getLevelRestriction)
    4. Con la regla más restrictiva:
       a. Si mismo almacén (whouseUse=1) → OK (no importa Viaja)
       b. Si otro almacén (whouseUse=2) → Validar Viaja
       c. Si TotalStock → OK
       d. Si OverStock → Verificar OverStkQty_AvedemQty > 0 y Stock > MaxOutflow en leadTime
    5. Asignar RuleID y tipo de stock (Total/Over) al candidato
```

### 14.4 Puntuación de Restricción (getLevelRestriction)

Cada regla suma puntos por cada filtro que aplica:
- `class_ABC_gen` informado: +1
- `class_ABC_SKU` informado: +2
- `aplicationCode` informado: +valor de `aplicationType` (1 a 8)
- `whouseDest` específico: +4
- `whouseOri` específico: +4

La regla con mayor puntuación es la más restrictiva y se prefiere.

---

## 15. Proceso de Importación desde SO99

### 15.1 Flujo de Importación

```
ImportBCC.import(planGroupList, incrementalImport, transferOrdersImport)
│
├── 1. Obtener Plangroups a importar
│   └── Filtrados por fecha de planificación (frecuencia)
│
├── 2. Backup de datos actuales
│   ├── Propuestas antiguas → BACK_IN_PROPOSALS1_DATE
│   └── Logs antiguos → BACK_LOGS
│
├── 3. Limpieza de datos obsoletos
│   ├── Propuestas procesadas (STATUS = 'P')
│   ├── TransferOrders antiguos
│   └── Logs antiguos (según numDaysToClear_Log_Info)
│
├── 4. Importar Masterdata
│   ├── IN_C_MASTERFILE ← SO99 C_MASTERFILE
│   ├── IN_C_ITEMLOCATION ← SO99 C_ITEMLOCATION
│   ├── IN_C_ITEMSUPPLIER ← SO99 C_ITEMSUPPLIER
│   ├── IN_C_SUPPLIERS ← SO99 C_SUPPLIERS
│   └── IN_C_WAREHOUSES ← SO99 C_WAREHOUSES
│
├── 5. Importar tablas de soporte
│   ├── IN_ITEM_SUPERSSESIONS ← SO99
│   ├── IN_ITEM_SEGMENTATION ← SO99 (custom fields)
│   └── IN_PLANGROUPS ← SO99
│
├── 6. Importar propuestas del día
│   └── IN_T_PROPOSALS1_DATE ← SO99 PROPOSALS1_DATE
│       └── Filtro: Plangroups activos, fecha actual
│
└── 7. Importar TransferOrders (si aplica)
    └── IN_TRANSFER_ORDERS ← SO99
```

---

## 16. Proceso de Exportación hacia ERP

### 16.1 Flujo de Exportación

```
Usuario → Marcar para Exportar → Save → Export
│
├── 1. Marcar propuestas como TO_EXPORT
│   └── Recorrer jerarquía: Plangroup → Supplier → Whouse → Propuesta(Idx=0)
│
├── 2. Truncate tablas OUT_*
│   ├── TRUNCATE OUT_PURCHASE_ORDERS
│   ├── TRUNCATE OUT_TRANSFER_ORDERS
│   └── TRUNCATE OUT_SUBSTITUTIONS
│
├── 3. Bulk Insert en OUT_PURCHASE_ORDERS
│   └── Para cada Plangroup con toExport=True:
│       └── Propuestas con tratar=TO_EXPORT y QtyFinal > 0
│
├── 4. Bulk Insert en OUT_TRANSFER_ORDERS
│   └── TransferOrders generados (PEM y SO99)
│
├── 5. Ejecutar SP de exportación
│   ├── Export_Procedure (mueve OUT_* → INTER_SO99)
│   └── Las propuestas pasan a estado EXPORTED (95)
│
└── 6. Guardar estado en BD (saveProposalList)
```

### 16.2 Formato de Exportación

**OUT_PURCHASE_ORDERS:**
| Campo | Descripción |
|---|---|
| ITEM | Código de producto |
| DESCRIPTION | Descripción |
| WHOUSE | Almacén de entrega |
| SUPPLIER | Proveedor |
| IDX | Índice de propuesta |
| YEAR_PROP, MONTH_PROP, DAY_PROP | Fecha de propuesta |
| YEAR_DELIVERY, MONTH_DELIVERY, DAY_DELIVERY | Fecha de entrega |
| QTY | Cantidad |
| UNIT | Unidad |
| PROPOSAL_ID | ID de propuesta |
| WHOUSE_FINAL | Almacén final (para centralizaciones) |
| CRDATE | Fecha de creación del registro |

---

## 17. User Journey – Flujo del Planificador

### 17.1 Diagrama de Flujo

```
┌─────────────────────────────────────────────────────────────┐
│  1. INICIO: El planificador abre PEM                        │
│     • La app conecta a SO99, PEM e INTER_SO99               │
│     • Carga configuración, idioma, columnas                 │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  2. IMPORTACIÓN: El planificador ejecuta la importación     │
│     • Se importan propuestas de SO99 para los Plangroups    │
│       cuya frecuencia de planificación corresponde al día   │
│     • Se importa masterdata, stocks, sustituciones          │
│     • Opcional: Carga Info ERP (reglas, red logística)      │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  3. PEM CALCULA AUTOMÁTICAMENTE:                            │
│     a) TransferOrders SO99 → linking + netificación         │
│     b) Sustituciones Locales → busca sustitutos en mismo    │
│        almacén y netifica                                   │
│     c) Sustituciones en Red → busca iguales/equivalentes    │
│        en toda la red logística y netifica                  │
│     d) Cálculo de Mínimos → evalúa MANP3, MANP4, MANP5      │
│     e) Centralización → agrupa pedidos por lote/compra      │
│     f) Promociones → aplica descuentos                      │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  4. VISUALIZACIÓN: El planificador ve el árbol jerárquico   │
│     🌐 Plangroup                                            │
│       🏭 Supplier                                           │
│         🏬 Warehouse                                        │
│           📋 Propuestas (grid con columnas configuradas)    │
│                                                             │
│     Colores por estado:                                     │
│     ⬜ Gris = Sin procesar                                  │
│     🟢 Verde = OK                                           │
│     🔴 Rojo = MANP3 (problema de lote mínimo)               │
│     🟠 Naranja = MANP4 (problema de compra mínima)          │
│     🟡 Amarillo = MANP5 (problema de portes)                │
│     🔵 Azul = Listo para exportar                           │
│     🟣 Lila = Exportado                                     │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  5. REVISIÓN DE PROBLEMAS (MANP3, MANP4, MANP5):            │
│     El planificador revisa los nodos con problemas:         │
│                                                             │
│     Para cada necesidad en MANP3/MANP4/MANP5:               │
│     • Abre el formulario de restricciones                   │
│       (Restriction_NecessitiesForm)                         │
│     • Ve: Producto, Cantidad, Mínimo requerido,             │
│       Opciones de acción                                    │
│                                                             │
│     Acciones disponibles:                                   │
│     • ADVANCE (0): Adelantar cantidad                       │
│     • BUY (1): Forzar compra (ignorar mínimo)               │
│     • NO_BUY (2): No comprar                                │
│     • TRANSFER (3): Traspasar desde otro almacén            │
│     • CEN_NO_BUY (4): Centralizar y no comprar              │
│                                                             │
│     El planificador también puede:                          │
│     • Editar cantidades manualmente en el grid              │
│     • Ver candidatos a sustitución (SubsProposalsForm)      │
│     • Ver movimientos/traspasos (MovimentsPorposalsForm)    │
│     • Editar reglas de compensación (RuleCompensaDialog)    │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  6. RECALCULAR (opcional):                                  │
│     Tras cambios manuales, el planificador puede:           │
│     • Re-ejecutar cálculos de sustituciones                 │
│     • Re-ejecutar cálculo de mínimos                        │
│     • Los estados se actualizan automáticamente             │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  7. APROBACIÓN Y EXPORTACIÓN:                               │
│     • El planificador selecciona Plangroups a exportar      │
│     • Marca como "To Export" → estado TO_EXPORT (90)        │
│     • Ejecuta Exportar:                                     │
│       - Se insertan en OUT_PURCHASE_ORDERS                  │
│       - Se insertan en OUT_TRANSFER_ORDERS                  │
│       - Se ejecuta SP que mueve datos a INTER_SO99          │
│     • Las propuestas pasan a estado EXPORTED (95)           │
│     • Los datos llegan al ERP para generar pedidos reales   │
└─────────────────────────────────────────────────────────────┘
```

### 17.2 Workflow del Planificador (Resumen)

1. **Abrir PEM** → Conexión a BBDD
2. **Importar** → Carga propuestas del día desde SO99
3. **Calcular** → PEM ejecuta todos los cálculos automáticos
4. **Visualizar** → Árbol con indicadores de color
5. **Revisar MANP3/MANP4/MANP5** → Resolver problemas manualmente
6. **Recalcular** si es necesario
7. **Aprobar** → Marcar para exportar
8. **Exportar** → Datos enviados al ERP

---

## 18. Sistema de Logging

### 18.1 Arquitectura de Logging

```
LogBCC (Business Control)
├── Modo Texto (inicial, antes de conexión DB)
│   └── Archivos en /Log/
└── Modo Base de Datos (tras conexión)
    ├── Cola en memoria (ConcurrentQueue)
    ├── Timer de volcado periódico
    └── Tabla LOGS en BD PEM
```

### 18.2 Tipos de Log

| Tipo | Descripción | Cuándo se usa |
|---|---|---|
| `User` | Acciones visibles para el usuario | Operaciones normales |
| `UserDetail` | Detalle de acciones de usuario | Debug de operaciones |
| `Debug` | Información de depuración | Solo en desarrollo |
| `Error` | Errores | Excepciones capturadas |

### 18.3 Tipos de Acción (LogTypeActionEnum)

| Acción | Descripción |
|---|---|
| `Import` | Procesos de importación |
| `Export` | Procesos de exportación |
| `Load_data` | Carga de datos |
| `Load_UI` | Carga de interfaz |
| `Load_Server` | Carga en modo servidor |
| `Compensa_Local` | Compensación local |
| `Compensa_Network` | Compensación en red |
| `Min_Lot` | Cálculo de lote mínimo |
| `Min_Compra` | Cálculo de compra mínima |
| `Min_Portes` | Cálculo de portes mínimos |
| `ValidateRule` | Validación de reglas |
| `AddCandidate` | Añadir candidato |
| `redLog` | Operaciones de red logística |
| `Trans_Order` | Órdenes de traspaso |
| `Centr` | Centralización |
| `Promo` | Promociones |

---

## 19. Parámetros de Configuración (ParamGen)

Gestionados por `ParamGenBCC` y persistidos en `CONFIG_Paramgen`.

| Parámetro | Tipo | Descripción | Valor típico |
|---|---|---|---|
| `Language` | String | Idioma de la app (ES/EN) | ES |
| `multiThread` | Boolean | Usar procesamiento paralelo (Parallel.ForEach) | true |
| `HorizontVisualizationTimeDays` | Integer | Días de horizonte de visualización (0 = todo) | 90 |
| `UnitCostFlag` | Boolean | true=PurchasePrice, false=UnitCost | false (0) |
| `numDaysToClear_Log_Info` | Integer | Días para limpiar logs de info | 30 |
| `ImportDay` | Date | Fecha de importación | Hoy |
| `skipImport` | Boolean | Saltar importación | false |
| `Log_User` | Boolean | Activar logging de usuario | true |
| `Log_UserDetail` | Boolean | Activar logging detallado de usuario | false |
| `Log_Debug` | Boolean | Activar logging de debug | false |
| `Log_num_Queue` | Integer | Tamaño de cola de log | 100 |
| `Adv_ABC_Restriction` | String | Restricción ABC para adelantos | "" |

---

## 20. Estructura de UI y Formularios

### 20.1 Formularios Principales

| Formulario | Archivo | Propósito |
|---|---|---|
| **MainFormNEW** | `UI/MainFormNEW.vb` | Pantalla principal con árbol jerárquico, grids, toolbars |
| **MainFormNEW_AUX** | `UI/MainFormNEW_AUX.vb` | Código auxiliar del MainForm (handlers, workers) |
| **ConsoleForm** | `UI/ConsoleForm.vb` | Modo consola/servidor |
| **LogsForm** | `UI/LogsForm.vb` | Visualización de logs |
| **configSQLConnexionForm** | `UI/configSQLConnexionForm.vb` | Configuración de conexiones SQL |

### 20.2 Formularios de Propuestas

| Formulario | Propósito |
|---|---|
| **ProposalsForm** | Grid detallado de propuestas |
| **SubsProposalsForm** | Grid de candidatos a sustitución |
| **MovimentsPorposalsForm** | Grid de movimientos/traspasos |

### 20.3 Diálogos

| Diálogo | Propósito |
|---|---|
| **RuleCompensaDialog** | Edición de reglas de compensación |
| **promoDialog** | Gestión de promociones |
| **statusDIalog** | Visualización/edición de estados |

### 20.4 Nodos de Resumen (SummaryNodes)

| Componente | Propósito |
|---|---|
| **Plangroup_Node_Info** | Vista resumen de un Plangroup |
| **Supplier_Node_Info** | Vista resumen de un Supplier |
| **Whouse_Node_Info** | Vista resumen de un Warehouse |
| **NodeUtils** | Utilidades para nodos |

### 20.5 Panel de Restricciones

| Formulario | Propósito |
|---|---|
| **Restriction_NecessitiesForm** | Panel donde el planificador revisa y resuelve necesidades con problemas (MANP3/4/5) |

---

## Apéndice A: Enums y Constantes Clave

### Equivalent_Type
- `S` (1) = Sustituto (SuperSession)
- `I` (2) = Igual
- `D` (3) = Distinto (Equivalente)

### Item_Aplication
- `ALL` (0), `SE` (1), `LINE` (2), `FAMI` (3), `SUBF` (4), `MARC` (5), `SEGM` (6), `FAMA` (7), `ITEM` (8)

### Plangroup_TypeAccum
- `ACCUM` (1) = Centraliza, no separa pedidos
- `NO_ACCUM` (2) = No centraliza
- `ALL_WHOUSE` (3) = Entrega en todos los almacenes

### Plangroup_LevelCenter
- `REG` (1) = Centraliza vía Regional
- `CEN` (2) = Centraliza directo a Central

### Restriction_Action
- `ADVANCE` (0) = Adelantar
- `BUY` (1) = Comprar
- `NO_BUY` (2) = No comprar
- `TRANSFER` (3) = Traspasar
- `CEN_NO_BUY` (4) = Centralizar sin comprar

---

## Apéndice B: Glosario

| Término | Significado |
|---|---|
| **SO99** | Sistema de planificación de demanda (origen de propuestas) |
| **Plangroup** | Grupo de planificación que agrupa productos con misma frecuencia de revisión |
| **Propuesta** | Sugerencia de compra generada por SO99 para un producto en un almacén |
| **Netificación** | Proceso de ajustar la cantidad de una propuesta restando stock de sustitutos |
| **Sustituto (S)** | Producto a extinguir que reemplaza al producto de la propuesta |
| **Igual (I)** | El mismo producto en otro almacén |
| **Equivalente (D)** | Producto diferente pero funcionalmente intercambiable |
| **MANP3** | Necesidad que no alcanza el lote mínimo de movimiento |
| **MANP4** | Necesidad que no alcanza la compra mínima del proveedor |
| **MANP5** | Necesidad que no alcanza el mínimo de portes |
| **Centralización** | Agrupación de pedidos en un almacén superior para cumplir mínimos |
| **Red Logística** | Jerarquía de almacenes: Central → Regional → Local |
| **TransferOrder** | Orden de traspaso de stock entre almacenes |
| **OverStock** | Stock que excede la demanda proyectada (disponible para intercambiar) |
| **ScopeDistribution** | Ámbito hasta dónde puede distribuirse un producto en la red |
| **SKUL** | Clave compuesta: SKU + Location |
