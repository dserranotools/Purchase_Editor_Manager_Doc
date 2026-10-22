# PEM - Documentation (Multi-Document Version)

> **Project:** PurchaseEditorManager (PEM)  
> **Version:** V 800.200514 Beta

---

[← INDEX](../index.md)

---

## Available documents

| # | Document | Audience | Content |
|---|---|---|---|
| 1 | [01_Technical_Specs.md](./01_Technical_Specs.md) | 👨‍💻 **Developers** | Architecture, class model, DB schema, DAL/BCC/BEC layers, logging, configuration, multithreading |
| 2 | [02_Business_Logic.md](./02_Business_Logic.md) | 📊 **Business Analysts** | Netting, substitutions (S/I/D), compensation rules, centralization, MANP3/4/5, import/export |
| 3 | [03_User_Journey.md](./03_User_Journey.md) | 👤 **Planners / Product** | Step-by-step planner flow, typical scenarios, troubleshooting, FAQs, tips |

---

## Which document should I read first?

```
Are you a developer and need to understand the code?
  → Start with 01_Technical_Specs.md

Are you a business analyst and need to understand the business rules?
  → Start with 02_Business_Logic.md

Are you a planner/buyer and need to learn how to use the tool?
  → Start with 03_User_Journey.md

Are you new to the project and need an overview?
  → Read 03_User_Journey.md first (what it does)
  → Then 02_Business_Logic.md (how it does it)
  → Finally 01_Technical_Specs.md (how it is built)
```

---

## Prefer a single document?

→ [../one-doc/README_en_one-doc.md](../one-doc/README_en_one-doc.md) — Everything in a single file (~50KB, 20 sections)

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
│   ├── UI/                   # WinForms forms
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
│   ├── one-doc/              # Unified version
│   │   ├── README.md
│   │   └── PEM_Specs_Complete.md
│   └── multi-doc/            # Multi-document version (this folder)
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
- **Configuration files:** External to the executable, in `Config/` folder