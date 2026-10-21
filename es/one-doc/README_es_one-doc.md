# PEM - Documentación (Versión Unificada)

> **Proyecto:** PurchaseEditorManager (PEM)  
> **Versión:** V 800.200514 Beta

---

[← ÍNDICE](../index.md)

---

## Documento disponible

| Documento | Audiencia | Contenido |
|---|---|---|
| [PEM_Specs_Complete.md](./PEM_Specs_Complete.md) | 👨‍💻👤📊 **Todos** | Especificaciones técnicas, lógica de negocio y user journey en un solo archivo (~50KB, 20 secciones) |

---

## ¿Qué contiene?

El documento [PEM_Specs_Complete.md](./PEM_Specs_Complete.md) cubre:

### 📐 Especificaciones Técnicas (para Devs)
- Arquitectura en capas (UI → BCC → DAL → BEC)
- Modelo de clases completo (todas las entidades BEC, DAL, BCC)
- Esquema de base de datos (33 tablas SQL Server)
- Flujo de datos entre las 3 bases de datos (SO99, PEM, ERP)
- Sistema de configuración (.conf files, ParamGen)
- Sistema de logging
- Estructura de UI y formularios

### 📊 Especificaciones de Negocio (Business Logic)
- Procesamiento de propuestas de compra
- Netificación de cantidades
- Sistema de sustitución (S), equivalencia (D) e igualdad (I)
- Redes logísticas y niveles de almacén (N0-N3)
- Centralización por lotes (ACCUM) y por compra
- Estados MANP3, MANP4, MANP5 y su significado
- Reglas de compensación (RuleEquivalences)
- Algoritmo de validación de reglas

### 👤 User Journey (Flujo del Planificador)
1. Inicio y conexión
2. Importación desde SO99
3. Cálculo automático (PEM)
4. Visualización jerárquica con indicadores de color
5. Revisión de problemas (MANP3/MANP4/MANP5)
6. Resolución manual (ADVANCE, BUY, NO_BUY, TRANSFER, CEN_NO_BUY)
7. Aprobación y exportación al ERP

---

## ¿Prefieres varios documentos?

→ [../multi-doc/README_es_multi-doc.md](../multi-doc/README_es_multi-doc.md) — 3 documentos separados por audiencia (Devs, Business, Planificadores)

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
│   ├── one-doc/              # Versión unificada (esta carpeta)
│   │   ├── README.md
│   │   └── PEM_Specs_Complete.md
│   └── multi-doc/            # Versión separada por audiencias
│       ├── README.md
│       ├── 01_Technical_Specs.md
│       ├── 02_Business_Logic.md
│       └── 03_User_Journey.md
└── PurchaseEditor.sln
```

---

## Notas técnicas rápidas

- **Lenguaje:** VB.NET, .NET Framework 4.8
- **UI:** DevExpress WinForms (Office 2016 Colorful)
- **Base de datos:** 3 instancias SQL Server (SO99, PEM, ERP/INTER)
- **Autenticación:** SQL Server o Windows/Active Directory
- **Idiomas:** Español (ES) e Inglés (EN)
- **Idioma por defecto:** Español
- **Archivos de configuración:** Fuera del ejecutable, en carpeta `Config/`