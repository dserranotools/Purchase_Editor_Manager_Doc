| [← README](README_en_multi-doc.md) | 📄 Technical Specs | [Business Logic](02_Business_Logic.md) | [User Journey](03_User_Journey.md) |
|:---:|:---:|:---:|:---:|

&nbsp;

# PEM - Technical Specifications
## For Developers

> **Project:** PurchaseEditorManager (PEM)  
> **Technology:** VB.NET WinForms (.NET Framework 4.8) + SQL Server + DevExpress  
> **Version:** V 800.200514 Beta

---

## Index

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Entry Point and Lifecycle](#3-entry-point-and-lifecycle)
4. [BEC Layer – Business Entity Classes](#4-bec-layer--business-entity-classes)
5. [DAL Layer – Data Access Layer](#5-dal-layer--data-access-layer)
6. [BCC Layer – Business Control Classes](#6-bcc-layer--business-control-classes)
7. [Data Model – SQL Server Schema](#7-data-model--sql-server-schema)
8. [Data Flow between Databases](#8-data-flow-between-databases)
9. [System Configuration](#9-system-configuration)
10. [Logging System](#10-logging-system)
11. [Configuration Parameters (ParamGen)](#11-configuration-parameters-paramgen)
12. [UI Structure and Forms](#12-ui-structure-and-forms)
13. [Multithreading and Performance](#13-multithreading-and-performance)
14. [Appendix: Enums, Constants and Utilities](#14-appendix-enums-constants-and-utilities)

---

## 1. Overview

**Purchase Editor Manager (PEM)** is a WinForms application that acts as an intermediary between the demand planning system **SO99+** and the client's **ERP**.

### Databases

| DB | Role | Example |
|---|---|---|
| **SO99+** | Source: planning, masterfile, stocks | `20244_DS_EmmaSleep` |
| **PEM** | Work: import, calculation, results | `BK_PEM_Silvestre_863_19259` |
| **ERP (INTER)** | Destination: export to ERP | `800_DS_Silvestre_INTER_SO99_11052018` |

```
SO99+ → [Import] → PEM (calculations and review) → [Export] → INTER_SO99 → ERP
```

---

## 2. System Architecture

### 4-Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│                    UI Layer                          │
│  MainFormNEW, ProposalsForm, Dialogs, SummaryNodes  │
│  (DevExpress WinForms + ColorStatus + Resources)    │
├─────────────────────────────────────────────────────┤
│                  BCC Layer                           │
│  Business Control Classes                           │
│  → Orchestrate business logic                       │
│  → Do not directly access BEC or DB                 │
├─────────────────────────────────────────────────────┤
│                  DAL Layer                           │
│  Data Access Layer (SqlClient + SQLServerHelper)    │
│  → Build BEC entities from SQL results              │
│  → Manage connections, transactions                 │
├─────────────────────────────────────────────────────┤
│                  BEC Layer                           │
│  Business Entity Classes (POCOs/DTOs)               │
│  → Transport data between layers                    │
│  → BaseEntity: modification tracking                │
└─────────────────────────────────────────────────────┘
```

### Dependencies

- **UI → BCC**: `MainFormNEW` creates `PurchaseEditorBCC` as a facade
- **BCC → DAL**: Each BCC instantiates its DALs (e.g.: `ProposalsBCC` → `ProposalDAL`, `ExportDAL`, `SKU_EquivDAL`)
- **BCC → BEC**: Use entities as containers (lists, dictionaries)
- **DAL → BEC**: Build and return entities from SQL queries
- **Global (AppConf)**: Shared singleton context (connections, logging, language, columns)

### Project/Solution

- Solution: `PurchaseEditor.sln`
- Project: `PurchaseEditor.vbproj`
- Root namespace: (implicit for VB project)
- Sub-namespaces: `BCC`, `BEC`, `DAL`, `ColorsPEM`, `UTIL`

---

## 3. Entry Point and Lifecycle

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

**Startup flow:**
1. `AppConf` → initializes logging in text mode
2. `loadConnectionConf()` → reads `connection.conf`, creates `ConnectionInfo` with 3 `ConncectionDB`
3. `tryConnection()` → tests SO99/PEM connection; if it fails, shows `configSQLConnexionForm`
4. `startLog_DB_mode()` → switches logging to database (queue + flush timer)
5. `loadFiles()` → loads `settings.conf`, `language_XX.conf`, `columns.conf`, `ParamGen`
6. Starts the corresponding form

---

## 4. BEC Layer – Business Entity Classes

### 4.1 BaseEntity

[BaseEntity.vb](/PurchaseEditor/BEC/BaseEntity.vb)

Base class for entities with state tracking:
- `IsModified` / `modifiedEntity()` — Marks the entity as modified
- `IsNewProposal` — Indicates if it is a new proposal
- `IsReadFromDB` — Indicates if it was correctly read from DB
- `UndoEntity()` / `EditEntity()` — Undo/edit control

### 4.2 Main Entities

#### `ItemMasterFileEntity` — Item Master File
```
ItemID, RefFabr, Descr, Mark, Section, Line, Family, SubFamily
FamilyMark, Plangroup, StatusFabr, Supplier, ScopeDistribution
RedLog, ABC_General, Segmentation[], Unit, UnitVol, UnitCube
UnitWeight, PalletQty
```
Source: `C_MASTERFILE` from SO99. The `FREEATTR1-8` fields are mapped to business attributes.

#### `ItemLocationEntity` — Item in a Warehouse
```
ItemID, Location (composite key SKUL)
LevelWhouse, Whouse, Supplier
stockTotal, stockTotal_Ini, AveDemQty, coverage
OverStkQty_Projected, OverStkQty_AvedemQty
UnitCost, UnitCost_WithOutRapel
minResLot, incresLot, qtyMinOverStok
ABC_SKU, expensive, travel, leadTime
isProjSTK_OverMaxOutflow
```

#### `ProposalEntity` — Purchase Proposal
```
Identifiers: ItemID, WhouseID, LocationID, SupplierID, PlangroupID
Indexes: IndProp, IdxProp, IdxMinDel, IdxFinal
Dates: DueDate, ReleaseDate
Quantities: StockActual, QPropSO99, QProp, QPropCost
            QtySust, QtySustCost, QtyAdv, QtyCent
            Qty_MinItem, Qty_MinCompr, Qty_MinPort
            QtyFinal, QtyFinalCost
Cost: UnitCost
Status: tratar (Necessity_Status)
Substitutions: CandidateListAux, SubstituteList
TransferSO99: SRC_Whouse, SRC_LocationID, ProposalLink, TransferLink
```

#### `SubstituteProposal` — Substitution Candidate
```
ItemID_Ori, Location_Orig → ItemID_Dest, Location_Dest
RefFabr, Descr, Mark
TotalStock, TotalStockInitial
Coverage, OverStock, OverStockInitial
EquivalentStock, EquivalentStockInitial, EquivalentStockUsed
Level (N0-N3), TypeEquivalence (S/I/D)
minResLot, qtyMinOverStok, ruleID, Qty, isOverStock
```

#### `NecessityEntity` — Purchase Necessity
```
W0, W1, W2 (locations), Item, Supplier
P0 (original proposal), N0 (net necessity)
NM, NM_MOV, NM_COMP (modified necessities)
Tratar (status)
```
Inherits from `InfoBaseNavision`.

#### Support Entities

| Class | Collection | Key |
|---|---|---|
| `TransferOrderEntity` | `TransferOrderList` | WhouseDest → List |
| `CentralizationEntity` | `CentralizationList` | WhouseCent → List |
| `RedLogEntity` | `RedLogMap → RedLogList` | RedLog → Location → Entity |
| `SKU_Equiv` | - | ItemID, Location, attributes |
| `SupplierEntity` | `SupplierList` | SupplierID |
| `SuppItem_MovEntity` | `SuppItem_MovList` | Supplier_Item |
| `SuppLocation_CompraEntity` | `SuppLocation_ComprasList` | Supplier_Location |
| `SuppLocation_PortesEntity` | `SuppLocation_PortesList` | Supplier_Location |
| `SuppLocation_PromoEntity` | `SuppLocation_PromoList` | Supplier_Location |
| `RuleEquivalencesBEC` | `Dictionary(Of RuleID, Rule)` | RuleID |

### 4.3 Proposal Hierarchy

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

### 4.4 Other Collections

| Collection | Key | Content |
|---|---|---|
| `ItemMasterFileList` | ItemID | `ItemMasterFileEntity` |
| `ItemLocationList` | "ItemID_Location" | `ItemLocationEntity` |
| `SubstitutesList` | "ItemID@Location" | `List(Of SubstituteEntity)` |
| `EquivalentList` | ItemID | `List(Of EquivalentItemID)` |
| `SubstituteProposal_List` | "OriItem_OriLoc_DestItem_DestLoc" | `SubstituteProposal` |

---

## 5. DAL Layer – Data Access Layer

### Data Access Pattern

All DAL classes use:
- `SqlConnection` + `SQLServerHelper` (custom helper)
- Pattern: `ConnectionOpen → ExecuteSQL/DataReader → ConnectionClose`
- No ORM — direct SQL with `StringBuilderDB` for building queries

### DAL Classes

| Class | Responsibility | Connections |
|---|---|---|
| **ItemDAL** | Items, ItemLocations, Substitutes, Equivalents, Implementation Stock, MasterFile | SO99, PEM |
| **ProposalsDAL** | Read/write proposals and TransferOrders | SO99, PEM |
| **SupplierDAL** | Movements (SuppItem_Mov), Minimum Purchases, Portes, Promotions | PEM |
| **PlangroupsDAL** | Read/write Plangroups, planning frequency | SO99, PEM |
| **RedLogisticaDAL** | Read logistics network (`IN_RED_LOG`) | PEM |
| **ImportDAL** | Backup, cleanup and insert of proposals and TransferOrders | SO99, PEM |
| **ExportDAL** | Truncate + BulkInsert into OUT_*, execution of export SP | PEM, ERP |
| **MovimentsDAL** | Read TransferOrders and Centralizations | PEM |
| **ConfigDAL** | Read ParamGen and ERP table mapping | PEM |
| **LogDAL** | Write logs to `LOGS` table | PEM |
| **IOFiles** | Read `.conf` files from the file system | N/A (File I/O) |
| **RuleEquivalencesDAL** | Read equivalence rules | PEM |
| **SKU_EquivDAL** | Read SKU equivalences | PEM |

### Key Operations per DAL

**ImportDAL:**
```
proposalsBack()          → INSERT INTO BACK_IN_PROPOSALS1_DATE SELECT ...
proposalsClean(list, inc) → DELETE FROM IN_T_PROPOSALS1_DATE WHERE ...
proposalsImport(list)     → INSERT INTO IN_T_PROPOSALS1_DATE SELECT ... FROM SO99
transferOrdersBack()      → TransferOrders backup
transferOrdersImport()    → Import from SO99
cleanOldDataDB()          → Clean logs and obsolete data
import_IN_Masterdata_Table() → Load C_MASTERFILE, C_ITEMLOCATION, etc.
import_IN_Table()         → Load C_SUPPLIERS, C_WAREHOUSES, etc.
```

**ExportDAL:**
```
truncateExportProposals()              → TRUNCATE OUT_PURCHASE_ORDERS
truncateExportTransferOrders_PEM()     → TRUNCATE OUT_TRANSFER_ORDERS
truncateExportTransferOrders_SO99()    → TRUNCATE OUT_TRANSFER_ORDERS_SO99
bulkInsert_ProposalsExport_List()      → SqlBulkCopy to OUT_PURCHASE_ORDERS
bulkInsert_TransferOrder_PEM_List()    → SqlBulkCopy to OUT_TRANSFER_ORDERS
exportProposals()                      → Execute export SP to ERP
exportTransferOrders_PEM()             → Execute export SP to ERP
```

---

## 6. BCC Layer – Business Control Classes

### 6.1 Main Orchestrator: `PurchaseEditorBCC`

[PurchseEditorBCC.vb](/PurchaseEditor/BCC/PurchseEditorBCC.vb)

Facade used by the UI. Exposes:

```
PurchaseEditorBCC(AppConf)
├── ProposalsBCC → Calculation engine
├── SupplierBCC  → Promotions, costs
└── RedLogisticaBCC → Logistics network

Public methods:
  initLoadData()          → Initial load (UI)
  initLoadData_Console()  → Initial load (Console)
  load_Info_ERP()         → Reload data from ERP
  reloadProposals(where)  → Reload proposals + items
  calcTransferOrdersSO99()→ Calculate TransferOrders SO99
  calcSustAll_Local()     → Local substitutions
  calcSustAll_Network()   → Network substitutions
  CalcProposalsConsole()  → Full calculation (console)
  CalcMinDel()            → Minimum calculation
  CalcPromo()             → Promotion calculation
  saveProposalList()      → Persist proposals
  getRuleEquivalencesBEC()→ Access rules
  markToExport_*()        → Mark for export
  saveExport*()           → Insert into OUT_*
  export*()               → Execute export SP
```

### 6.2 Calculation Engine: `ProposalsBCC`

[ProposalBCC.vb](/PurchaseEditor/BCC/ProposalBCC.vb) (~1161 lines)

**Internal dependencies:**
```
ProposalsBCC
├── ProposalsDAL, ExportDAL, SKU_EquivDAL
├── ItemBCC, SupplierBCC, RedLogisticaBCC
├── RuleEquivalencesBCC, MovimentsBCC, PlangroupsBCC
└── Internal structures:
    ├── mTransferSO99Plangroup
    ├── mProposalPlangroupList
    ├── mCandidateList_Proposals
    └── mSummary_NEC_ProcesList
```

**Load methods:**
| Method | Description |
|---|---|
| `loadInit(connectionInfo)` | Loads DALs, BCCs, TransferOrders and proposals. Profiling timer |
| `load_Info_ERP(connectionPEM)` | Reloads Plangroups, rules, equivalences, items, suppliers from ERP |
| `LoadTransferOrderList(whereSQL)` | Loads filtered SO99 TransferOrders |
| `LoadproposalList(whereSQL)` | Loads proposals from PEM |

**Calculation methods:**
| Method | Algorithm |
|---|---|
| `calcTrasnferOrdersSO99()` | Links SO99 TransferOrders → Proposals, netifies |
| `calcSustAll_Local(parallel)` | Same-warehouse substitutions (S → D), netifies |
| `calcSustAll_Network(parallel)` | Network substitutions (S → I → D), generates TransferOrders, netifies |
| `CalcMinDel(parallel, debug)` | Evaluates lot, purchase, and portes minimums |

**Parallel processing:**
```
GoOverProposalsParallel(typeOperation, sourceEquivalent)
  └── Parallel.ForEach(mProposalPlangroupList, goOver_Plangroup)
       └── Parallel.ForEach(supplier.whouseList, goOver_Whouse)
            └── Parallel.ForEach(whouse.idxProposalList[0], calc_NetificationProposal)
```

### 6.3 ItemBCC — Item Management

[ItemBCC.vb](/PurchaseEditor/BCC/ItemBCC.vb)

- Loads and exposes: `ItemMasterFileList`, `ItemLocationList`, `SubstitutesList`, `EquivalentList`
- `getItemLocation(itemID, location)` → Looks up SKUL
- `getItemLocationsByItem(itemID, ...)` → All warehouses for an item, with filters:
  - Exclude current location
  - Exclude SRC location (TransferOrders)
  - Filter by `stockTotal > 0`
  - Filter by Implementation Stock (`IN_ITEM_STOCK_IMPL`)
- `isMinLotSatisfied(itemLocation)` → `stockTotal >= minResLot`
- `isProjectedStockOverMax_v800(itemLocation)` → Projected stock validation

### 6.4 SupplierBCC — Supplier Management

[SupplierBCc.vb](/PurchaseEditor/BCC/SupplierBCc.vb)

- 4 collections: `SuppItem_MovList`, `SuppLocation_ComprasList`, `SuppLocation_PortesList`, `SuppLocation_PromoList`
- Hierarchical search in Purchases:
  1. Search `Supplier_Location`
  2. If not found, search `Supplier_ALL`
  3. If found, select the best restriction for the item (`getBestSuppLocation_Compras`)
- `isAnyRestrictionForThisSupplier_*()` → Existence queries for early-exit

### 6.5 RedLogisticaBCC — Logistics Network

[RedLogisticaBCC.vb](/PurchaseEditor/BCC/RedLogisticaBCC.vb)

- `getWhouseLevel(redLogID, loc1, loc2, scopeDistribution)` → N0/N1/N2/N3/NoLevel
- `getWhouse_REG(redLogID, whouse, itemID)` → Regional warehouse
- `getWhouse_CEN(redLogID, whouse, itemID)` → Central warehouse
- `isRegional(redLogID, whouse)` / `isCentral(redLogID, whouse)`
- Self-register: if a warehouse is not defined, adds it with a warning

### 6.6 MovimentsBCC — Movements

[MovimentsBCC.vb](/PurchaseEditor/BCC/MovimentsBCC.vb)

- `TransferOrdersList`: `ConcurrentDictionary(Of WhouseDest, List(Of TransferOrderEntity))`
- `CentralizationList`: `Dictionary(Of WhouseCent, List(Of CentralizationEntity))`
- `addTransferOrder()` — Adds with concurrency handling (`TryAdd`)
- `removeTrasnferOrder()` — By necessity, searches by `PlangroupID_WhouseDest_WhouseOri_ItemID`
- `addCentralization()` — Adds centralization

### 6.7 PlangroupsBCC — Planning Groups

[PlangroupsBCC.vb](/PurchaseEditor/BCC/PlangroupsBCC.vb)

- `getPlanGroup_FreqPlanification_List()` → Dictionary of planning dates
- `getPlangroupsTable()` / `savePlangroupsTable()` → Plangroups editing
- `centralizeToREG(plangroupID)` → Centralize via Regional?

### 6.8 RuleEquivalencesBCC — Compensation Rules

[RuleEquivalencesBCC.vb](/PurchaseEditor/BCC/Equivalences/RuleEquivalencesBCC.vb)

- `validateRule(proposal, whouseUse, candidate, SKU_Ori, SKU_Equi, itemLocation, ItemBCC)` → Boolean
- Cascade validation of each rule field
- Selection of the most restrictive rule (`getLevelRestriction`)
- Check TotalStock vs OverStock

### 6.9 ImportBCC — Import

[ImportBCC.vb](/PurchaseEditor/BCC/ImportBCC.vb)

- `import(planGroupList, incrementalImport, transferOrdersImport)`
- Flow: Backup → Clean → Masterdata → Tables → Proposals → TransferOrders
- Profiling timer on each step

### 6.10 LogBCC — Logging

[LogBCC.vb](/PurchaseEditor/BCC/LogBCC.vb)

- `info()`, `warning()`, `errorLog()`, `infoProposal()`, `errorProposal()`
- `writeLogQueue()` / `forceWriteLogQueue()` — Flush queue to DB
- `writeLogText()` — Write to text file

---

## 7. Data Model – SQL Server Schema

### 7.1 Configuration Tables (2)

| Table | Key columns |
|---|---|
| `CONFIG_Paramgen` | Runtime parameters: Language, multiThread, UnitCostFlag, ImportDay, Log_*, etc. |
| `CONFIG_Tables_SO99` | Table name mapping between PEM and INTER_SO99 for export |

### 7.2 Input Tables – IN_* (17)

| Table | Source | PK / Key |
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

### 7.3 Work Tables – WRK_* (4)

| Table | Purpose |
|---|---|
| `WRK_NECESSITIES` | Calculated necessities with W0/W1/W2, NM, NM_MOV, NM_COMP |
| `WRK_RESTRICTIONS` | Applied restrictions (action: ADVANCE/BUY/NO_BUY/TRANSFER/CEN_NO_BUY) |
| `WRK_SUBSTITUTIONS` | Calculated substitutions between items |
| `WRK_CENTRALIZATION` | Generated centralizations |

### 7.4 Output Tables – OUT_* (2)

| Table | Destination |
|---|---|
| `OUT_PURCHASE_ORDERS` | Purchase orders → ERP |
| `OUT_TRANSFER_ORDERS` | Transfer orders → ERP |

### 7.5 Backup Tables – BACK_* (6)

`BACK_IN_PROPOSALS1_DATE`, `BACK_LOGS`, `BACK_OUT_PURCHASE_ORDERS`, `BACK_OUT_TRANSFER_ORDERS`, `BACK_WRK_CENTRALIZATION`, `BACK_WRK_SUBSTITUTIONS`

### 7.6 Logging Tables – LOG_* (3)

`LOGS`, `LOG_TYPE_ACCIONS`, `LOG_ACTIONS_MESSAGE`

---

## 8. Data Flow between Databases

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
- **Planning**: `PROPOSALS1_DATE` (filtered by active Plangroups for the day)
- **Substitutions**: `SUPERSSESIONS`
- **Segmentation**: Custom SO99 fields
- **TransferOrders**: Transfer orders between warehouses

### ② ERP → PEM (ProposalsBCC.load_Info_ERP)

On user demand:
- `IN_ITEM_EQUIVALENCES`, `IN_ITEM_SEGMENTATION`, `IN_ITEM_STOCK_IMPL`
- `IN_RULE_EQUIVALENCES`, `IN_RED_LOG`
- `IN_SUPP_ITEM_MOV`, `IN_SUPP_LOCATION_COMPRA`, `IN_SUPP_LOCATION_PORTES`, `IN_SUPP_LOCATION_PROMOTIONS`

### ③ PEM → ERP (ExportDAL)

- **BulkInsert**: `OUT_PURCHASE_ORDERS`, `OUT_TRANSFER_ORDERS`, `OUT_SUBSTITUTIONS`
- **Stored Procedure**: Moves data from OUT tables to equivalent tables in INTER_SO99
- Table name mapping managed by `CONFIG_Tables_SO99`

---

## 9. System Configuration

### Files

| File | Format | Purpose |
|---|---|---|
| `Config/connection.conf` | `key:value` | 3 SQL connection strings (SO99, PEM, ERP) |
| `Config/settings.conf` | `key:value` | General parameters |
| `Config/columns.conf` | `key: value1, value2, ...` | Visible columns per grid type |
| `Config/Languages/language_ES.conf` | `"key"="value"` | Spanish translations |
| `Config/Languages/language_EN.conf` | `"key"="value"` | English translations |
| `app.config` | XML | DevExpress skin, .NET Framework, logging |

### connection.conf

```
DataBaseName_SO99: <name>      DatabaseServer_SO99: <server>
SQLUser_SO99: <user>           SQLPwd_SO99: <pwd>        # Empty = AD Auth

DataBaseName_PEM: <name>       DatabaseServer_PEM: <server>
SQLUser_PEM: <user>            SQLPwd_PEM: <pwd>

DataBaseName_ERP: <name>       DatabaseServer_ERP: <server>
SQLUser_ERP: <user>            SQLPwd_ERP: <pwd>
```

### settings.conf

```
Language: ES|EN
multiThreading: true|false
HorizontVisualizationTimeDays: 90
UnitCostFlag: 0|1              # 0=UnitCost, 1=PurchasePrice
```

### columns.conf

Defines visible columns for each view type:
- `colsGeneral` — General proposal view
- `colsSO99` — SO99 specific view
- `colsSub` — Substitutions view
- `colsCentra` / `colsCentraDetail` — Centralization views
- `colsTransferSO99` — TransferOrders view
- `colsSubstitute` — Substitution candidates grid
- `colsInterDepot` — Inter-warehouse transfer grid

---

## 10. Logging System

### Architecture

```
LogBCC
├── Text Mode (startup)
│   └── Files in /Log/ with LogMgmt
└── DB Mode (after connection)
    ├── ConcurrentQueue(Of LogEntity) in memory
    ├── Timer that periodically flushes the queue
    ├── LOGS table in PEM DB
    └── Auxiliary tables: LOG_TYPE_ACCIONS, LOG_ACTIONS_MESSAGE
```

### Log Levels

| Level | Enum | Usage |
|---|---|---|
| Info | `LogTypeModeEnum.Info` | Normal operations, timings |
| Warning | - | Non-critical alerts |
| Error | - | Caught exceptions |
| User | `LogType.User` | User-visible actions |
| UserDetail | `LogType.UserDetail` | Operation detail |
| Debug | `LogType.Debug` | Development only |

### Action Types (LogTypeActionEnum)

`Import`, `Export`, `Load_data`, `Load_UI`, `Load_Server`, `Compensa_Local`, `Compensa_Network`, `Min_Lot`, `Min_Compra`, `Min_Portes`, `ValidateRule`, `AddCandidate`, `AddToList`, `redLog`, `Trans_Order`, `Centr`, `Promo`, `manualSolveRes`

### LOGS Table

```
LOG_ID, DATE_LOG, TYPE_MODE, ITEM_ID, WHOUSE, WHOUSE_DEST
SUPPLIER, IDX, TYPE_ACTION, ACTION_MSG, EXTRA_INFO
EXCEPTION_MSG, COMPUTER, USER_PEM, CLASS_NAME, FUNCTION_NAME
```

---

## 11. Configuration Parameters (ParamGen)

Managed by `ParamGenBCC` and `ParamGenBEC`, persisted in `CONFIG_Paramgen`.

| Parameter | Type | Description | Default |
|---|---|---|---|
| `Language` | String | Language (ES/EN) | ES |
| `multiThread` | Boolean | Parallel.ForEach in calculations | true |
| `HorizontVisualizationTimeDays` | Integer | Visible days (0=all) | 90 |
| `UnitCostFlag` | Boolean | true=PurchasePrice, false=UnitCost | false |
| `numDaysToClear_Log_Info` | Integer | Days to purge logs | 30 |
| `ImportDay` | Date | Import date | Today |
| `skipImport` | Boolean | Skip import | false |
| `Log_User` | Boolean | Enable user log | true |
| `Log_UserDetail` | Boolean | Enable detailed log | false |
| `Log_Debug` | Boolean | Enable debug log | false |
| `Log_num_Queue` | Integer | Log queue size | 100 |
| `Adv_ABC_Restriction` | String | ABC filter for advances | "" |

---

## 12. UI Structure and Forms

### Main Forms

| Form | Purpose |
|---|---|
| `MainFormNEW` + `MainFormNEW_AUX` | Main screen: hierarchical tree, toolbars, background workers |
| `ConsoleForm` | Server/console mode (without full graphical UI) |
| `LogsForm` | Historical log viewer |
| `configSQLConnexionForm` | SQL connection configuration (3 tabs) |

### Data Forms

| Form | Data displayed |
|---|---|
| `ProposalsForm` | Warehouse proposal grid |
| `SubsProposalsForm` | Substitution candidates grid |
| `MovimentsPorposalsForm` | TransferOrders and Centralizations grid |

### Dialogs

| Dialog | Purpose |
|---|---|
| `RuleCompensaDialog` | Compensation rules CRUD |
| `promoDialog` | Promotion management |
| `statusDIalog` | Necessity status display/editing |

### Tree Nodes (SummaryNodes)

| Component | Content |
|---|---|
| `Plangroup_Node_Info` | Plangroup summary: color, supplier count, proposals |
| `Supplier_Node_Info` | Supplier summary: color, warehouses, general status |
| `Whouse_Node_Info` | Warehouse summary: color, proposal count |
| `NodeUtils` | Shared utilities: color calculation, status aggregation |

### Restrictions Panel

| Form | Purpose |
|---|---|
| `Restriction_NecessitiesForm` | Review/resolution panel for MANP3/4/5 |

### Color Scheme (ColorsPEM)

| Class | Responsibility |
|---|---|
| `ColorStatus` | Mapping `Necessity_Status` → Color (border, background, icon) |
| `ColorsBasic` | Application base colors |
| `ColorsProposalForm` | Proposal grid specific colors |
| `ColorsRestrictionForm` | Restrictions panel colors |
| `ColorsRulesDialog` | Rules dialog colors |

---

## 13. Multithreading and Performance

### Parallelism in Calculations

Controlled by `ParamGen.multiThread`:

```
GoOverProposalsParallel(typeOperation, sourceEquivalent)
  └── Parallel.ForEach(Plangroups)
       └── Parallel.ForEach(Whouses)      # Only if Compensa_Local/Network
            └── Parallel.ForEach(Proposals) # Only if Compensa_Local/Network
```

For `Min_Lot` / `Min_Compra` / `Min_Portes`, parallelism is at Supplier level:
```
  └── Parallel.ForEach(Plangroups)
       └── calcMinimums(supplier)          # Sequential per supplier
```

### Profiling

The `UTIL.timerCount` class provides time measurement:
```vb
Dim timer As New UTIL.timerCount("OperationName")
timer.startCount("Step1")
' ... work ...
timer.stepTime("Step2")
' ... work ...
timer.stopAndPrint()
```

### BulkInsert

Export uses `SqlBulkCopy` for bulk insertion into `OUT_PURCHASE_ORDERS` and `OUT_TRANSFER_ORDERS`.

---

## 14. Appendix: Enums, Constants and Utilities

### EnumsBCC

#### Equivalent_Type
| Value | Code | Meaning |
|---|---|---|
| S | 1 | Substitute (SuperSession: obsolete product → new) |
| I | 2 | Identical (same product, different warehouse) |
| D | 3 | Different/Equivalent (different product, same function) |

#### Equivalent_Expensive
| Value | Meaning |
|---|---|
| YES (1) | Expensive products only |
| NO (2) | Cheap products only |
| ALL (3) | No price restriction |

#### Equivalent_Travel
| Value | Meaning |
|---|---|
| YES (1) | Can travel between warehouses |
| NO (2) | Cannot travel |
| NONE (3) | No restriction |

#### Item_Aplication
Sorted by importance (higher = more restrictive):
| Value | Code | Meaning |
|---|---|---|
| 0 | ALL | All products |
| 1 | SECC | Section |
| 2 | LINE | Line |
| 3 | FAMI | Family |
| 4 | SUBF | SubFamily |
| 5 | MARC | Brand |
| 6 | SEGM | Segmentation |
| 7 | FAMA | Family + Brand |
| 8 | ITEM | Specific product |

#### ItemLocation_LevelWhouse
| Level | Meaning |
|---|---|
| N0 (0) | Same warehouse |
| N1 (1) | Same region |
| N2 (2) | Other region (regional) |
| N3 (3) | Remaining warehouses |
| NoLevel (-1) | No relationship |

#### Necessity_Status
| Status | Code | Meaning |
|---|---|---|
| NO_PROCESSED | 0 | Not processed |
| SI | 100 | Covered |
| NO | 200 | No necessity |
| MANP3 | 30 | Minimum lot issue |
| MANP3_V | 31 | MANP3 reviewed |
| MANP4 | 41 | Minimum purchase issue |
| MANP4_V | 42 | MANP4 reviewed |
| MANP5 | 51 | Minimum portes issue |
| MANP5_V | 52 | MANP5 reviewed |
| MANP5_C | 53 | Centralized |
| WAIT | 60 | Waiting |
| TRANSFER_SO99 | 70 | TransferOrder SO99 |
| NOT_BUY | 80 | Do not buy |
| TO_EXPORT | 90 | Marked for export |
| EXPORTED | 95 | Exported |

#### Restriction_Action
| Action | Code | Meaning |
|---|---|---|
| ADVANCE | 0 | Advance quantity to a future period |
| BUY | 1 | Force purchase (ignore minimum) |
| NO_BUY | 2 | Do not buy |
| TRANSFER | 3 | Transfer from another warehouse |
| CEN_NO_BUY | 4 | Centralize without buying |

#### Plangroup_TypeAccum
| Type | Meaning |
|---|---|
| ACCUM (1) | Centralizes: supplier does not split orders |
| NO_ACCUM (2) | Does not centralize: supplier accepts separate orders |
| ALL_WHOUSE (3) | Delivers to all warehouses |

#### Plangroup_LevelCenter
| Level | Meaning |
|---|---|
| REG (1) | Centralizes to Regional |
| CEN (2) | Centralizes directly to Central |

### UTIL

| Class | Key methods |
|---|---|
| `StringBuilderDB` | `append()`, `IN_List()`, `dateToSql()`, `strToSql()`, `numToSql()`, `boolToSql()` |
| `Utils` | `SQLStr()`, `styleGrid()`, `timerCount` (profiling), `countRowsAffectedMessage()` |
| `timerCount` | `startCount()`, `stepTime()`, `stopCount()`, `stopAndPrint()`, `getTotalTime()` |