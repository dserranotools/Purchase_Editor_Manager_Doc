[← README](./README_en_one-doc.md)

&nbsp;

# PEM - Purchase Editor Manager
## Technical and Business Specifications Document

> **Version:** 1.0  
> **Project:** PurchaseEditorManager (PEM)  
> **Technology:** VB.NET WinForms (.NET Framework 4.8) + SQL Server  
> **UI Components:** DevExpress WinForms (Skin Office 2016 Colorful)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Class Model – BEC Layer (Business Entity Classes)](#3-class-model--bec-layer-business-entity-classes)
4. [Class Model – DAL Layer (Data Access Layer)](#4-class-model--dal-layer-data-access-layer)
5. [Class Model – BCC Layer (Business Control Classes)](#5-class-model--bcc-layer-business-control-classes)
6. [Data Model – SQL Server Schema](#6-data-model--sql-server-schema)
7. [Data Flow Between Databases](#7-data-flow-between-databases)
8. [System Configuration](#8-system-configuration)
9. [Business Logic – Proposal Processing](#9-business-logic--proposal-processing)
10. [Substitution, Equivalences and Compensation System](#10-substitution-equivalences-and-compensation-system)
11. [Logistics Networks and Warehouse Levels](#11-logistics-networks-and-warehouse-levels)
12. [Centralization by Batch and by Purchase](#12-centralization-by-batch-and-by-purchase)
13. [Necessity Status (MANP3, MANP4, MANP5)](#13-necessity-status-manp3-manp4-manp5)
14. [Compensation Rules (RuleEquivalences)](#14-compensation-rules-ruleequivalences)
15. [Import Process from SO99](#15-import-process-from-so99)
16. [Export Process to ERP](#16-export-process-to-erp)
17. [User Journey – Planner Workflow](#17-user-journey--planner-workflow)
18. [Logging System](#18-logging-system)
19. [Configuration Parameters (ParamGen)](#19-configuration-parameters-paramgen)
20. [UI Structure and Forms](#20-ui-structure-and-forms)

---

## 1. Project Overview

### What is PEM?

**Purchase Editor Manager (PEM)** is a Windows desktop application (WinForms) that acts as an **intelligent intermediary between the SO99+ planning system and the client's ERP**. Its main function is to:

1. **Import purchase proposals** from SO99+ (demand planning system)
2. **Calculate and optimize** these proposals by applying business rules:
   - Netting of quantities (adjust Qty based on actual stock)
   - Product substitution (discontinued items → equivalent items)
   - Compensation between warehouses (transfers of surplus stock)
   - Adjustment to delivery minimums (minimum batch, minimum purchase, minimum shipping)
   - Order centralization (by batch and by purchase)
3. **Present to the planner** a hierarchical view (Plangroup → Supplier → Warehouse → Proposal) with visual status indicators
4. **Allow manual review** of issues (MANP3, MANP4, MANP5) and their resolution
5. **Export purchase orders and transfer orders** to the ERP (intermediate database INTER_SO99)

### Databases Involved

| Database | Role | Example |
|---|---|---|
| **SO99+** | Source of planning data (proposals, masterfile, stocks) | `20244_DS_EmmaSleep` |
| **PEM** | Own working database (import, calculation, results) | `BK_PEM_Silvestre_863_19259` |
| **ERP (INTER)** | Intermediate database for exporting results to the ERP | `800_DS_Silvestre_INTER_SO99_11052018` |

### High-Level Flow

```
SO99+ → [Import] → PEM (calculations and review) → [Export] → INTER_SO99 → ERP
```

---

## 2. System Architecture

### Layered Architecture (Logical 3-Tier on the Same Machine)

```
┌─────────────────────────────────────────────────────┐
│                    UI Layer                          │
│  MainFormNEW, ProposalsForm, Dialogs, SummaryNodes  │
│  (DevExpress WinForms + ColorStatus)                │
├─────────────────────────────────────────────────────┤
│                  BCC Layer                           │
│  Business Control Classes (PurchaseEditorBCC,       │
│  ProposalsBCC, ItemBCC, SupplierBCC, etc.)          │
│  → Orchestrate business logic                       │
├─────────────────────────────────────────────────────┤
│                  DAL Layer                           │
│  Data Access Layer (SQL Server, SqlClient)          │
│  → Direct access to the 3 databases                 │
├─────────────────────────────────────────────────────┤
│                  BEC Layer                           │
│  Business Entity Classes (POCOs/DTOs)               │
│  → Transport data between layers                    │
└─────────────────────────────────────────────────────┘
```

### Dependencies Between Layers

- **UI → BCC**: Forms create and use `PurchaseEditorBCC` as the main facade
- **BCC → DAL**: BCC classes instantiate DAL to read/write to the database
- **BCC → BEC**: Use BEC entities as data containers
- **DAL → BEC**: Build BEC entities from SQL results
- **Global (AppConf)**: Shared context with connections, logging, language, parameters

### Entry Point

[PurchaseEditorMain.vb](/PurchaseEditor/PurchaseEditorMain.vb) → `Main()`:
1. Creates `AppConf` (global context)
2. Loads `connection.conf`
3. Tests SQL connection; if it fails, shows `configSQLConnexionForm`
4. Loads `settings.conf`, `language_XX.conf`, `columns.conf`, `ParamGen`
5. Starts the UI based on command-line arguments:
   - No args → `MainFormNEW`
   - `\Logs` → `LogsForm`
   - `\Console` → `ConsoleForm` (server/console mode)

---

## 3. Class Model – BEC Layer (Business Entity Classes)

**Business Entity Classes** are plain objects that represent domain data. Many inherit from `BaseEntity`, which provides modification tracking and editing state.

### 3.1 Main Entities

#### `ItemMasterFileEntity` — Product Master File
| Field | Description |
|---|---|
| `ItemID` | Unique product code |
| `RefFabr` | Manufacturer reference |
| `Descr` | Product description |
| `Mark` | Brand (FREEATTR1) |
| `Section` | Section (FREEATTR2) |
| `Line` | Line (FREEATTR3) |
| `Family` | Family (MACROFAM) |
| `SubFamily` | SubFamily (PRODFAM) |
| `FamilyMark` | Family + Brand combined |
| `Plangroup` | Assigned planning group |
| `StatusFabr` | Manufacturer status: ACTIVO (active), HASTA FIN (until end), Unknown |
| `Supplier` | Main supplier (FREEATTR5) |
| `ScopeDistribution` | Distribution scope: _ALL, _ALL_R, _REG (FREEATTR6) |
| `RedLog` | Assigned logistics network (FREEATTR7) |
| `ABC_General` | General ABC classification (FREEATTR8) |
| `Segmentation` | List of product segmentations |
| `Unit`, `UnitVol`, `UnitCube`, `UnitWeight`, `PalletQty` | Logistics data |

#### `ItemLocationEntity` — Product in a Warehouse
| Field | Description |
|---|---|
| `ItemID` + `Location` | Compound key (SKUL) |
| `LevelWhouse` | Level in the logistics network (N0, N1, N2, N3, NoLevel) |
| `Whouse` | Warehouse |
| `Supplier` | Supplier |
| `stockTotal` | Total virtual stock (VIRTSTK) |
| `stockTotal_Ini` | Initial virtual stock (before operations) |
| `AveDemQty` | Deseasonalized average demand |
| `coverage` | Desired minimum coverage days |
| `OverStkQty_Projected` | Quantity available for exchange (projected) |
| `OverStkQty_AvedemQty` | Quantity available for exchange (avg demand) |
| `UnitCost` | Unit cost |
| `UnitCost_WithOutRapel` | Actual cost without rappel |
| `minResLot` | Minimum reserve/transfer batch |
| `incresLot` | Batch increment |
| `qtyMinOverStok` | Minimum stock frozen in the warehouse |
| `ABC_SKU` | SKU ABC classification |
| `expensive` | Is it expensive? (S=yes/N=no/ALL=all) |
| `travel` | Can it travel? (S=yes/N=no/NONE=none) |
| `leadTime` | Product lead time in the warehouse |
| `isProjSTK_OverMaxOutflow` | Is projected stock > MaxOutflow? |

#### `ProposalEntity` — Purchase Proposal
| Field | Description |
|---|---|
| `getKey` | Key: Plangroup_Supplier_Location_Item_IdxFinal |
| `SKUL` | Key: Plangroup_Supplier_Location_Item |
| `ItemID`, `WhouseID`, `LocationID`, `SupplierID`, `PlangroupID` | Identifiers |
| `IndProp` | SO99 proposal index |
| `DueDate`, `ReleaseDate` | Delivery and order dates |
| `StockActual` | Current stock in the warehouse |
| `QPropSO99` | Original proposed quantity from SO99 |
| `QProp`, `QPropCost` | Proposed quantity and its cost |
| `QtySust`, `QtySustCost` | Substituted quantity and its cost |
| `QtyAdv` | Advanced quantity |
| `QtyCent` | Centralized quantity |
| `Qty_MinItem` | Minimum quantity per item (batch) |
| `Qty_MinCompr` | Minimum purchase quantity |
| `Qty_MinPort` | Minimum shipping quantity |
| `QtyFinal`, `QtyFinalCost` | Final net quantity |
| `UnitCost` | Applied unit cost |
| `IdxProp` | Proposal index |
| `IdxMinDel` | Minimum delivery index |
| `IdxFinal` | Final index |
| `tratar` | Necessity status (enum `Necessity_Status`) |
| `CandidateListAux` | List of substitution candidates for this proposal |
| `SubstituteList` | Final list of applied substitutions |
| `SRC_Whouse`, `SRC_LocationID` | Origin for SO99 TransferOrders |
| `ProposalLink`, `TransferLink` | Link between proposals and transfers |

#### `SubstituteProposal` — Substitution Candidate
| Field | Description |
|---|---|
| `ItemID_Ori`, `Location_Orig` | Origin product and location (the one yielding stock) |
| `ItemID_Dest`, `Location_Dest` | Destination product and location (the receiver) |
| `RefFabr`, `Descr`, `Mark` | Candidate product data |
| `TotalStockInitial`, `TotalStock` | Initial and current total stock |
| `Coverage` | Coverage days |
| `OverStockInitial`, `OverStock` | Initial and current overstock |
| `EquivalentStock` | Equivalent stock available for transfer |
| `EquivalentStockInitial` | Initial equivalent stock |
| `EquivalentStockUsed` | Equivalent stock already used |
| `Level` | Warehouse level (N0=same, N1=same region, N2=other region, N3=rest) |
| `TypeEquivalence` | Type: S=Substitute (Sustituto), I=Same (Igual), D=Different/Equivalent (Equivalente) |
| `minResLot` | Minimum transfer batch |
| `qtyMinOverStok` | Minimum frozen stock |
| `ruleID` | ID of the compensation rule that validated this candidate |
| `Qty` | Quantity to transfer |
| `isOverStock` | Whether overstock is being used (vs. total stock) |

#### `NecessityEntity` — Purchase Necessity
Represents a calculated net necessity that must be covered.

| Field | Description |
|---|---|
| `W0` | Original proposal location |
| `W1` | Intermediate location (regional) |
| `W2` | Final delivery location |
| `Item` | Product code |
| `Supplier` | Supplier |
| `P0` | Original proposed quantity (SO99) |
| `N0` | Net necessity (Proposal QtyFinal) |
| `NM` | Modified net necessity (after manual adjustments) |
| `NM_MOV` | Modified net necessity by item-location |
| `NM_COMP` | Modified net necessity by supplier-location purchase |
| `Tratar` | Necessity status |

#### Support Entities
- `TransferOrderEntity` — Transfer order between warehouses
- `CentralizationEntity` — Order centralization
- `RedLogEntity` — Logistics network node (Location → Regional → Central)
- `SKU_Equiv` — SKU representation for equivalence rules
- `SupplierEntity`, `SuppItem_MovEntity`, `SuppLocation_CompraEntity`, `SuppLocation_PortesEntity`, `SuppLocation_PromoEntity` — Supplier data

### 3.2 Collections (Dictionary-based)
- `ProposalPlangroupList` — `Dictionary(Of PlangroupID, ProposalPlangroupEntity)`
- `ProposalPlangroupEntity` → `supplierList` → `whouseList` → `idxProposalList(Idx)`
- `ItemMasterFileList` — `Dictionary(Of ItemID, ItemMasterFileEntity)`
- `ItemLocationList` — `Dictionary(Of "ItemID_Location", ItemLocationEntity)`
- `SubstitutesList` — `Dictionary(Of "ItemID@Location", List(Of SubstituteEntity))`
- `SubstituteProposal_List` — `Dictionary(Of Key, SubstituteProposal)`
- `EquivalentList` — `Dictionary(Of ItemID, List(Of EquivalentItemID))`

---

## 4. Class Model – DAL Layer (Data Access Layer)

The Data Access Layer classes encapsulate all SQL Server access using `SqlClient` and `SQLServerHelper`.

| DAL Class | Responsibility | Connections |
|---|---|---|
| `ItemDAL` | Reading Items, ItemLocations, Substitutes, Equivalents, Implementation Stock, MasterFile | SO99, PEM |
| `ProposalsDAL` | Reading/writing proposals (IN_PROPOSALS, TransferOrders) | SO99, PEM |
| `SupplierDAL` | Reading supplier data: movements, minimum purchases, shipping, promotions | PEM |
| `PlangroupsDAL` | Reading/writing Plangroups and their planning frequency | SO99, PEM |
| `RedLogisticaDAL` | Reading the logistics network (IN_RED_LOG) | PEM |
| `ImportDAL` | Import process: backup, cleanup and insertion of proposals and TransferOrders | SO99, PEM |
| `ExportDAL` | Export to ERP: PurchaseOrders, TransferOrders (PEM + SO99), Substitutions, clearing OUT tables | PEM, ERP |
| `MovimentsDAL` | Reading movements (TransferOrders, Centralizations) | PEM |
| `ConfigDAL` | Reading configuration: ParamGen, ERP mapping tables | PEM |
| `LogDAL` | Writing logs to database (LOGS) | PEM |
| `IOFiles` | Reading configuration files (.conf) from the file system | N/A |
| `RuleEquivalencesDAL` | Reading equivalence/compensation rules | PEM |
| `SKU_EquivDAL` | Reading equivalences between SKUs | PEM |

---

## 5. Class Model – BCC Layer (Business Control Classes)

### 5.1 Main Orchestrator: `PurchaseEditorBCC`

Main facade used by the UI. Coordinates all subsystems:

```
PurchaseEditorBCC
├── ProposalsBCC      → Calculations, substitutions, minimums, export
├── SupplierBCC       → Promotions, costs
└── RedLogisticaBCC   → Logistics network
```

### 5.2 Calculation Engine: `ProposalsBCC`

The largest and most complex class in the system (~1161 lines). Responsibilities:

- Initial loading of proposals and TransferOrders
- Calculation of SO99 TransferOrders (linking proposals → transfers)
- Netting of proposals with substitutes/equivalents
- Search for substitution candidates (Local + Network)
- Validation of candidates against compensation rules
- Calculation of delivery minimums (MinLot, MinCompra, MinPortes)
- Generation of TransferOrders and Centralizations
- Marking for export
- Export of PurchaseOrders and TransferOrders

**Key Methods:**
| Method | Description |
|---|---|
| `loadInit()` | Loads all DALs, dependent BCCs, TransferOrders and proposals |
| `calcTrasnferOrdersSO99()` | Links SO99 TransferOrders with purchase proposals |
| `calcSustAll_Local()` | Calculates substitutions within the same warehouse |
| `calcSustAll_Network()` | Calculates substitutions across the logistics network |
| `CalcMinDel()` | Calculates delivery minimums (batch, purchase, shipping) |
| `markToExport_*()` | Marks proposals for export |
| `saveExportProposals()` | Inserts into OUT_PURCHASE_ORDERS |
| `saveExportTransferOrders_PEM()` | Inserts into OUT_TRANSFER_ORDERS |

### 5.3 Product Support: `ItemBCC`

- Loads and manages `ItemMasterFileList`, `ItemLocationList`, `SubstitutesList`, `EquivalentList`
- Queries implementation stock (`IN_ITEM_STOCK_IMPL` restrictions)
- Filters ItemLocations by stock > 0 and exclusions
- Verifies minimum batches and minimum transfer Qty

### 5.4 Supplier Support: `SupplierBCC`

- Manages `SuppItem_MovList` (minimums per item-supplier)
- Manages `SuppLocation_ComprasList` (minimum purchases per supplier-location)
- Manages `SuppLocation_PortesList` (minimum shipping)
- Manages `SuppLocation_PromoList` (promotions per supplier-location)
- Hierarchical search: first Supplier_Location, then Supplier_ALL

### 5.5 Logistics Network: `RedLogisticaBCC`

- Loads `RedLogMap`: hierarchical network Location → Regional → Central
- Determines levels between warehouses (N0, N1, N2, N3)
- Queries regional and central warehouses
- Auto-registration of undefined warehouses (with warning)

### 5.6 Movements: `MovimentsBCC`

- Manages `TransferOrdersList` (generated transfer orders)
- Manages `CentralizationList` (generated centralizations)
- Allows adding/removing TransferOrders and Centralizations

### 5.7 Plangroups: `PlangroupsBCC`

- Loads the list of Plangroups from SO99
- Queries planning frequency
- Determines whether a Plangroup centralizes to REG (Regional → Central) or CEN (direct to Central)

### 5.8 Equivalence Rules: `RuleEquivalencesBCC`

- Loads and manages compensation rules
- `validateRule()`: evaluates each candidate against all applicable rules
- Selects the most restrictive rule that passes
- Checks: WhouseOri, WhouseDest, Application, ABC_gen, ABC_SKU, Expensive, Travel, TotalStock/OverStock

### 5.9 Import: `ImportBCC`

- Orchestrates the import process from SO99:
  1. Backup current tables
  2. Cleanup obsolete data
  3. Import Masterdata
  4. Import support tables
  5. Import today's proposals
  6. Import TransferOrders

### 5.10 Logging: `LogBCC`

- Logging system with in-memory queue and database dump
- Levels: Info, Warning, Error, User, UserDetail, Debug
- Support for proposal-specific logging

---

## 6. Data Model – SQL Server Schema

### System Tables (33 tables total)

#### Configuration Tables
| Table | Description |
|---|---|
| `CONFIG_Paramgen` | Runtime configuration parameters (language, flags, dates) |
| `CONFIG_Tables_SO99` | Table name mapping between PEM and INTER_SO99 |

#### Input Tables (IN_*)
| Table | Origin | Description |
|---|---|---|
| `IN_T_PROPOSALS1_DATE` | SO99 | Imported purchase proposals |
| `IN_C_MASTERFILE` | SO99 | Product master file |
| `IN_C_ITEMLOCATION` | SO99 | Products by location/warehouse |
| `IN_C_ITEMSUPPLIER` | SO99 | Product-supplier relationship |
| `IN_C_SUPPLIERS` | SO99 | Suppliers |
| `IN_C_PLANGROUPS` | SO99 | Planning groups |
| `IN_C_WAREHOUSES` | SO99 | Warehouses |
| `IN_ITEM_SUPERSSESIONS` | SO99 | Substitutions (obsolete → new products) |
| `IN_ITEM_EQUIVALENCES` | ERP | Equivalences between products |
| `IN_ITEM_SEGMENTATION` | ERP | Product segmentation |
| `IN_ITEM_STOCK_IMPL` | ERP | Stock restrictions by implementation |
| `IN_RULE_EQUIVALENCES` | ERP | Compensation/equivalence rules |
| `IN_SUPP_ITEM_MOV` | ERP | Minimum movements per item-supplier |
| `IN_SUPP_LOCATION_COMPRA` | ERP | Minimum purchases per supplier-location |
| `IN_SUPP_LOCATION_PORTES` | ERP | Minimum shipping per supplier-location |
| `IN_SUPP_LOCATION_PROMOTIONS` | ERP | Promotions per supplier-location |
| `IN_RED_LOG` | ERP | Logistics network (Location → Regional → Central) |

#### Working Tables (WRK_*)
| Table | Description |
|---|---|
| `WRK_NECESSITIES` | Calculated necessities (W0, W1, W2, NM, NM_MOV, NM_COMP) |
| `WRK_RESTRICTIONS` | Restrictions applied to necessities |
| `WRK_SUBSTITUTIONS` | Calculated substitutions |
| `WRK_CENTRALIZATION` | Generated centralizations |

#### Output Tables (OUT_*)
| Table | Description |
|---|---|
| `OUT_PURCHASE_ORDERS` | Purchase orders ready for export to ERP |
| `OUT_TRANSFER_ORDERS` | Transfer orders ready for export to ERP |

#### Backup Tables (BACK_*)
| Table | Description |
|---|---|
| `BACK_IN_PROPOSALS1_DATE` | Backup of proposals before cleanup |
| `BACK_LOGS` | Backup of old logs |
| `BACK_OUT_PURCHASE_ORDERS` | Backup of exported purchase orders |
| `BACK_OUT_TRANSFER_ORDERS` | Backup of exported transfer orders |
| `BACK_WRK_CENTRALIZATION` | Backup of centralizations |
| `BACK_WRK_SUBSTITUTIONS` | Backup of substitutions |

#### Logging Tables
| Table | Description |
|---|---|
| `LOGS` | Main operations log |
| `LOG_TYPE_ACCIONS` | Action types for the log |
| `LOG_ACTIONS_MESSAGE` | Predefined log messages |

---

## 7. Data Flow Between Databases

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  SO99+   │     │   PEM    │     │ERP/INTER │
│ (Source) │     │ (Working)│     │(Target)  │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                 │                 │
     │  ① IMPORT       │                 │
     │────────────────►│                 │
     │  Proposals,     │                 │
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

### ① Import from SO99
- `ImportDAL.proposalsImport()`: Reads `IN_PROPOSALS1_DATE` and filters by active Plangroups
- `ImportDAL.import_IN_Masterdata_Table()`: Loads masterfiles, itemlocations, etc.
- `ImportDAL.import_IN_Table()`: Loads suppliers, warehouses, segmentations, superSessions
- `ImportDAL.transferOrdersImport()`: Loads TransferOrders from SO99

### ② Import from ERP
- Runs on demand ("Load ERP Info" button)
- Loads equivalences, rules, segmentations, stock restrictions, logistics network
- Method: `ProposalsBCC.load_Info_ERP()`

### ③ Export to ERP
- `ExportDAL.exportProposals()`: Executes stored procedure that moves `OUT_PURCHASE_ORDERS` → INTER_SO99
- `ExportDAL.exportTransferOrders_PEM()`: Similar for `OUT_TRANSFER_ORDERS`
- `ExportDAL.exportTransferOrders_SO99()`: Exports SO99-specific TransferOrders
- `ExportDAL.exportSubstitutions()`: Exports `OUT_SUBSTITUTIONS`

---

## 8. System Configuration

### Configuration Files

| File | Purpose |
|---|---|
| `Config/connection.conf` | Connection strings to the 3 databases |
| `Config/settings.conf` | General parameters (language, multithreading, horizon, etc.) |
| `Config/columns.conf` | Visible columns and order in UI grids |
| `Config/Languages/language_ES.conf` | Spanish translations |
| `Config/Languages/language_EN.conf` | English translations |
| `app.config` | .NET configuration (DevExpress, Framework, logging) |

### connection.conf Format
```
DataBaseName_SO99: <name>
DatabaseServer_SO99: <server>
SQLUser_SO99: <user>       # Empty = Windows Auth
SQLPwd_SO99: <password>

DataBaseName_PEM: <name>
DatabaseServer_PEM: <server>
SQLUser_PEM: <user>
SQLPwd_PEM: <password>

DataBaseName_ERP: <name>
DatabaseServer_ERP: <server>
SQLUser_ERP: <user>
SQLPwd_ERP: <password>
```

---

## 9. Business Logic – Proposal Processing

### 9.1 Data Hierarchy

Proposals are organized in a 4-level hierarchy:

```
Plangroup (Planning Group)
  └── Supplier
       └── Warehouse/Whouse
            └── Proposal (indexed by IdxProp)
```

Each proposal has an `IdxProp` (0=main, 1=alternative, 2=second alternative, etc.).

### 9.2 Main Calculation Flow

```
1. Data Loading (loadInit)
   ├── Load SO99 TransferOrders
   ├── Load proposals from database
   ├── Load Items, ItemLocations
   ├── Load Logistics Network
   ├── Load Equivalence Rules
   ├── Load Movements
   ├── Load Plangroups
   └── Load Suppliers

2. SO99 TransferOrders Calculation (calcTrasnferOrdersSO99)
   └── Links proposals with SO99 TransferOrders
       └── Nets quantities

3. Local Substitutions (calcSustAll_Local)
   └── For each proposal with QtySust > 0:
       ├── resetProposal()
       ├── Search substitutes (type S) in same warehouse → validateCandidates
       ├── Search equivalents (type D) in same warehouse → validateCandidates
       └── netificaProposals() [net proposals]

4. Network Substitutions (calcSustAll_Network)
   └── For each proposal with QtySust > 0:
       ├── Search substitutes (type S) across the entire network → validateCandidates
       ├── Search same items (type I) across the entire network → validateCandidates
       ├── Search equivalents (type D) across the entire network → validateCandidates
       ├── netificaProposals() [net proposals]
       └── Generate TransferOrders for transfers

5. Delivery Minimums Calculation (CalcMinDel)
   └── Evaluates minimum batch, minimum purchase and minimum shipping restrictions

6. Promotions Calculation (CalcPromo)
   └── Applies supplier discounts and promotions

7. Export
   └── Mark for export → Insert into OUT_* → Execute export SP
```

### 9.3 Proposal Netting

Netting is the process of adjusting a proposal's `QtyFinal` by subtracting the stock of selected substitute/equivalent candidates.

```
For each validated candidate (sorted by EquivalentStock descending):
    If proposal QtyFinal > 0:
        If candidate EquivalentStock >= QtyFinal:
            candidate.EquivalentStockUsed = QtyFinal
            QtyFinal = 0
        Else:
            candidate.EquivalentStockUsed = EquivalentStock
            QtyFinal -= EquivalentStock
```

---

## 10. Substitution, Equivalences and Compensation System

### 10.1 Equivalence Types

| Type | Code | Description | Example |
|---|---|---|---|
| **Substitute (Sustituto) (S)** | S | Discontinued product replaced by another | Old item → New item (SuperSession) |
| **Same (Igual) (I)** | I | Same product in another warehouse | Same SKU in regional warehouse |
| **Equivalent/Different (Equivalente) (D)** | D | Different product but functionally interchangeable | Brand A → Equivalent Brand B |

### 10.2 Candidate Search Flow

```
Proposal with QtySust > 0
│
├── LOCAL Substitution (same warehouse)
│   ├── 1st Search Substitutes (S)
│   │   └── Search in SubstitutesList items that substitute the proposal's item
│   │       └── For each substitute item warehouse:
│   │           └── If stock > 0 and meets minResLot → Add candidate
│   ├── Validate candidates against rules (whouseUse=1)
│   ├── Net proposal with validated candidates
│   │
│   ├── If QtyFinal > 0: 2nd Search Equivalents (D)
│   │   └── Search in EquivalentList for equivalent items
│   │       └── For each equivalent item warehouse:
│   │           └── If stock > 0 and meets minResLot → Add candidate
│   ├── Validate candidates against rules (whouseUse=1)
│   └── Net proposal
│
└── NETWORK Substitution (all warehouses in the logistics network)
    ├── 1st Search Substitutes (S) across entire network
    ├── Validate candidates against rules (whouseUse=2)
    ├── Net
    ├── 2nd Search Same items (I) across entire network (same item, other warehouse)
    ├── Validate and net
    ├── 3rd Search Equivalents (D) across entire network
    ├── Validate and net
    └── Generate TransferOrders for transfers between warehouses
```

### 10.3 Priority Order

Candidates are processed in order of **highest available equivalent stock** (`OrderByDescending(EquivalentStockInitial)`), which maximizes compensation with the fewest transfers.

---

## 11. Logistics Networks and Warehouse Levels

### 11.1 Logistics Network Structure

The logistics network is defined in `IN_RED_LOG` and has a 3-level hierarchy:

```
Central (CEN)
  ├── Regional 1 (REG)
  │   ├── Warehouse 1.1
  │   ├── Warehouse 1.2
  │   └── Warehouse 1.3
  ├── Regional 2 (REG)
  │   ├── Warehouse 2.1
  │   └── Warehouse 2.2
  └── ...
```

### 11.2 Levels Between Warehouses (ItemLocation_LevelWhouse)

| Level | Meaning | Condition |
|---|---|---|
| **N0** | Same warehouse | `location1 = location2` |
| **N1** | Same region | `regional(loc1) = regional(loc2)` |
| **N2** | Different region (destination's regional) | ScopeDistribution = `_ALL` or (`_ALL_R` and loc2 is regional) |
| **N3** | Rest of warehouses | ScopeDistribution = `_ALL_R` and loc2 is NOT regional |
| **NoLevel** | No valid relationship | Do not share a logistics network |

### 11.3 ScopeDistribution (Distribution Scope)

Comes from the `FREEATTR6` field of the MasterFile:
- `_ALL`: Can use any warehouse in the network
- `_ALL_R`: Can use regionals from other regions + rest
- `_REG`: Can only use warehouses from its own region

---

## 12. Centralization by Batch and by Purchase

### 12.1 Centralization Types (Plangroup_TypeAccum)

| Type | Code | Description |
|---|---|---|
| **ACCUM** | ACCUM | The supplier does not separate orders. Mandatory delivery to upper-level warehouse and creation of downstream TransferOrders |
| **NO_ACCUM** | NO_ACCUM | The supplier accepts separate orders in the same warehouse |
| **ALL_WHOUSE** | (empty) | The supplier delivers to all warehouses. Orders and transfers according to original proposals |

### 12.2 Centralization Levels (Plangroup_LevelCenter)

| Level | Description |
|---|---|
| **REG** | Centralizes first to Regional, then from Regional to Central |
| **CEN** | Centralizes directly to Central (without going through Regional) |

### 12.3 Centralization Flow

```
For each Supplier → Whouse → Proposal (only Idx=0):
    If the proposal requires centralization:
        1. Group quantities by Plangroup/Supplier
        2. Determine centralization warehouse (REG or CEN)
        3. Create CentralizationEntity
        4. Generate TransferOrders from central to destination warehouses
        5. Original proposals are marked as MANP5 or MANP5_C
```

---

## 13. Necessity Status (MANP3, MANP4, MANP5)

### 13.1 Necessity States (Necessity_Status)

| State | Code | Color | Meaning |
|---|---|---|---|
| `NO_PROCESSED` | 0 | Gray | Unprocessed |
| `SI` | 100 | - | OK (necessity covered) |
| `NO` | 200 | - | No necessity |
| **MANP3** | 30 | 🔴 Red | **Minimum batch restriction not met** |
| **MANP3_V** | 31 | 🔴 Red | MANP3 manually validated/reviewed |
| **MANP4** | 41 | 🟠 Orange | **Minimum purchase restriction not met** |
| **MANP4_V** | 42 | 🟠 Orange | MANP4 manually validated/reviewed |
| **MANP5** | 51 | 🟡 Yellow | **Minimum shipping restriction not met** |
| **MANP5_V** | 52 | 🟡 Yellow | MANP5 validated |
| **MANP5_C** | 53 | 🟡 | Centralized |
| `WAIT` | 60 | - | Waiting |
| `TRANSFER_SO99` | 70 | Dark green | TransferOrder from SO99 |
| `TRANSFER_Link` | 75 | - | TransferOrder linked to proposal |
| `TRANSFER_CEN` | 76 | - | Centralization TransferOrder |
| `TRANS_Netifica` | 77 | - | Netted TransferOrder |
| `NOT_BUY` | 80 | - | Do not buy |
| **TO_EXPORT** | 90 | 🔵 Blue | **Marked for export** |
| **EXPORTED** | 95 | 🟣 Purple | **Already exported to ERP** |
| `ANULAR` | -1 | - | Cancelled |

### 13.2 Business Meaning of MANP3, MANP4, MANP5

- **MANP3 (Min Lot)**: The required quantity does not reach the **minimum movement batch** of the product in that warehouse. The planner must decide whether to force the purchase anyway (assuming the minimum batch overcost) or look for alternatives (substitutes, centralization).

- **MANP4 (Min Purchase)**: The required quantity does not reach the **minimum purchase** required by the supplier for that location. The planner must decide whether to increase the quantity to the supplier's minimum or to centralize/group orders.

- **MANP5 (Min Shipping)**: The required quantity does not reach the **minimum shipping** (freight costs). Similar to MANP4 but related to transportation costs.

### 13.3 UI Visualization

The `ColorStatus` system assigns visual colors to each state:
- Gray: Unprocessed
- Green: OK
- Red: MANP3 (serious batch issue)
- Orange: MANP4 (minimum purchase issue)
- Yellow: MANP5 (shipping issue)
- Blue: Ready for export
- Purple: Exported

The hierarchical tree nodes (Plangroup → Supplier → Warehouse) display the color of the most severe status among their children.

---

## 14. Compensation Rules (RuleEquivalences)

### 14.1 Structure of a Rule

Defined in the `IN_RULE_EQUIVALENCES` table. Each rule specifies conditions to allow a source SKU to compensate a destination SKU.

| Field | Description | Values |
|---|---|---|
| `RuleID` | Unique rule identifier | String |
| `whouse_use` | Scope: 1 = same warehouse, 2 = other warehouses | 1/2 |
| `typeEquiv` | Equivalence type | I, D, S |
| `whouseOri` | Allowed source warehouse | Code or "ALL" |
| `whouseDest` | Allowed destination warehouse | Code or "ALL" |
| `aplicationType` | Application type | ALL, SECC, LINE, FAMI, SUBF, MARC, SEGM, FAMA, ITEM |
| `aplicationCode` | Specific application code | Section, line, family code, etc. |
| `class_ABC_gen` | General ABC classification | A, B, C |
| `class_ABC_SKU` | SKU ABC classification | A, B, C |
| `expensive` | Price restriction | S=Only expensive, N=Only cheap, ALL=All |
| `travel` | Can travel between warehouses | S=Yes, N=No, NONE=Unrestricted |
| `TotalStock` | Uses all stock or only overstock | True=TotalStock, False=OverStock |

### 14.2 Application Levels (Item_Aplication)

Ordered from least to most restrictive:

| Type | Meaning | Example |
|---|---|---|
| ALL | All products | No filter |
| SECC (1) | Section | All products in the "DAIRY" section |
| LINE (2) | Line | "YOGURTS" line |
| FAMI (3) | Family | "NATURAL YOGURT" family |
| SUBF (4) | SubFamily | "NATURAL SKIMMED YOGURT" subfamily |
| MARC (5) | Brand | "DANONE" brand |
| SEGM (6) | Segmentation | "BIO" segment |
| FAMA (7) | Family + Brand | Family "YOGURT" + Brand "DANONE" |
| ITEM (8) | Specific product | ItemID "123456" |

### 14.3 Rule Validation Algorithm

```
validateRule(proposal, whouseUse, candidate, SKU_Ori, SKU_Equi, itemLocation):
    1. Filter rules by whouse_use and typeEquiv
    2. For each candidate rule, check:
       a. whouseOri = SKU_Equi.location OR "ALL"
       b. whouseDest = SKU_Ori.location OR "ALL"
       c. Application matches (by type: section, line, family, etc.)
       d. ABC_gen matches
       e. ABC_SKU matches
       f. Expensive/Cheap matches
    3. From those that match, select the MOST RESTRICTIVE (highest getLevelRestriction)
    4. With the most restrictive rule:
       a. If same warehouse (whouseUse=1) → OK (Travel doesn't matter)
       b. If other warehouse (whouseUse=2) → Validate Travel
       c. If TotalStock → OK
       d. If OverStock → Verify OverStkQty_AvedemQty > 0 and Stock > MaxOutflow in leadTime
    5. Assign RuleID and stock type (Total/Over) to the candidate
```

### 14.4 Restriction Score (getLevelRestriction)

Each rule adds points for each filter it applies:
- `class_ABC_gen` specified: +1
- `class_ABC_SKU` specified: +2
- `aplicationCode` specified: +value of `aplicationType` (1 to 8)
- `whouseDest` specific: +4
- `whouseOri` specific: +4

The rule with the highest score is the most restrictive and is preferred.

---

## 15. Import Process from SO99

### 15.1 Import Flow

```
ImportBCC.import(planGroupList, incrementalImport, transferOrdersImport)
│
├── 1. Get Plangroups to import
│   └── Filtered by planning date (frequency)
│
├── 2. Backup current data
│   ├── Old proposals → BACK_IN_PROPOSALS1_DATE
│   └── Old logs → BACK_LOGS
│
├── 3. Cleanup obsolete data
│   ├── Processed proposals (STATUS = 'P')
│   ├── Old TransferOrders
│   └── Old logs (based on numDaysToClear_Log_Info)
│
├── 4. Import Masterdata
│   ├── IN_C_MASTERFILE ← SO99 C_MASTERFILE
│   ├── IN_C_ITEMLOCATION ← SO99 C_ITEMLOCATION
│   ├── IN_C_ITEMSUPPLIER ← SO99 C_ITEMSUPPLIER
│   ├── IN_C_SUPPLIERS ← SO99 C_SUPPLIERS
│   └── IN_C_WAREHOUSES ← SO99 C_WAREHOUSES
│
├── 5. Import support tables
│   ├── IN_ITEM_SUPERSSESIONS ← SO99
│   ├── IN_ITEM_SEGMENTATION ← SO99 (custom fields)
│   └── IN_PLANGROUPS ← SO99
│
├── 6. Import today's proposals
│   └── IN_T_PROPOSALS1_DATE ← SO99 PROPOSALS1_DATE
│       └── Filter: Active Plangroups, current date
│
└── 7. Import TransferOrders (if applicable)
    └── IN_TRANSFER_ORDERS ← SO99
```

---

## 16. Export Process to ERP

### 16.1 Export Flow

```
User → Mark for Export → Save → Export
│
├── 1. Mark proposals as TO_EXPORT
│   └── Traverse hierarchy: Plangroup → Supplier → Whouse → Proposal(Idx=0)
│
├── 2. Truncate OUT_* tables
│   ├── TRUNCATE OUT_PURCHASE_ORDERS
│   ├── TRUNCATE OUT_TRANSFER_ORDERS
│   └── TRUNCATE OUT_SUBSTITUTIONS
│
├── 3. Bulk Insert into OUT_PURCHASE_ORDERS
│   └── For each Plangroup with toExport=True:
│       └── Proposals with tratar=TO_EXPORT and QtyFinal > 0
│
├── 4. Bulk Insert into OUT_TRANSFER_ORDERS
│   └── Generated TransferOrders (PEM and SO99)
│
├── 5. Execute export SP
│   ├── Export_Procedure (moves OUT_* → INTER_SO99)
│   └── Proposals transition to EXPORTED state (95)
│
└── 6. Save state to database (saveProposalList)
```

### 16.2 Export Format

**OUT_PURCHASE_ORDERS:**
| Field | Description |
|---|---|
| ITEM | Product code |
| DESCRIPTION | Description |
| WHOUSE | Delivery warehouse |
| SUPPLIER | Supplier |
| IDX | Proposal index |
| YEAR_PROP, MONTH_PROP, DAY_PROP | Proposal date |
| YEAR_DELIVERY, MONTH_DELIVERY, DAY_DELIVERY | Delivery date |
| QTY | Quantity |
| UNIT | Unit |
| PROPOSAL_ID | Proposal ID |
| WHOUSE_FINAL | Final warehouse (for centralizations) |
| CRDATE | Record creation date |

---

## 17. User Journey – Planner Workflow

### 17.1 Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  1. START: The planner opens PEM                           │
│     • The app connects to SO99, PEM and INTER_SO99         │
│     • Loads configuration, language, columns               │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  2. IMPORT: The planner runs the import                    │
│     • Proposals are imported from SO99 for the Plangroups  │
│       whose planning frequency corresponds to the day      │
│     • Masterdata, stocks, substitutions are imported       │
│     • Optional: Load ERP Info (rules, logistics network)   │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  3. PEM AUTOMATICALLY CALCULATES:                          │
│     a) SO99 TransferOrders → linking + netting             │
│     b) Local Substitutions → searches substitutes in same  │
│        warehouse and nets                                  │
│     c) Network Substitutions → searches same/equivalents   │
│        across entire logistics network and nets            │
│     d) Minimums Calculation → evaluates MANP3, MANP4, MANP5│
│     e) Centralization → groups orders by batch/purchase    │
│     f) Promotions → applies discounts                      │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  4. VISUALIZATION: The planner sees the hierarchical tree  │
│     🌐 Plangroup                                           │
│       🏭 Supplier                                          │
│         🏬 Warehouse                                       │
│           📋 Proposals (grid with configured columns)      │
│                                                             │
│     Colors by status:                                      │
│     ⬜ Gray = Unprocessed                                  │
│     🟢 Green = OK                                          │
│     🔴 Red = MANP3 (minimum batch issue)                   │
│     🟠 Orange = MANP4 (minimum purchase issue)             │
│     🟡 Yellow = MANP5 (shipping issue)                     │
│     🔵 Blue = Ready for export                             │
│     🟣 Purple = Exported                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  5. PROBLEM REVIEW (MANP3, MANP4, MANP5):                  │
│     The planner reviews nodes with issues:                 │
│                                                             │
│     For each necessity in MANP3/MANP4/MANP5:               │
│     • Opens the restrictions form                          │
│       (Restriction_NecessitiesForm)                        │
│     • Sees: Product, Quantity, Required minimum,           │
│       Action options                                       │
│                                                             │
│     Available actions:                                     │
│     • ADVANCE (0): Advance quantity                        │
│     • BUY (1): Force purchase (ignore minimum)             │
│     • NO_BUY (2): Do not buy                               │
│     • TRANSFER (3): Transfer from another warehouse        │
│     • CEN_NO_BUY (4): Centralize and do not buy            │
│                                                             │
│     The planner can also:                                  │
│     • Edit quantities manually in the grid                 │
│     • View substitution candidates (SubsProposalsForm)     │
│     • View movements/transfers (MovimentsPorposalsForm)    │
│     • Edit compensation rules (RuleCompensaDialog)         │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  6. RECALCULATE (optional):                                │
│     After manual changes, the planner can:                 │
│     • Re-run substitution calculations                     │
│     • Re-run minimum calculation                           │
│     • States update automatically                          │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  7. APPROVAL AND EXPORT:                                   │
│     • The planner selects Plangroups to export             │
│     • Marks as "To Export" → TO_EXPORT state (90)          │
│     • Runs Export:                                         │
│       - Inserts into OUT_PURCHASE_ORDERS                   │
│       - Inserts into OUT_TRANSFER_ORDERS                   │
│       - Executes SP that moves data to INTER_SO99          │
│     • Proposals transition to EXPORTED state (95)          │
│     • Data reaches the ERP to generate actual orders       │
└─────────────────────────────────────────────────────────────┘
```

### 17.2 Planner Workflow (Summary)

1. **Open PEM** → Database connection
2. **Import** → Load today's proposals from SO99
3. **Calculate** → PEM runs all automatic calculations
4. **Visualize** → Tree with color indicators
5. **Review MANP3/MANP4/MANP5** → Resolve issues manually
6. **Recalculate** if necessary
7. **Approve** → Mark for export
8. **Export** → Data sent to ERP

---

## 18. Logging System

### 18.1 Logging Architecture

```
LogBCC (Business Control)
├── Text Mode (initial, before DB connection)
│   └── Files in /Log/
└── Database Mode (after connection)
    ├── In-memory queue (ConcurrentQueue)
    ├── Periodic dump timer
    └── LOGS table in PEM database
```

### 18.2 Log Types

| Type | Description | When Used |
|---|---|---|
| `User` | User-visible actions | Normal operations |
| `UserDetail` | User action detail | Operation debugging |
| `Debug` | Debug information | Development only |
| `Error` | Errors | Caught exceptions |

### 18.3 Action Types (LogTypeActionEnum)

| Action | Description |
|---|---|
| `Import` | Import processes |
| `Export` | Export processes |
| `Load_data` | Data loading |
| `Load_UI` | Interface loading |
| `Load_Server` | Server mode loading |
| `Compensa_Local` | Local compensation |
| `Compensa_Network` | Network compensation |
| `Min_Lot` | Minimum batch calculation |
| `Min_Compra` | Minimum purchase calculation |
| `Min_Portes` | Minimum shipping calculation |
| `ValidateRule` | Rule validation |
| `AddCandidate` | Add candidate |
| `redLog` | Logistics network operations |
| `Trans_Order` | Transfer orders |
| `Centr` | Centralization |
| `Promo` | Promotions |

---

## 19. Configuration Parameters (ParamGen)

Managed by `ParamGenBCC` and persisted in `CONFIG_Paramgen`.

| Parameter | Type | Description | Typical Value |
|---|---|---|---|
| `Language` | String | App language (ES/EN) | ES |
| `multiThread` | Boolean | Use parallel processing (Parallel.ForEach) | true |
| `HorizontVisualizationTimeDays` | Integer | Visualization horizon days (0 = all) | 90 |
| `UnitCostFlag` | Boolean | true=PurchasePrice, false=UnitCost | false (0) |
| `ImportDay` | Date | Import date | Today |
| `skipImport` | Boolean | Skip import | false |
| `Log_User` | Boolean | Enable user logging | true |
| `Log_UserDetail` | Boolean | Enable detailed user logging | false |
| `Log_Debug` | Boolean | Enable debug logging | false |
| `Log_num_Queue` | Integer | Log queue size | 100 |
| `Adv_ABC_Restriction` | String | ABC restriction for advances | "" |

---

## 20. UI Structure and Forms

### 20.1 Main Forms

| Form | File | Purpose |
|---|---|---|
| **MainFormNEW** | `UI/MainFormNEW.vb` | Main screen with hierarchical tree, grids, toolbars |
| **MainFormNEW_AUX** | `UI/MainFormNEW_AUX.vb` | Auxiliary code for MainForm (handlers, workers) |
| **ConsoleForm** | `UI/ConsoleForm.vb` | Console/server mode |
| **LogsForm** | `UI/LogsForm.vb` | Log visualization |
| **configSQLConnexionForm** | `UI/configSQLConnexionForm.vb` | SQL connection configuration |

### 20.2 Proposal Forms

| Form | Purpose |
|---|---|
| **ProposalsForm** | Detailed proposal grid |
| **SubsProposalsForm** | Substitution candidates grid |
| **MovimentsPorposalsForm** | Movements/transfers grid |

### 20.3 Dialogs

| Dialog | Purpose |
|---|---|
| **RuleCompensaDialog** | Compensation rule editing |
| **promoDialog** | Promotion management |
| **statusDIalog** | Status visualization/editing |

### 20.4 Summary Nodes (SummaryNodes)

| Component | Purpose |
|---|---|
| **Plangroup_Node_Info** | Plangroup summary view |
| **Supplier_Node_Info** | Supplier summary view |
| **Whouse_Node_Info** | Warehouse summary view |
| **NodeUtils** | Node utilities |

### 20.5 Restrictions Panel

| Form | Purpose |
|---|---|
| **Restriction_NecessitiesForm** | Panel where the planner reviews and resolves problem necessities (MANP3/4/5) |

---

## Appendix A: Key Enums and Constants

### Equivalent_Type
- `S` (1) = Substitute (Sustituto) (SuperSession)
- `I` (2) = Same (Igual)
- `D` (3) = Different (Equivalente)

### Item_Aplication
- `ALL` (0), `SE` (1), `LINE` (2), `FAMI` (3), `SUBF` (4), `MARC` (5), `SEGM` (6), `FAMA` (7), `ITEM` (8)

### Plangroup_TypeAccum
- `ACCUM` (1) = Centralizes, does not separate orders
- `NO_ACCUM` (2) = Does not centralize
- `ALL_WHOUSE` (3) = Delivers to all warehouses

### Plangroup_LevelCenter
- `REG` (1) = Centralizes via Regional
- `CEN` (2) = Centralizes directly to Central

### Restriction_Action
- `ADVANCE` (0) = Advance
- `BUY` (1) = Buy
- `NO_BUY` (2) = Do not buy
- `TRANSFER` (3) = Transfer
- `CEN_NO_BUY` (4) = Centralize without buying

---

## Appendix B: Glossary

| Term | Meaning |
|---|---|
| **SO99** | Demand planning system (source of proposals) |
| **Plangroup** | Planning group that groups products with the same review frequency |
| **Proposal** | Purchase suggestion generated by SO99 for a product in a warehouse |
| **Netting (Netificación)** | Process of adjusting a proposal's quantity by subtracting substitute stock |
| **Substitute (Sustituto) (S)** | Discontinued product that replaces the proposal's product |
| **Same (Igual) (I)** | The same product in another warehouse |
| **Equivalent (Equivalente) (D)** | Different product but functionally interchangeable |
| **MANP3** | Necessity that does not meet the minimum movement batch |
| **MANP4** | Necessity that does not meet the supplier's minimum purchase |
| **MANP5** | Necessity that does not meet the minimum shipping |
| **Centralization (Centralización)** | Grouping orders at an upper-level warehouse to meet minimums |
| **Logistics Network (Red Logística)** | Warehouse hierarchy: Central → Regional → Local |
| **TransferOrder** | Stock transfer order between warehouses |
| **OverStock** | Stock that exceeds projected demand (available for exchange) |
| **ScopeDistribution** | Scope of how far a product can be distributed in the network |
| **SKUL** | Compound key: SKU + Location |