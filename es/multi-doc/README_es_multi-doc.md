# PEM - Documentación (Versión Multi-Documento)

> **Proyecto:** PurchaseEditorManager (PEM)  
> **Versión:** V 800.200514 Beta

---

[← ÍNDICE](../index.md)

---

## Documentos disponibles

| # | Documento | Audiencia | Contenido |
|---|---|---|---|
| 1 | [01_Technical_Specs.md](./01_Technical_Specs.md) | 👨‍💻 **Developers** | Arquitectura, modelo de clases, DB schema, capas DAL/BCC/BEC, logging, configuración, multithreading |
| 2 | [02_Business_Logic.md](./02_Business_Logic.md) | 📊 **Business Analysts** | Netificación, sustituciones (S/I/D), reglas de compensación, centralización, MANP3/4/5, import/export |
| 3 | [03_User_Journey.md](./03_User_Journey.md) | 👤 **Planificadores / Product** | Flujo paso a paso del planificador, escenarios típicos, resolución de problemas, FAQs, consejos |

---

## ¿Qué documento leer primero?

```
¿Eres developer y necesitas entender el código?
  → Empieza por 01_Technical_Specs.md

¿Eres business analyst y necesitas entender las reglas de negocio?
  → Empieza por 02_Business_Logic.md

¿Eres planificador/comprador y necesitas aprender a usar la herramienta?
  → Empieza por 03_User_Journey.md

¿Eres nuevo en el proyecto y necesitas una visión general?
  → Lee primero 03_User_Journey.md (qué hace)
  → Luego 02_Business_Logic.md (cómo lo hace)
  → Finalmente 01_Technical_Specs.md (cómo está construido)
```

---

## ¿Prefieres un solo documento?

→ [../one-doc/README_es_one-doc.md](../one-doc/README_es_one-doc.md) — Todo en un solo archivo (~50KB, 20 secciones)

---

## Estructura del proyecto

```
Trunk/
├── PurchaseEditor/           # Código fuente VB.NET
│   ├── BCC/                  # Business Control Classes
│   │   └── Equivalences/     # Reglas de compensación
│   ├── BEC/                  # Business Entity Classes
│   │   └── Equivalences/
│   ├── DAL/                  # Data Access Layer
│   │   └── Equivalences/
│   ├── UI/                   # Formularios WinForms
│   │   ├── Proposals/
│   │   ├── Dialogs/
│   │   ├── SummaryNodes/
│   │   └── Debug_Aux/
│   ├── Global/               # AppConf, ParamGen
│   │   └── LOG/
│   ├── ColorsPEM/            # Colores por estado
│   ├── UTIL/                 # StringBuilderDB, Utils
│   ├── Config/               # .conf files
│   │   └── Languages/        # ES/EN
│   └── Images/               # Iconos Ribbon
├── SQL/                      # Scripts SQL Server
│   ├── CreateDB_Script_NEW.sql
│   ├── Export_Procedure.sql
│   └── ...
├── docs/                     # Documentación
│   ├── README.md
│   ├── one-doc/              # Versión unificada
│   │   ├── README.md
│   │   └── PEM_Specs_Complete.md
│   └── multi-doc/            # Versión multi-documento (esta carpeta)
│       ├── README.md
│       ├── 01_Technical_Specs.md
│       ├── 02_Business_Logic.md
│       └── 03_User_Journey.md
└── PurchaseEditor.sln
```

---

## Documentación complementaria

| Documento | Contenido |
|---|---|
| 📖 [Flujos Detallados](Flujos_Detallados.md) | Explicación paso a paso de los flujos de negocio más complejos, con ejemplos concretos y fragmentos de código |

---

## Notas técnicas rápidas

- **Lenguaje:** VB.NET, .NET Framework 4.8
- **UI:** DevExpress WinForms (Office 2016 Colorful)
- **Base de datos:** 3 instancias SQL Server (SO99, PEM, ERP/INTER)
- **Autenticación:** SQL Server o Windows/Active Directory
- **Idiomas:** Español (ES) e Inglés (EN)
- **Idioma por defecto:** Español
- **Archivos de configuración:** Fuera del ejecutable, en carpeta `Config/`