| [← README](README_en_multi-doc.md) | [Technical Specs](01_Technical_Specs.md) | [Business Logic](02_Business_Logic.md) | 📄 User Journey |
|:---:|:---:|:---:|:---:|

&nbsp;

# PEM - User Journey
## Planner Flow

> **Project:** PurchaseEditorManager (PEM)  
> **Audience:** Planners, Purchasing Managers, Trainers, Product Owners  
> **Version:** V 800.200514 Beta

---

## Index

1. [Planner's Day-to-Day Overview](#1-planners-day-to-day-overview)
2. [Step 1: Starting PEM](#2-step-1-starting-pem)
3. [Step 2: Importing from SO99](#3-step-2-importing-from-so99)
4. [Step 3: Automatic Calculation](#4-step-3-automatic-calculation)
5. [Step 4: Viewing Results](#5-step-4-viewing-results)
6. [Step 5: Reviewing and Resolving Issues](#6-step-5-reviewing-and-resolving-issues)
7. [Step 6: Manual Operations](#7-step-6-manual-operations)
8. [Step 7: Approval and Export to ERP](#8-step-7-approval-and-export-to-erp)
9. [Typical Scenarios](#9-typical-scenarios)
10. [Shortcuts and Tips](#10-shortcuts-and-tips)
11. [Frequently Asked Questions](#11-frequently-asked-questions)

---

## 1. Planner's Day-to-Day Overview

```
┌──────────────────────────────────────────────────────┐
│                 THE PLANNER'S DAY                    │
├──────────────────────────────────────────────────────┤
│  08:00  Open PEM                                     │
│  08:05  Import proposals from SO99                   │
│  08:10  Load ERP Info (if there are changes)         │
│  08:15  Run automatic calculations                   │
│  08:20  Review the tree → See colors                 │
│  08:30  Resolve MANP3 (red)                          │
│  09:00  Resolve MANP4 (orange)                       │
│  09:30  Resolve MANP5 (yellow)                       │
│  10:00  Review OK proposals (green)                  │
│  10:15  Recalculate if there were manual changes     │
│  10:30  Mark for export                              │
│  10:35  Export to ERP                                │
│  10:40  Review Logs → Confirm export OK              │
└──────────────────────────────────────────────────────┘
```

> 💡 **Note:** Times are estimates. An experienced planner with few Plangroups can complete the cycle in 30-45 minutes.

---

## 2. Step 1: Starting PEM

### What the planner sees

1. **Launches the application** from the shortcut
2. The application loads configuration and tests the connection to the 3 databases
3. **If the connection fails**, a configuration window appears:
   - Allows changing server, database, user, and password
   - Supports Windows authentication (empty user/password fields)
4. Once connected, the **main screen** opens

### Main Screen (MainFormNEW)

```
┌─────────────────────────────────────────────────────────────┐
│ [Toolbar: Import | Calculate | Export | Config | Logs]      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  🌐 Hierarchical Tree              │ 📋 Proposal Grid       │
│  ┌─────────────────────────┐   │ ┌─────────────────────────┐│
│  │ 🟢 DAIRY_DAILY          │   │ │Item │RefFabr│Qty│Status ││
│  │  └─ 🟢 DANONE           │   │ │1234 │DAN001 │50 │OK     ││
│  │     └─ 🟢 VAL4          │   │ │5678 │DAN002 │30 │MANP3  ││
│  │     └─ 🔴 ALI1          │   │ │9012 │DAN003 │20 │MANP4  ││
│  │  └─ 🟠 NESTLÉ           │   │ │...                      ││
│  │     └─ 🔵 MAD1          │   │ │                         ││
│  │ 🔴 FROZEN_WEEKLY        │   │ │                         ││
│  │  └─ ...                 │   │ │                         ││
│  └─────────────────────────┘   │ └─────────────────────────┘│
│                                                             │
│ [Restrictions / Substitutions / Movements Panel]            │
└─────────────────────────────────────────────────────────────┘
```

**Tree color coding:**

| Color | Status | Meaning for the planner |
|---|---|---|
| ⬜ **Gray** | Unprocessed | Has not been calculated yet |
| 🟢 **Green** | OK | No issues. Ready to export |
| 🔴 **Red** | MANP3 | Warning! Minimum batch issue |
| 🟠 **Orange** | MANP4 | Warning! Minimum purchase issue |
| 🟡 **Yellow** | MANP5 | Warning: shipping cost issue |
| 🔵 **Blue** | To Export | Marked for export |
| 🟣 **Purple** | Exported | Already sent to ERP |

The color of each node reflects the **most severe status** of its children.

---

## 3. Step 2: Importing from SO99

### When to import

- At the start of each planning day
- When SO99 has generated new proposals
- The import frequency depends on the Plangroups (daily, weekly...)

### How to import

1. Press the **"Import"** button on the toolbar
2. PEM automatically:
   - Determines which Plangroups need to be planned today
   - Imports proposals from SO99 for those Plangroups
   - Imports updated master data (products, stocks, suppliers)
   - Backs up previous data
3. A progress bar shows the progress
4. When finished, a confirmation message with:
   - Number of proposals imported
   - Number of records cleaned up
   - Total time

### Optional: Load ERP Info

**"Load ERP Info"** button — Only needed if there have been changes to:
- Compensation rules
- Logistics network
- Supplier restrictions
- Product equivalences

---

## 4. Step 3: Automatic Calculation

### What PEM does automatically

The planner presses **"Calculate"** and PEM executes in sequence:

```
✅ 1. TransferOrders SO99
   └── Links transfers planned by SO99 with purchase proposals
   └── Discounts transferred quantities

✅ 2. Local Substitutions
   └── Searches within the same warehouse for products that can substitute / be equivalent
   └── Applies compensation rules
   └── Nets out (discounts stock found)

✅ 3. Network Substitutions
   └── Searches the ENTIRE logistics network: same product in other warehouses
   └── Searches for equivalents in other warehouses
   └── Generates TransferOrders for the necessary transfers

✅ 4. Minimums Calculation
   └── Evaluates minimum batch → marks MANP3 if not met
   └── Evaluates minimum purchase → marks MANP4 if not met
   └── Evaluates minimum shipping → marks MANP5 if not met

✅ 5. Centralization
   └── Groups orders from ACCUM suppliers
   └── Generates TransferOrders from the hub to the destinations

✅ 6. Promotions
   └── Applies discounts and rebates
```

### Multithreading Mode

If `multiThreading=true` (default), PEM uses parallel processing:
- Multiple Plangroups are calculated simultaneously
- Within each Plangroup, multiple Warehouses in parallel
- **Faster** on multi-core machines

---

## 5. Step 4: Viewing Results

### Tree Navigation

```
🌐 Plangroup "DAIRY_DAILY" 🟢
│   → Expand to see suppliers
│
├── 🏭 DANONE 🟢
│   │  → Expand to see warehouses
│   │
│   ├── 🏬 VAL4 (Valencia) 🟢
│   │   └── Click to see proposals in the grid
│   │
│   ├── 🏬 ALI1 (Alicante) 🔴
│   │   └── Warning! Has proposals with MANP3
│   │
│   └── 🏬 CAS1 (Castellón) 🟠
│       └── Warning! Has proposals with MANP4
│
└── 🏭 NESTLÉ 🟢
    ├── 🏬 MAD1 (Madrid) 🔵
    └── 🏬 SEV1 (Sevilla) 🟣
```

### The Proposal Grid

When clicking on a warehouse, the grid shows all its proposals with the configured columns:

| ItemID | RefFabr | Description | QProp | QtyFinal | Status |
|---|---|---|---|---|---|
| 123456 | REF001 | Whole milk 1L | 100 | 100 | 🟢 OK |
| 789012 | REF002 | Semi-skimmed milk 1L | 50 | 15 | 🔴 MANP3 |
| 345678 | REF003 | Skimmed milk 1L | 30 | 30 | 🟠 MANP4 |

### Visible columns

The displayed columns depend on the view type and are configured in `columns.conf`:
- General View: item, reference, description, brand, family, dates, quantities, costs
- SO99 View: adds centralization and transfer columns
- Substitutions View: columns for candidates, stock, rules
- Centralization View: grouping columns

---

## 6. Step 5: Reviewing and Resolving Issues

### 6.1 Identifying Issues

The planner navigates the tree looking for nodes with alert colors:
- 🔴 **Red** = MANP3 → Critical batch issues
- 🟠 **Orange** = MANP4 → Minimum purchase issues
- 🟡 **Yellow** = MANP5 → Shipping cost issues

### 6.2 Restrictions Panel (Restriction_NecessitiesForm) (Restricciones)

Double-clicking a node with issues opens the restrictions panel:

```
┌─────────────────────────────────────────────────────────────┐
│ RESTRICTIONS - ALI1 (Alicante) - DANONE                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Product: 789012 - Semi-skimmed milk 1L                     │
│  Needed quantity: 15 units                                  │
│  Minimum batch: 50 units                                    │
│  Current status: MANP3                                      │
│                                                             │
│  What action to take?                                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ( ) ADVANCE  - Push forward to next period           │   │
│  │ ( ) BUY       - Force purchase of 50 units           │   │
│  │ ( ) NO_BUY    - Do not buy                           │   │
│  │ ( ) TRANSFER  - Transfer from another warehouse      │   │
│  │ ( ) CEN_NO_BUY- Centralize without buying            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  [Apply]  [Cancel]                                          │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 Resolution Strategy

**For MANP3 (minimum batch):**

| Situation | Recommended action |
|---|---|
| Critical product, stockout unacceptable | **BUY** — absorb the full batch cost |
| Non-critical product, stock available elsewhere | **TRANSFER** — search the logistics network |
| Low-turnover product, can wait | **ADVANCE** — accumulate for the next period |
| Discontinued product or no real demand | **NO_BUY** — do not buy |

**For MANP4 (minimum purchase):**

| Situation | Recommended action |
|---|---|
| Sole supplier, no alternative | **BUY** — accept the minimum |
| Can be grouped with other warehouses | **CEN_NO_BUY** — centralize to reach the minimum |

**For MANP5 (shipping):**
| Situation | Recommended action |
|---|---|
| Shipping cost is acceptable | **BUY** — pay shipping |
| Can be grouped with other orders | **CEN_NO_BUY** — share shipping costs |

### 6.4 View Substitution Candidates

**"View Substitutions"** button → Opens `SubsProposalsForm`:

```
┌─────────────────────────────────────────────────────────────┐
│ SUBSTITUTION CANDIDATES - Proposal 789012                   │
├─────────────────────────────────────────────────────────────┤
│ ItemOrig│LocOrig│ItemDest│Stock │Type│Rule│Qty to transfer  │
│ 789012  │VAL4   │789012  │60 u  │I   │R01 │15 u             │
│ 890123  │MAD1   │789012  │40 u  │D   │R02 │0 u              │
└─────────────────────────────────────────────────────────────┘
```

- Shows which products/warehouses can offset the proposal
- Indicates the rule that allows it
- Shows the quantity to be transferred

### 6.5 View Movements

**"View Movements"** button → Opens `MovimentsPorposalsForm`:
- Generated TransferOrders
- Created centralizations
- Origin, destination, quantity, date

---

## 7. Step 6: Manual Operations

### Edit Quantities

In the proposal grid, the planner can manually edit:
- `QtyFinal`: final quantity to buy
- `ReleaseDate`: order release date
- `DueDate`: delivery date

Edited cells are marked and the proposal status is automatically updated.

### Edit Compensation Rules

**"Rules"** button → Opens `RuleCompensaDialog`:
- List of all equivalence rules
- Create, edit, delete rules
- Fields: equivalence type, warehouses, application, ABC, expensive/cheap, travel, stock

### Manage Plangroups

**"Plangroups"** menu → Opens `PlangroupsForm`:
- View/edit planning frequency
- Change centralization type (ACCUM/NO_ACCUM)
- Change centralization level (REG/CEN)

---

## 8. Step 7: Approval and Export to ERP

### 8.1 Mark for Export

1. Select one or more Plangroups in the tree
2. **"Mark for Export"** button
3. The Plangroup's proposals change to **TO_EXPORT** (🔵 blue) status
4. Nodes change to blue color

### 8.2 Export

1. **"Export"** button
2. PEM executes:
   - Clears the output tables
   - Inserts purchase orders into `OUT_PURCHASE_ORDERS`
   - Inserts transfers into `OUT_TRANSFER_ORDERS`
   - Executes the stored procedure for ERP export
3. Proposals change to **EXPORTED** (🟣 purple) status
4. Confirmation message with:
   - Number of purchase orders exported
   - Number of TransferOrders exported
   - Number of substitutions applied

### 8.3 Verify Export

- Check the **Log** ("Logs" button) to see the export history
- Verify in the ERP that the orders arrived correctly
- Cross-check the number of records generated vs. those exported

---

## 9. Typical Scenarios

### Scenario 1: Normal day, no issues

```
1. Import → 120 proposals
2. Calculate → All green ✅
3. Mark for export
4. Export → 120 orders sent to ERP
⏱️ Total time: ~5 minutes
```

### Scenario 2: Several MANP3s in critical products

```
1. Import → 150 proposals
2. Calculate → 12 in MANP3 🔴 (top products)
3. Review each MANP3:
   • 8 → BUY (force purchase, they are critical)
   • 3 → TRANSFER (stock available in other warehouses)
   • 1 → ADVANCE (low turnover, can wait)
4. Recalculate → All green except the ADVANCE
5. Mark for export (except ADVANCE)
6. Export → 149 orders
⏱️ Total time: ~20-30 minutes
```

### Scenario 3: Issues with a supplier that does not split orders (ACCUM)

```
1. Import → 80 proposals from supplier DANONE
2. Calculate → Several MANP4 🟠 (don't reach the minimum purchase)
3. The planner reviews:
   • DANONE is ACCUM → centralize at regional level
   • All DANONE proposals are grouped at REG_ESTE
   • PEM generates TransferOrders: REG_ESTE → VAL4, ALI1, CAS1
4. Total quantity at REG_ESTE: 250 u → Exceeds the 200 minimum!
5. Mark for export
6. Export
⏱️ Total time: ~15 minutes
```

### Scenario 4: Loading new rules from ERP

```
1. The purchasing team has defined new compensation rules
2. Planner presses "Load ERP Info"
3. Rules, equivalences, logistics network are updated
4. Recalculate → New rules enable more substitutions
5. Some MANP3/4/5 are automatically resolved
⏱️ Total time: ~5-10 minutes
```

---

## 10. Shortcuts and Tips

### Productivity Tips

| Tip | Explanation |
|---|---|
| **Prioritize reds** | MANP3s are usually the most critical. Resolve them first |
| **Use the ABC filter** | If an ABC restriction is configured, C products can wait |
| **Recalculate after changes** | If you edit quantities or resolve restrictions, recalculate to see the impact |
| **Check the rules** | If an expected candidate does not appear, review the compensation rules |
| **Check the Log** | If something fails, the Log shows exactly what happened and where |
| **Centralize when you can** | Grouping orders often resolves multiple MANP4/MANP5 at once |

### Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| F5 | Refresh/Recalculate |
| Ctrl+E | Export |
| Ctrl+I | Import |
| Ctrl+L | View Logs |

---

## 11. Frequently Asked Questions

### ❓ Why is a proposal still in MANP3 even though I already searched for substitutes?

Possible causes:
- There are no substitutes/equivalents defined for that product
- The compensation rules do not allow the operation (review rules)
- The available stock does not reach the minimum transfer batch
- The product does not "travel" (travel restriction = NO)

### ❓ What is the difference between an Equal candidate (I) and an Equivalent candidate (D)?

- **Equal (I)**: It is EXACTLY the same product, only the warehouse changes. E.g.: DANONE Milk 1L at VAL4 → DANONE Milk 1L at MAD1
- **Equivalent (D)**: It is a DIFFERENT but interchangeable product. E.g.: DANONE Milk 1L → PASCUAL Milk 1L

### ❓ Can I export only some Plangroups and leave others for later?

Yes. Mark for export only the Plangroups you want to send to the ERP. The rest remain pending.

### ❓ What happens if I export the same Plangroup twice?

The `OUT_*` tables are cleared (TRUNCATE) before each export, so only the current state is exported. Already-exported proposals (EXPORTED status) are not re-sent.

### ❓ Can I undo an export?

Not directly from PEM. Contact the ERP team to cancel the generated orders. In PEM, the proposals remain in EXPORTED (95) status.

### ❓ How often should I import from SO99?

It depends on your Plangroups' planning frequency. The norm is once a day for daily Plangroups, and on the corresponding day for weekly/biweekly ones.

### ❓ The calculation is very slow, what can I do?

- Verify that `multiThreading=true` in `settings.conf`
- Reduce the visualization horizon (`HorizontVisualizationTimeDays`)
- Check that the database connections are fast
- If you use a remote SQL server, network latency has a significant impact

---

## Visual Flow Summary

```
START
  │
  ├─ [1] OPEN PEM ─────────────────────────────────────── 1 min
  │      • Connection to SO99, PEM, ERP
  │
  ├─ [2] IMPORT ───────────────────────────────────────── 2-5 min
  │      • Load the day's proposals from SO99
  │      • (Optional) Load ERP Info
  │
  ├─ [3] CALCULATE ───────────────────────────────────── 1-5 min
  │      • TransferOrders, Substitutions, Minimums, Centralization
  │
  ├─ [4] REVIEW ───────────────────────────────────────── 5-30 min
  │      • Color tree: identify reds/oranges
  │      • Restrictions panel: resolve MANP3/4/5
  │      • Edit quantities if needed
  │
  ├─ [5] RECALCULATE (optional) ───────────────────────── 1-3 min
  │      • Reflect manual changes
  │
  ├─ [6] APPROVE ──────────────────────────────────────── 1 min
  │      • Mark Plangroups for export
  │
  └─ [7] EXPORT ───────────────────────────────────────── 1-2 min
         • Send orders to ERP
         • Confirm in Logs
```