| [← README](README_es_multi-doc.md) | 📄 Technical Specs | [Business Logic](02_Business_Logic.md) | [User Journey](03_User_Journey.md) |
|:---:|:---:|:---:|:---:|

&nbsp;

# PEM - Technical Specifications
## Para Developers

> **Proyecto:** PurchaseEditorManager (PEM)  
> **Tecnología:** VB.NET WinForms (.NET Framework 4.8) + SQL Server + DevExpress  
> **Versión:** V 800.200514 Beta

---

## Índice

1. [Visión General](#1-visión-general)
2. [Arquitectura del Sistema](#2-arquitectura-del-sistema)
3. [Entry Point y Ciclo de Vida](#3-entry-point-y-ciclo-de-vida)
4. [Capa BEC – Business Entity Classes](#4-capa-bec--business-entity-classes)
5. [Capa DAL – Data Access Layer](#5-capa-dal--data-access-layer)
6. [Capa BCC – Business Control Classes](#6-capa-bcc--business-control-classes)
7. [Modelo de Datos – Esquema SQL Server](#7-modelo-de-datos--esquema-sql-server)
8. [Flujo de Datos entre Bases de Datos](#8-flujo-de-datos-entre-bases-de-datos)
9. [Configuración del Sistema](#9-configuración-del-sistema)
10. [Sistema de Logging](#10-sistema-de-logging)
11. [Parámetros de Configuración (ParamGen)](#11-parámetros-de-configuración-paramgen)
12. [Estructura de UI y Formularios](#12-estructura-de-ui-y-formularios)
13. [Multithreading y Rendimiento](#13-multithreading-y-rendimiento)
14. [Apéndice: Enums, Constantes y Utilidades](#14-apéndice-enums-constantes-y-utilidades)

---

## 1. Visión General

**Purchase Editor Manager (PEM)** es una aplicación WinForms que actúa como intermediario entre el sistema de planificación de demanda **SO99+** y el **ERP** del cliente.

### Bases de Datos

| BD | Rol | Ejemplo |
|---|---|---|
| **SO99+** | Origen: planificación, masterfile, stocks | `20244_DS_EmmaSleep` |
| **PEM** | Trabajo: importación, cálculo, resultados | `BK_PEM_Silvestre_863_19259` |
| **ERP (INTER)** | Destino: exportación al ERP | `800_DS_Silvestre_INTER_SO99_11052018` |

```
SO99+ → [Import] → PEM (cálculos y revisión) → [Export] → INTER_SO99 → ERP
```

---

## 2. Arquitectura del Sistema

### Arquitectura en 4 Capas

```
┌─────────────────────────────────────────────────────┐
│                    UI Layer                          │
│  MainFormNEW, ProposalsForm, Dialogs, SummaryNodes  │
│  (DevExpress WinForms + ColorStatus + Resources)    │
├─────────────────────────────────────────────────────┤
│                  BCC Layer                           │
│  Business Control Classes                           │
│  → Orquestan la lógica de negocio                   │
│  → No acceden directamente a BEC ni BD              │
├─────────────────────────────────────────────────────┤
│                  DAL Layer                           │
│  Data Access Layer (SqlClient + SQLServerHelper)    │
│  → Construyen entidades BEC desde resultados SQL    │
│  → Gestionan conexiones, transacciones              │
├─────────────────────────────────────────────────────┤
│                  BEC Layer                           │
│  Business Entity Classes (POCOs/DTOs)               │
│  → Transportan datos entre capas                    │
│  → BaseEntity: tracking de modificaciones           │
└─────────────────────────────────────────────────────┘
```

### Dependencias

- **UI → BCC**: `MainFormNEW` crea `PurchaseEditorBCC` como fachada
- **BCC → DAL**: Cada BCC instancia sus DALs (ej: `ProposalsBCC` → `ProposalDAL`, `ExportDAL`, `SKU_EquivDAL`)
- **BCC → BEC**: Usan entidades como contenedores (listas, diccionarios)
- **DAL → BEC**: Construyen y devuelven entidades desde consultas SQL
- **Global (AppConf)**: Contexto singleton compartido (conexiones, logging, idioma, columnas)

### Proyecto/Solución

- Solución: `PurchaseEditor.sln`
- Proyecto: `PurchaseEditor.vbproj`
- Namespace raíz: (implícito por proyecto VB)
- Sub-namespaces: `BCC`, `BEC`, `DAL`, `ColorsPEM`, `UTIL`

---

## 3. Entry Point y Ciclo de Vida

[PurchaseEditorMain.vb](/PurchaseEditor/PurchaseEditorMain.vb)

```vb
Shared Sub Main()
    Application.EnableVisualStyles()
    Dim AppConf As AppConf = New AppConf()
    
    AppConf.LogBCC.info("Start PEM", ...)
    AppConf.loadConnectionConf()
    
    If tryConnection(AppConf) Then
        AppConf.startLog_DB_mode()
        AppConf.loadFiles()
        
        Select Case Environment.GetCommandLineArgs(1)
            Case "\Logs"    → Application.Run(New LogsForm(AppConf))
            Case "\Console" → Application.Run(New ConsoleForm(AppConf))
            Case Else       → Application.Run(New MainFormNEW(AppConf))
        End Select
    End If
End Sub
```

**Flujo de inicio:**
1. `AppConf` → inicializa logging en modo texto
2. `loadConnectionConf()` → lee `connection.conf`, crea `ConnectionInfo` con 3 `ConncectionDB`
3. `tryConnection()` → prueba conexión SO99/PEM; si falla, muestra `configSQLConnexionForm`
4. `startLog_DB_mode()` → cambia logging a base de datos (cola + timer de volcado)
5. `loadFiles()` → carga `settings.conf`, `language_XX.conf`, `columns.conf`, `ParamGen`
6. Inicia el formulario correspondiente

---

## 4. Capa BEC – Business Entity Classes

### 4.1 BaseEntity

[BaseEntity.vb](/PurchaseEditor/BEC/BaseEntity.vb)

Clase base para entidades con tracking de estado:
- `IsModified` / `modifiedEntity()` — Marca la entidad como modificada
- `IsNewProposal` — Indica si es una propuesta nueva
- `IsReadFromDB` — Indica si se leyó correctamente de BD
- `UndoEntity()` / `EditEntity()` — Control de edición/deshacer

### 4.2 Entidades Principales

#### `ItemMasterFileEntity` — Ficha Maestra de Producto
```
ItemID, RefFabr, Descr, Mark, Section, Line, Family, SubFamily
FamilyMark, Plangroup, StatusFabr, Supplier, ScopeDistribution
RedLog, ABC_General, Segmentation[], Unit, UnitVol, UnitCube
UnitWeight, PalletQty
```
Origen: `C_MASTERFILE` de SO99. Los campos `FREEATTR1-8` se mapean a atributos de negocio.

#### `ItemLocationEntity` — Producto en un Almacén
```
ItemID, Location (clave compuesta SKUL)
LevelWhouse, Whouse, Supplier
stockTotal, stockTotal_Ini, AveDemQty, coverage
OverStkQty_Projected, OverStkQty_AvedemQty
UnitCost, UnitCost_WithOutRapel
minResLot, incresLot, qtyMinOverStok
ABC_SKU, expensive, travel, leadTime
isProjSTK_OverMaxOutflow
```

#### `ProposalEntity` — Propuesta de Compra
```
Identificadores: ItemID, WhouseID, LocationID, SupplierID, PlangroupID
Índices: IndProp, IdxProp, IdxMinDel, IdxFinal
Fechas: DueDate, ReleaseDate
Cantidades: StockActual, QPropSO99, QProp, QPropCost
            QtySust, QtySustCost, QtyAdv, QtyCent
            Qty_MinItem, Qty_MinCompr, Qty_MinPort
            QtyFinal, QtyFinalCost
Coste: UnitCost
Estado: tratar (Necessity_Status)
Sustituciones: CandidateListAux, SubstituteList
TransferSO99: SRC_Whouse, SRC_LocationID, ProposalLink, TransferLink
```

#### `SubstituteProposal` — Candidato a Sustitución
```
ItemID_Ori, Location_Orig → ItemID_Dest, Location_Dest
RefFabr, Descr, Mark
TotalStock, TotalStockInitial
Coverage, OverStock, OverStockInitial
EquivalentStock, EquivalentStockInitial, EquivalentStockUsed
Level (N0-N3), TypeEquivalence (S/I/D)
minResLot, qtyMinOverStok, ruleID, Qty, isOverStock
```

#### `NecessityEntity` — Necesidad de Compra
```
W0, W1, W2 (ubicaciones), Item, Supplier
P0 (propuesta original), N0 (necesidad neta)
NM, NM_MOV, NM_COMP (necesidades modificadas)
Tratar (estado)
```
Hereda de `InfoBaseNavision`.

#### Entidades de Soporte

| Clase | Colección | Clave |
|---|---|---|
| `TransferOrderEntity` | `TransferOrderList` | WhouseDest → List |
| `CentralizationEntity` | `CentralizationList` | WhouseCent → List |
| `RedLogEntity` | `RedLogMap → RedLogList` | RedLog → Location → Entity |
| `SKU_Equiv` | - | ItemID, Location, atributos |
| `SupplierEntity` | `SupplierList` | SupplierID |
| `SuppItem_MovEntity` | `SuppItem_MovList` | Supplier_Item |
| `SuppLocation_CompraEntity` | `SuppLocation_ComprasList` | Supplier_Location |
| `SuppLocation_PortesEntity` | `SuppLocation_PortesList` | Supplier_Location |
| `SuppLocation_PromoEntity` | `SuppLocation_PromoList` | Supplier_Location |
| `RuleEquivalencesBEC` | `Dictionary(Of RuleID, Rule)` | RuleID |

### 4.3 Jerarquía de Propuestas

```
ProposalPlangroupList: Dictionary(Of PlangroupID, ProposalPlangroupEntity)
  └── ProposalPlangroupEntity
       ├── PlangroupID, PlangroupDesc, toExport
       └── supplierList: Dictionary(Of SupplierID, ProposalSupplierEntity)
            └── ProposalSupplierEntity
                 ├── SupplierID, SupplierDesc
                 └── whouseList: Dictionary(Of WhouseID, ProposalWhouseEntity)
                      └── ProposalWhouseEntity
                           ├── WhouseID, WhouseDesc, LocationID
                           └── idxProposalList: Dictionary(Of Idx, ProposalList)
                                └── ProposalList: Dictionary(Of Key, ProposalEntity)
```

### 4.4 Otras Colecciones

| Colección | Clave | Contenido |
|---|---|---|
| `ItemMasterFileList` | ItemID | `ItemMasterFileEntity` |
| `ItemLocationList` | "ItemID_Location" | `ItemLocationEntity` |
| `SubstitutesList` | "ItemID@Location" | `List(Of SubstituteEntity)` |
| `EquivalentList` | ItemID | `List(Of EquivalentItemID)` |
| `SubstituteProposal_List` | "OriItem_OriLoc_DestItem_DestLoc" | `SubstituteProposal` |

---

## 5. Capa DAL – Data Access Layer

### Patrón de Acceso a Datos

Todas las clases DAL usan:
- `SqlConnection` + `SQLServerHelper` (helper propio)
- Patrón: `ConnectionOpen → ExecuteSQL/DataReader → ConnectionClose`
- Sin ORM — SQL directo con `StringBuilderDB` para construir queries

### Clases DAL

| Clase | Responsabilidad | Conexiones |
|---|---|---|
| **ItemDAL** | Items, ItemLocations, Sustitutos, Equivalentes, Stock Implantación, MasterFile | SO99, PEM |
| **ProposalsDAL** | Lectura/escritura de propuestas y TransferOrders | SO99, PEM |
| **SupplierDAL** | Movimientos (SuppItem_Mov), Compras mínimas, Portes, Promociones | PEM |
| **PlangroupsDAL** | Lectura/escritura de Plangroups, frecuencia de planificación | SO99, PEM |
| **RedLogisticaDAL** | Lectura de red logística (`IN_RED_LOG`) | PEM |
| **ImportDAL** | Backup, limpieza e inserción de propuestas y TransferOrders | SO99, PEM |
| **ExportDAL** | Truncate + BulkInsert en OUT_*, ejecución de SP de exportación | PEM, ERP |
| **MovimentsDAL** | Lectura de TransferOrders y Centralizaciones | PEM |
| **ConfigDAL** | Lectura de ParamGen y mapeo de tablas ERP | PEM |
| **LogDAL** | Escritura de logs en tabla `LOGS` | PEM |
| **IOFiles** | Lectura de archivos `.conf` del sistema de archivos | N/A (File I/O) |
| **RuleEquivalencesDAL** | Lectura de reglas de equivalencia | PEM |
| **SKU_EquivDAL** | Lectura de equivalencias entre SKUs | PEM |

### Operaciones Clave por DAL

**ImportDAL:**
```
proposalsBack()          → INSERT INTO BACK_IN_PROPOSALS1_DATE SELECT ...
proposalsClean(list, inc) → DELETE FROM IN_T_PROPOSALS1_DATE WHERE ...
proposalsImport(list)     → INSERT INTO IN_T_PROPOSALS1_DATE SELECT ... FROM SO99
transferOrdersBack()      → Backup de TransferOrders
transferOrdersImport()    → Import desde SO99
cleanOldDataDB()          → Limpieza de logs y datos obsoletos
import_IN_Masterdata_Table() → Carga C_MASTERFILE, C_ITEMLOCATION, etc.
import_IN_Table()         → Carga C_SUPPLIERS, C_WAREHOUSES, etc.
```

**ExportDAL:**
```
truncateExportProposals()              → TRUNCATE OUT_PURCHASE_ORDERS
truncateExportTransferOrders_PEM()     → TRUNCATE OUT_TRANSFER_ORDERS
truncateExportTransferOrders_SO99()    → TRUNCATE OUT_TRANSFER_ORDERS_SO99
bulkInsert_ProposalsExport_List()      → SqlBulkCopy a OUT_PURCHASE_ORDERS
bulkInsert_TransferOrder_PEM_List()    → SqlBulkCopy a OUT_TRANSFER_ORDERS
exportProposals()                      → Ejecuta SP de exportación a ERP
exportTransferOrders_PEM()             → Ejecuta SP de exportación a ERP
```

---

## 6. Capa BCC – Business Control Classes

### 6.1 Orquestador Principal: `PurchaseEditorBCC`

[PurchseEditorBCC.vb](/PurchaseEditor/BCC/PurchseEditorBCC.vb)

Fachada usada por la UI. Expone:

```
PurchaseEditorBCC(AppConf)
├── ProposalsBCC → Motor de cálculo
├── SupplierBCC  → Promociones, costes
└── RedLogisticaBCC → Red logística

Métodos públicos:
  initLoadData()          → Carga inicial (UI)
  initLoadData_Console()  → Carga inicial (Consola)
  load_Info_ERP()         → Recarga datos desde ERP
  reloadProposals(where)  → Recarga propuestas + items
  calcTransferOrdersSO99()→ Calcula TransferOrders SO99
  calcSustAll_Local()     → Sustituciones locales
  calcSustAll_Network()   → Sustituciones en red
  CalcProposalsConsole()  → Cálculo completo (consola)
  CalcMinDel()            → Cálculo de mínimos
  CalcPromo()             → Cálculo de promociones
  saveProposalList()      → Persiste propuestas
  getRuleEquivalencesBEC()→ Acceso a reglas
  markToExport_*()        → Marcado para exportación
  saveExport*()           → Inserción en OUT_*
  export*()               → Ejecución SP de export
```

### 6.2 Motor de Cálculo: `ProposalsBCC`

[ProposalBCC.vb](/PurchaseEditor/BCC/ProposalBCC.vb) (~1161 líneas)

**Dependencias internas:**
```
ProposalsBCC
├── ProposalsDAL, ExportDAL, SKU_EquivDAL
├── ItemBCC, SupplierBCC, RedLogisticaBCC
├── RuleEquivalencesBCC, MovimentsBCC, PlangroupsBCC
└── Estructuras internas:
    ├── mTransferSO99Plangroup
    ├── mProposalPlangroupList
    ├── mCandidateList_Proposals
    └── mSummary_NEC_ProcesList
```

**Métodos de carga:**
| Método | Descripción |
|---|---|
| `loadInit(connectionInfo)` | Carga DALs, BCCs, TransferOrders y propuestas. Timer de perfilado |
| `load_Info_ERP(connectionPEM)` | Recarga Plangroups, reglas, equivalencias, items, suppliers desde ERP |
| `LoadTransferOrderList(whereSQL)` | Carga TransferOrders SO99 filtrados |
| `LoadproposalList(whereSQL)` | Carga propuestas desde PEM |

**Métodos de cálculo:**
| Método | Algoritmo |
|---|---|
| `calcTrasnferOrdersSO99()` | Vincula TransferOrders SO99 → Propuestas, netifica |
| `calcSustAll_Local(parallel)` | Sustituciones en mismo almacén (S → D), netifica |
| `calcSustAll_Network(parallel)` | Sustituciones en red (S → I → D), genera TransferOrders, netifica |
| `CalcMinDel(parallel, debug)` | Evalúa mínimos de lote, compra, portes |

**Procesamiento paralelo:**
```
GoOverProposalsParallel(typeOperation, sourceEquivalent)
  └── Parallel.ForEach(mProposalPlangroupList, goOver_Plangroup)
       └── Parallel.ForEach(supplier.whouseList, goOver_Whouse)
            └── Parallel.ForEach(whouse.idxProposalList[0], calc_NetificationProposal)
```

### 6.3 ItemBCC — Gestión de Productos

[ItemBCC.vb](/PurchaseEditor/BCC/ItemBCC.vb)

- Carga y expone: `ItemMasterFileList`, `ItemLocationList`, `SubstitutesList`, `EquivalentList`
- `getItemLocation(itemID, location)` → Busca SKUL
- `getItemLocationsByItem(itemID, ...)` → Todos los almacenes de un producto, con filtros:
  - Excluir ubicación actual
  - Excluir ubicación SRC (TransferOrders)
  - Filtrar por `stockTotal > 0`
  - Filtrar por Stock de Implantación (`IN_ITEM_STOCK_IMPL`)
- `isMinLotSatisfied(itemLocation)` → `stockTotal >= minResLot`
- `isProjectedStockOverMax_v800(itemLocation)` → Proyección de stock vs MaxOutflow

### 6.4 SupplierBCC — Gestión de Proveedores

[SupplierBCc.vb](/PurchaseEditor/BCC/SupplierBCc.vb)

- 4 colecciones: `SuppItem_MovList`, `SuppLocation_ComprasList`, `SuppLocation_PortesList`, `SuppLocation_PromoList`
- Búsqueda jerárquica en Compras:
  1. Buscar `Supplier_Location`
  2. Si no existe, buscar `Supplier_ALL`
  3. Si existe, seleccionar la mejor restricción para el ítem (`getBestSuppLocation_Compras`)
- `isAnyRestrictionForThisSupplier_*()` → Consultas de existencia para early-exit

### 6.5 RedLogisticaBCC — Red Logística

[RedLogisticaBCC.vb](/PurchaseEditor/BCC/RedLogisticaBCC.vb)

- `getWhouseLevel(redLogID, loc1, loc2, scopeDistribution)` → N0/N1/N2/N3/NoLevel
- `getWhouse_REG(redLogID, whouse, itemID)` → Almacén regional
- `getWhouse_CEN(redLogID, whouse, itemID)` → Almacén central
- `isRegional(redLogID, whouse)` / `isCentral(redLogID, whouse)`
- Auto-registro: si un almacén no está definido, lo añade con warning

### 6.6 MovimentsBCC — Movimientos

[MovimentsBCC.vb](/PurchaseEditor/BCC/MovimentsBCC.vb)

- `TransferOrdersList`: `ConcurrentDictionary(Of WhouseDest, List(Of TransferOrderEntity))`
- `CentralizationList`: `Dictionary(Of WhouseCent, List(Of CentralizationEntity))`
- `addTransferOrder()` — Añade con manejo de concurrencia (`TryAdd`)
- `removeTrasnferOrder()` — Por necesidad, busca por `PlangroupID_WhouseDest_WhouseOri_ItemID`
- `addCentralization()` — Añade centralización

### 6.7 PlangroupsBCC — Grupos de Planificación

[PlangroupsBCC.vb](/PurchaseEditor/BCC/PlangroupsBCC.vb)

- `getPlanGroup_FreqPlanification_List()` → Diccionario de fechas de planificación
- `getPlangroupsTable()` / `savePlangroupsTable()` → Edición de Plangroups
- `centralizeToREG(plangroupID)` → ¿Centraliza vía Regional?

### 6.8 RuleEquivalencesBCC — Reglas de Compensación

[RuleEquivalencesBCC.vb](/PurchaseEditor/BCC/Equivalences/RuleEquivalencesBCC.vb)

- `validateRule(proposal, whouseUse, candidate, SKU_Ori, SKU_Equi, itemLocation, ItemBCC)` → Boolean
- Validación en cascada de cada campo de la regla
- Selección de la regla más restrictiva (`getLevelRestriction`)
- Verificación de TotalStock vs OverStock

### 6.9 ImportBCC — Importación

[ImportBCC.vb](/PurchaseEditor/BCC/ImportBCC.vb)

- `import(planGroupList, incrementalImport, transferOrdersImport)`
- Flujo: Backup → Clean → Masterdata → Tables → Proposals → TransferOrders
- Timer de perfilado en cada paso

### 6.10 LogBCC — Logging

[LogBCC.vb](/PurchaseEditor/BCC/LogBCC.vb)

- `info()`, `warning()`, `errorLog()`, `infoProposal()`, `errorProposal()`
- `writeLogQueue()` / `forceWriteLogQueue()` — Volcado de cola a BD
- `writeLogText()` — Escritura en archivo de texto

---

## 7. Modelo de Datos – Esquema SQL Server

### 7.1 Tablas de Configuración (2)

| Tabla | Columnas clave |
|---|---|
| `CONFIG_Paramgen` | Parámetros runtime: Language, multiThread, UnitCostFlag, ImportDay, Log_*, etc. |
| `CONFIG_Tables_SO99` | Mapeo de nombres de tablas entre PEM e INTER_SO99 para exportación |

### 7.2 Tablas de Entrada – IN_* (17)

| Tabla | Origen | PK / Clave |
|---|---|---|
| `IN_T_PROPOSALS1_DATE` | SO99 | ITEM, DESTWHS, INDPROP, IMPORT_DATE |
| `IN_C_MASTERFILE` | SO99 | ITEM |
| `IN_C_ITEMLOCATION` | SO99 | ITEM, LOCATION |
| `IN_C_ITEMSUPPLIER` | SO99 | ITEM, SUPPLIER |
| `IN_C_SUPPLIERS` | SO99 | SUPPLIER |
| `IN_C_PLANGROUPS` | SO99 | PLANGROUP |
| `IN_C_WAREHOUSES` | SO99 | WAREHOUSE |
| `IN_ITEM_SUPERSSESIONS` | SO99 | SUPERSEDED_ITEM, SUPERSEDING_ITEM |
| `IN_ITEM_EQUIVALENCES` | ERP | ITEM_ORI, ITEM_EQUI |
| `IN_ITEM_SEGMENTATION` | ERP | ITEM, SEGMENT |
| `IN_ITEM_STOCK_IMPL` | ERP | ITEM, LOCATION |
| `IN_RULE_EQUIVALENCES` | ERP | RULE_ID |
| `IN_SUPP_ITEM_MOV` | ERP | SUPPLIER, ITEM |
| `IN_SUPP_LOCATION_COMPRA` | ERP | SUPPLIER, LOCATION |
| `IN_SUPP_LOCATION_PORTES` | ERP | SUPPLIER, LOCATION |
| `IN_SUPP_LOCATION_PROMOTIONS` | ERP | SUPPLIER, LOCATION |
| `IN_RED_LOG` | ERP | RED_LOG, LOCATION |

### 7.3 Tablas de Trabajo – WRK_* (4)

| Tabla | Propósito |
|---|---|
| `WRK_NECESSITIES` | Necesidades calculadas con W0/W1/W2, NM, NM_MOV, NM_COMP |
| `WRK_RESTRICTIONS` | Restricciones aplicadas (acción: ADVANCE/BUY/NO_BUY/TRANSFER/CEN_NO_BUY) |
| `WRK_SUBSTITUTIONS` | Sustituciones calculadas entre items |
| `WRK_CENTRALIZATION` | Centralizaciones generadas |

### 7.4 Tablas de Salida – OUT_* (2)

| Tabla | Destino |
|---|---|
| `OUT_PURCHASE_ORDERS` | Órdenes de compra → ERP |
| `OUT_TRANSFER_ORDERS` | Órdenes de traspaso → ERP |

### 7.5 Tablas de Backup – BACK_* (6)

`BACK_IN_PROPOSALS1_DATE`, `BACK_LOGS`, `BACK_OUT_PURCHASE_ORDERS`, `BACK_OUT_TRANSFER_ORDERS`, `BACK_WRK_CENTRALIZATION`, `BACK_WRK_SUBSTITUTIONS`

### 7.6 Tablas de Logging – LOG_* (3)

`LOGS`, `LOG_TYPE_ACCIONS`, `LOG_ACTIONS_MESSAGE`

---

## 8. Flujo de Datos entre Bases de Datos

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  SO99+   │     │   PEM    │     │ERP/INTER │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                 │                 │
     │ ① IMPORT        │                 │
     │────────────────►│                 │
     │                 │                 │
     │                 │ ② IMPORT ERP    │
     │                 │◄────────────────│
     │                 │                 │
     │                 │ ③ EXPORT        │
     │                 │────────────────►│
```

### ① SO99 → PEM (ImportBCC / ImportDAL)

- **Masterdata**: `C_MASTERFILE`, `C_ITEMLOCATION`, `C_ITEMSUPPLIER`, `C_SUPPLIERS`, `C_WAREHOUSES`
- **Planificación**: `PROPOSALS1_DATE` (filtrado por Plangroups activos del día)
- **Sustituciones**: `SUPERSSESIONS`
- **Segmentación**: Campos custom de SO99
- **TransferOrders**: Órdenes de traspaso entre almacenes

### ② ERP → PEM (ProposalsBCC.load_Info_ERP)

Bajo demanda del usuario:
- `IN_ITEM_EQUIVALENCES`, `IN_ITEM_SEGMENTATION`, `IN_ITEM_STOCK_IMPL`
- `IN_RULE_EQUIVALENCES`, `IN_RED_LOG`
- `IN_SUPP_ITEM_MOV`, `IN_SUPP_LOCATION_COMPRA`, `IN_SUPP_LOCATION_PORTES`, `IN_SUPP_LOCATION_PROMOTIONS`

### ③ PEM → ERP (ExportDAL)

- **BulkInsert**: `OUT_PURCHASE_ORDERS`, `OUT_TRANSFER_ORDERS`, `OUT_SUBSTITUTIONS`
- **Stored Procedure**: Mueve datos de tablas OUT a tablas equivalentes en INTER_SO99
- Mapeo de nombres de tabla gestionado por `CONFIG_Tables_SO99`

---

## 9. Configuración del Sistema

### Archivos

| Archivo | Formato | Propósito |
|---|---|---|
| `Config/connection.conf` | `key:value` | 3 cadenas de conexión SQL (SO99, PEM, ERP) |
| `Config/settings.conf` | `key:value` | Parámetros generales |
| `Config/columns.conf` | `key: value1, value2, ...` | Columnas visibles por tipo de grid |
| `Config/Languages/language_ES.conf` | `"key"="value"` | Traducciones español |
| `Config/Languages/language_EN.conf` | `"key"="value"` | Traducciones inglés |
| `app.config` | XML | DevExpress skin, .NET Framework, logging |

### connection.conf

```
DataBaseName_SO99: <nombre>      DatabaseServer_SO99: <server>
SQLUser_SO99: <user>             SQLPwd_SO99: <pwd>        # Vacío = AD Auth

DataBaseName_PEM: <nombre>       DatabaseServer_PEM: <server>
SQLUser_PEM: <user>              SQLPwd_PEM: <pwd>

DataBaseName_ERP: <nombre>       DatabaseServer_ERP: <server>
SQLUser_ERP: <user>              SQLPwd_ERP: <pwd>
```

### settings.conf

```
Language: ES|EN
multiThreading: true|false
HorizontVisualizationTimeDays: 90
UnitCostFlag: 0|1              # 0=UnitCost, 1=PurchasePrice
```

### columns.conf

Define columnas visibles para cada tipo de vista:
- `colsGeneral` — Vista general de propuestas
- `colsSO99` — Vista específica SO99
- `colsSub` — Vista de sustituciones
- `colsCentra` / `colsCentraDetail` — Vistas de centralización
- `colsTransferSO99` — Vista de TransferOrders
- `colsSubstitute` — Grid de candidatos a sustitución
- `colsInterDepot` — Grid de traspasos entre almacenes

---

## 10. Sistema de Logging

### Arquitectura

```
LogBCC
├── Modo Texto (inicio)
│   └── Archivos en /Log/ con LogMgmt
└── Modo BD (tras conexión)
    ├── ConcurrentQueue(Of LogEntity) en memoria
    ├── Timer que vuelca la cola periódicamente
    ├── Tabla LOGS en BD PEM
    └── Tablas auxiliares: LOG_TYPE_ACCIONS, LOG_ACTIONS_MESSAGE
```

### Niveles de Log

| Nivel | Enum | Uso |
|---|---|---|
| Info | `LogTypeModeEnum.Info` | Operaciones normales, tiempos |
| Warning | - | Alertas no críticas |
| Error | - | Excepciones capturadas |
| User | `LogType.User` | Acciones visibles para el usuario |
| UserDetail | `LogType.UserDetail` | Detalle de operaciones |
| Debug | `LogType.Debug` | Solo en desarrollo |

### Tipos de Acción (LogTypeActionEnum)

`Import`, `Export`, `Load_data`, `Load_UI`, `Load_Server`, `Compensa_Local`, `Compensa_Network`, `Min_Lot`, `Min_Compra`, `Min_Portes`, `ValidateRule`, `AddCandidate`, `AddToList`, `redLog`, `Trans_Order`, `Centr`, `Promo`, `manualSolveRes`

### Tabla LOGS

```
LOG_ID, DATE_LOG, TYPE_MODE, ITEM_ID, WHOUSE, WHOUSE_DEST
SUPPLIER, IDX, TYPE_ACTION, ACTION_MSG, EXTRA_INFO
EXCEPTION_MSG, COMPUTER, USER_PEM, CLASS_NAME, FUNCTION_NAME
```

---

## 11. Parámetros de Configuración (ParamGen)

Gestionado por `ParamGenBCC` y `ParamGenBEC`, persistido en `CONFIG_Paramgen`.

| Parámetro | Tipo | Descripción | Default |
|---|---|---|---|
| `Language` | String | Idioma (ES/EN) | ES |
| `multiThread` | Boolean | Parallel.ForEach en cálculos | true |
| `HorizontVisualizationTimeDays` | Integer | Días visibles (0=todo) | 90 |
| `UnitCostFlag` | Boolean | true=PurchasePrice, false=UnitCost | false |
| `numDaysToClear_Log_Info` | Integer | Días para purgar logs | 30 |
| `ImportDay` | Date | Fecha de importación | Hoy |
| `skipImport` | Boolean | Omitir importación | false |
| `Log_User` | Boolean | Activar log de usuario | true |
| `Log_UserDetail` | Boolean | Activar log detallado | false |
| `Log_Debug` | Boolean | Activar log de debug | false |
| `Log_num_Queue` | Integer | Tamaño de cola de log | 100 |
| `Adv_ABC_Restriction` | String | Filtro ABC para adelantos | "" |

---

## 12. Estructura de UI y Formularios

### Formularios Principales

| Formulario | Propósito |
|---|---|
| `MainFormNEW` + `MainFormNEW_AUX` | Pantalla principal: árbol jerárquico, toolbars, background workers |
| `ConsoleForm` | Modo servidor/consola (sin UI gráfica completa) |
| `LogsForm` | Visor de logs históricos |
| `configSQLConnexionForm` | Configuración de conexiones SQL (3 pestañas) |

### Formularios de Datos

| Formulario | Datos mostrados |
|---|---|
| `ProposalsForm` | Grid de propuestas de un warehouse |
| `SubsProposalsForm` | Grid de candidatos a sustitución |
| `MovimentsPorposalsForm` | Grid de TransferOrders y Centralizaciones |

### Diálogos

| Diálogo | Propósito |
|---|---|
| `RuleCompensaDialog` | CRUD de reglas de compensación |
| `promoDialog` | Gestión de promociones |
| `statusDIalog` | Visualización/edición de estados de necesidad |

### Nodos del Árbol (SummaryNodes)

| Componente | Contenido |
|---|---|
| `Plangroup_Node_Info` | Resumen del Plangroup: color, cantidad de suppliers, propuestas |
| `Supplier_Node_Info` | Resumen del Supplier: color, whouses, estado general |
| `Whouse_Node_Info` | Resumen del Warehouse: color, número de propuestas |
| `NodeUtils` | Utilidades compartidas: cálculo de colores, agregación de estados |

### Panel de Restricciones

| Formulario | Propósito |
|---|---|
| `Restriction_NecessitiesForm` | Panel de revisión/resolución de MANP3/4/5 |

### Esquema de Colores (ColorsPEM)

| Clase | Responsabilidad |
|---|---|
| `ColorStatus` | Mapeo `Necessity_Status` → Color (borde, fondo, icono) |
| `ColorsBasic` | Colores base de la aplicación |
| `ColorsProposalForm` | Colores específicos del grid de propuestas |
| `ColorsRestrictionForm` | Colores del panel de restricciones |
| `ColorsRulesDialog` | Colores del diálogo de reglas |

---

## 13. Multithreading y Rendimiento

### Paralelismo en Cálculos

Controlado por `ParamGen.multiThread`:

```
GoOverProposalsParallel(typeOperation, sourceEquivalent)
  └── Parallel.ForEach(Plangroups)
       └── Parallel.ForEach(Whouses)      # Solo si Compensa_Local/Network
            └── Parallel.ForEach(Proposals) # Solo si Compensa_Local/Network
```

Para `Min_Lot` / `Min_Compra` / `Min_Portes`, el paralelismo es a nivel Supplier:
```
  └── Parallel.ForEach(Plangroups)
       └── calcMinimums(supplier)          # Secuencial por supplier
```

### Profiling

La clase `UTIL.timerCount` proporciona medición de tiempos:
```vb
Dim timer As New UTIL.timerCount("OperationName")
timer.startCount("Step1")
' ... work ...
timer.stepTime("Step2")
' ... work ...
timer.stopAndPrint()
```

### BulkInsert

La exportación usa `SqlBulkCopy` para inserción masiva en `OUT_PURCHASE_ORDERS` y `OUT_TRANSFER_ORDERS`.

---

## 14. Apéndice: Enums, Constantes y Utilidades

### EnumsBCC

#### Equivalent_Type
| Valor | Código | Significado |
|---|---|---|
| S | 1 | Sustituto (SuperSession: producto obsoleto → nuevo) |
| I | 2 | Igual (mismo producto, distinto almacén) |
| D | 3 | Distinto/Equivalente (producto diferente, misma función) |

#### Equivalent_Expensive
| Valor | Significado |
|---|---|
| YES (1) | Solo productos caros |
| NO (2) | Solo productos baratos |
| ALL (3) | Sin restricción de precio |

#### Equivalent_Travel
| Valor | Significado |
|---|---|
| YES (1) | Puede viajar entre almacenes |
| NO (2) | No puede viajar |
| NONE (3) | Sin restricción |

#### Item_Aplication
Ordenados por importancia (mayor = más restrictivo):
| Valor | Código | Significado |
|---|---|---|
| 0 | ALL | Todos los productos |
| 1 | SECC | Sección |
| 2 | LINE | Línea |
| 3 | FAMI | Familia |
| 4 | SUBF | SubFamilia |
| 5 | MARC | Marca |
| 6 | SEGM | Segmentación |
| 7 | FAMA | Familia + Marca |
| 8 | ITEM | Producto específico |

#### ItemLocation_LevelWhouse
| Nivel | Significado |
|---|---|
| N0 (0) | Mismo almacén |
| N1 (1) | Misma región |
| N2 (2) | Otra región (regional) |
| N3 (3) | Resto de almacenes |
| NoLevel (-1) | Sin relación |

#### Necessity_Status
| Estado | Código | Significado |
|---|---|---|
| NO_PROCESSED | 0 | Sin procesar |
| SI | 100 | Cubierta |
| NO | 200 | Sin necesidad |
| MANP3 | 30 | Problema lote mínimo |
| MANP3_V | 31 | MANP3 revisado |
| MANP4 | 41 | Problema compra mínima |
| MANP4_V | 42 | MANP4 revisado |
| MANP5 | 51 | Problema portes mínimos |
| MANP5_V | 52 | MANP5 revisado |
| MANP5_C | 53 | Centralizado |
| WAIT | 60 | En espera |
| TRANSFER_SO99 | 70 | TransferOrder SO99 |
| NOT_BUY | 80 | No comprar |
| TO_EXPORT | 90 | Marcado para exportar |
| EXPORTED | 95 | Exportado |

#### Restriction_Action
| Acción | Código | Significado |
|---|---|---|
| ADVANCE | 0 | Adelantar cantidad a un período futuro |
| BUY | 1 | Forzar compra (ignorar mínimo) |
| NO_BUY | 2 | No comprar |
| TRANSFER | 3 | Traspasar desde otro almacén |
| CEN_NO_BUY | 4 | Centralizar sin comprar |

#### Plangroup_TypeAccum
| Tipo | Significado |
|---|---|
| ACCUM (1) | Centraliza: proveedor no separa pedidos |
| NO_ACCUM (2) | No centraliza: proveedor acepta pedidos separados |
| ALL_WHOUSE (3) | Entrega en todos los almacenes |

#### Plangroup_LevelCenter
| Nivel | Significado |
|---|---|
| REG (1) | Centraliza vía Regional → Central |
| CEN (2) | Centraliza directo al Central |

### UTIL

| Clase | Métodos clave |
|---|---|
| `StringBuilderDB` | `append()`, `IN_List()`, `dateToSql()`, `strToSql()`, `numToSql()`, `boolToSql()` |
| `Utils` | `SQLStr()`, `styleGrid()`, `timerCount` (perfilado), `countRowsAffectedMessage()` |
| `timerCount` | `startCount()`, `stepTime()`, `stopCount()`, `stopAndPrint()`, `getTotalTime()` |