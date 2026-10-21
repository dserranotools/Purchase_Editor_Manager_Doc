# PEM - Documentation (Unified Version)

> **Project:** PurchaseEditorManager (PEM)  
> **Version:** V 800.200514 Beta

---

[← INDEX](../index.md)

---

## Available document

| Document | Audience | Contents |
|---|---|---|
| [PEM_Specs_Complete.md](./PEM_Specs_Complete.md) | 👨‍💻👤📊 **Everyone** | Technical specifications, business logic, and user journey in a single file (~50KB, 20 sections) |

---

## What's included?

The document [PEM_Specs_Complete.md](./PEM_Specs_Complete.md) covers:

### 📐 Technical Specifications (for Devs)
- Layered architecture (UI → BCC → DAL → BEC)
- Complete class model (all BEC, DAL, BCC entities)
- Database schema (33 SQL Server tables)
- Data flow across the 3 databases (SO99, PEM, ERP)
- Configuration system (.conf files, ParamGen)
- Logging system
- UI structure and forms

### 📊 Business Specifications (Business Logic)
- Purchase proposal processing
- Quantity netting
- Substitution (S), equivalence (D), and equality (I) system
- Logistics networks and warehouse levels (N0–N3)
- Batch-based (ACCUM) and purchase-based centralization
- MANP3, MANP4, MANP5 statuses and their meaning
- Compensation rules (RuleEquivalences)
- Rule validation algorithm

### 👤 User Journey (Planner Flow)
1. Startup and connection
2. Import from SO99
3. Automatic calculation (PEM)
4. Hierarchical view with color indicators
5. Issue review (MANP3/MANP4/MANP5)
6. Manual resolution (ADVANCE, BUY, NO_BUY, TRANSFER, CEN_NO_BUY)
7. Approval and export to ERP

---

## Prefer multiple documents?

→ [../multi-doc/README_en_multi-doc.md](../multi-doc/README_en_multi-doc.md) — 3 separate documents by audience (Devs, Business, Planners)

---

## Project structure

```
Trunk/
├── PurchaseEditor/           # VB.NET source code
│   ├── BCC/                  # Business Control Classes
│   │   └── Equivalences/     # Compensation rules
│   ├── BEC/                  # Business Entity Classes
│   │   └── Equivalences/
│   ├── DAL/                  # Data Access Layer
│   │   └── Equivalences/
│   ├── UI/                   # WinForms
│   │   ├── Proposals/
│   │   ├── Dialogs/
│   │   ├── SummaryNodes/
│   │   └── Debug_Aux/
│   ├── Global/               # AppConf, ParamGen
│   │   └── LOG/
│   ├── ColorsPEM/            # Status colors
│   ├── UTIL/                 # StringBuilderDB, Utils
│   ├── Config/               # .conf files
│   │   └── Languages/        # ES/EN
│   └── Images/               # Ribbon icons
├── SQL/                      # SQL Server scripts
│   ├── CreateDB_Script_NEW.sql
│   ├── Export_Procedure.sql
│   └── ...
├── docs/                     # Documentation
│   ├── README.md
│   ├── one-doc/              # Unified version (this folder)
│   │   ├── README.md
│   │   └── PEM_Specs_Complete.md
│   └── multi-doc/            # Audience-separated version
│       ├── README.md
│       ├── 01_Technical_Specs.md
│       ├── 02_Business_Logic.md
│       └── 03_User_Journey.md
└── PurchaseEditor.sln
```

---

## Quick technical notes

- **Language:** VB.NET, .NET Framework 4.8
- **UI:** DevExpress WinForms (Office 2016 Colorful)
- **Database:** 3 SQL Server instances (SO99, PEM, ERP/INTER)
- **Authentication:** SQL Server or Windows/Active Directory
- **Languages:** Spanish (ES) and English (EN)
- **Default language:** Spanish
- **Configuration files:** External to the executable, in the `Config/` folder