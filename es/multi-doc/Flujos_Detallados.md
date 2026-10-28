# Flujos Detallados de PEM

> **Documentación complementaria** — Explicación paso a paso de los flujos de negocio más complejos de PEM.  
> Este documento amplía los conceptos descritos en las especificaciones generales con ejemplos concretos, fragmentos de código y casos borde.

---

## Estado del documento

| # | Flujo | Estado |
|---|---|---|
| 1 | [TransferOrders SO99 → Netificación de propuestas](#sec1) | ✅ Completado |
| 2 | [Búsqueda de candidatos: Sustitutos (S), Iguales (I), Equivalentes (D)](#sec2) | ⏳ Pendiente |
| 3 | [Validación de candidatos contra reglas de compensación](#sec3) | ⏳ Pendiente |
| 4 | [Cálculo de MANP3/4/5 (mínimos de lote, compra, portes)](#sec4) | ⏳ Pendiente |
| 5 | [Centralización ACCUM → TransferOrders](#sec5) | ⏳ Pendiente |
| 6 | [Exportación: BulkInsert + Stored Procedures](#sec6) | ⏳ Pendiente |
| 7 | [Importación desde SO99 (backup, clean, load)](#sec7) | ⏳ Pendiente |

---

<a id="sec1"></a>
## 1. TransferOrders SO99 → Netificación de propuestas

### Contexto

SO99 genera dos tipos de datos que PEM importa:
- **Propuestas de compra**: sugerencias de compra de producto X al proveedor Y para el almacén Z
- **TransferOrders (traspasos)**: órdenes de movimiento de stock entre almacenes, del tipo "sobra producto en VAL4, lleva 50 uds a MAD1"

Ambos se cargan en estructuras separadas dentro de `ProposalsBCC`:

```vb
' Carga de propuestas de compra → mProposalPlangroupList
Me.mProposalPlangroupList = ProposalsDAL.getProposalList(whereSQL, mItemBCC)

' Carga de TransferOrders → mTransferSO99Plangroup
Me.mTransferSO99Plangroup = ProposalsDAL.getTransferOrdersList(whereSQL, mItemBCC)
```

### El problema

Cuando SO99 genera una TransferOrder, también ajusta la propuesta de compra original para reflejar que parte de la necesidad se cubre con el traspaso. Sin embargo, PEM necesita **recuperar esa cantidad** para sus propios cálculos de sustituciones y mínimos, ya que trabaja sobre las propuestas de compra completas y luego aplica sus propias reglas.

### Ejemplo concreto

```
Escenario:
  - MAD1 necesita 100 uds de Leche A para la semana que viene
  - VAL4 tiene excedente de 50 uds de Leche A
  
SO99 genera:
  - Propuesta de compra: "Comprar 100 uds Leche A para MAD1"
  - TransferOrder:       "Traspasar 50 uds Leche A de VAL4 → MAD1"
```

### Paso a paso en PEM

#### Paso 1: Vincular TransferOrder con propuesta de compra

El método `netificaProposals_TransferordersSO99` recorre todas las TransferOrders importadas y busca su propuesta de compra asociada mediante `getSRC_Proposal()`:

```vb
Private Sub netificaProposals_TransferordersSO99(ByRef transferSO99Whouse As BEC.ProposalWhouseEntity)
    For Each transferSO99 As BEC.ProposalEntity In transferSO99Whouse.idxProposalList(0).Values
        
        ' Busca la propuesta de compra a la que está vinculada esta TransferOrder
        proposal_SRC = mProposalPlangroupList.getSRC_Proposal(transferSO99)
        
        If proposal_SRC IsNot Nothing Then
            ' ...
        End If
    Next
End Sub
```

La vinculación se hace a través del campo `ProposalLink` de la TransferOrder, que contiene un identificador que permite localizar la propuesta de compra destino en `mProposalPlangroupList`.

#### Paso 2: Netificar (incrementar la propuesta de compra)

```vb
' Aumenta la QProp de la propuesta destino con la cantidad de la TransferOrder
proposal_SRC.QProp += transferSO99.saveQtySust
```

**¿Por qué incrementar y no decrementar?** Porque PEM necesita que `QProp` refleje la **necesidad completa original** (100 uds) para poder aplicar correctamente sus propios cálculos de sustitución y compensación. Si la cantidad ya estuviera reducida por el traspaso, los cálculos de sustituciones se harían sobre 50 uds en lugar de 100, perdiendo oportunidades de optimización.

Además, se almacena información de trazabilidad en la propuesta destino:

```vb
proposal_SRC.TransferLink = transferSO99.SKUL          ' Qué TransferOrder la cubre
proposal_SRC.SRC_Whouse = transferSO99.WhouseID         ' De qué almacén viene
proposal_SRC.SRC_LocationID = transferSO99.LocationID
proposal_SRC.Virtstk_SRC = transferSO99.StockActual     ' Cuánto stock hay en origen
```

Si la TransferOrder cubre completamente la necesidad (`QtyFinal = 0`), la propuesta se marca como `TRANS_Netifica` (77):

```vb
If proposal_SRC.QtyFinal = 0 Then
    proposal_SRC.tratar = EnumsBCC.Necessity_Status.TRANS_Netifica
End If
```

#### Paso 3: Sustituir la TransferOrder (si aplica)

Si la propuesta destino tiene sustitutos (calculados en fases posteriores), la TransferOrder original se **divide** para reflejar los productos sustitutos:

```vb
' En createTransferLink_CEN_or_SUST:
If proposal_SRC.substituteList.Count > 0 Then
    Me.transferLink_SUST(proposal_SRC, transferSO99, transferSO99List_New)
End If
```

El método `transferLink_SUST` crea nuevas TransferOrders con los productos sustitutos:

```vb
' Para cada sustituto de la propuesta destino que coincide con el producto original
For Each sust As BEC.SubstituteProposal In proposal_SRC.substituteList.Values
    If sust.ItemID_Ori <> sust.ItemID_Dest Then
        ' Crea una nueva TransferOrder con el producto sustituto
        transferSO99_New = New BEC.ProposalEntity(sust.ItemID_Ori, ...)
        transferSO99_New.SRC_LocationID = sust.Location_Orig
        transferSO99_New.tratar = EnumsBCC.Necessity_Status.TRANSFER_Link
        
        ' Traspasa la cantidad de la TransferOrder original a la nueva
        If transferSO99.QtyFinal >= sust.Qty Then
            transferSO99.QtyFinal -= sust.Qty
            transferSO99_New.QtyFinal = sust.Qty
            sust.Qty = 0
        End If
        
        transferSO99List_New.Add(transferSO99_New)
    End If
Next
```

#### Ejemplo visual completo

```
TransferOrder SO99: 50 uds Leche A   VAL4 → MAD1
                     ↓
        1. Vincular: propuestaMAD1.TransferLink = "<SKUL>"
                      propuestaMAD1.SRC_Whouse = "VAL4"
                     ↓
        2. Netificar: propuestaMAD1.QProp += 50 (ahora QProp = 100+50 = 150?)
         [NOTA: realmente QProp recupera la cantidad original completa]
                     ↓
  (Más adelante: se ejecutan sustituciones locales/red)
                     ↓
        3. Si propuestaMAD1 tiene sustitutos (ej: Leche B 20 uds en VAL4):
                     ↓
           TransferOrder 1: 30 uds Leche A   VAL4 → MAD1  (original, reducida)
           TransferOrder 2: 20 uds Leche B   VAL4 → MAD1  (nueva, del sustituto)
```

### Casos borde

**¿Qué pasa si el `ItemLocation` de origen no existe?**
```vb
itemLocation_src = Me.mItemBCC.getItemLocation(transferSO99.ItemID, transferSO99.SRC_LocationID)
If itemLocation_src Is Nothing Then
    transferSO99.Virtstk_SRC = Single.MinValue.ToString  ' Marca como inválido
End If
```

**¿Qué pasa si la propuesta destino ya tenía otra TransferOrder vinculada?**
```vb
If String.IsNullOrWhiteSpace(proposal_SRC.TransferLink) Then
    ' Primera TransferOrder
    proposal_SRC.TransferLink = transferSO99.SKUL
Else
    ' Ya tenía una, se concatenan
    proposal_SRC.TransferLink += ", " & transferSO99.SKUL
    proposal_SRC.Virtstk_SRC += ", " & transferSO99.StockActual.ToString
    proposal_SRC.SRC_LocationID += ", " & transferSO99.LocationID
    proposal_SRC.SRC_Whouse += ", " & transferSO99.WhouseID
End If
```

**¿Qué pasa con las TransferOrders que se centralizan?**
Si la propuesta destino se centraliza (`proposal_SRC.CentralizeTo` no vacío), la TransferOrder se re-vincula al almacén central en lugar del destino original (método `transferLink_CEN`).

---

<a id="sec2"></a>
## 2. Búsqueda de candidatos: Sustitutos (S), Iguales (I), Equivalentes (D)

> ⏳ **Pendiente de documentar**

Este flujo explicará:
- Cómo PEM busca, para cada propuesta con `QtySust > 0`, candidatos en el mismo almacén (Local) y en toda la red (Network)
- Diferencias entre: Sustitutos (S) de `IN_ITEM_SUPERSSESIONS`, Iguales (I) del mismo `ItemID` en otras ubicaciones, Equivalentes (D) de `IN_ITEM_EQUIVALENCES`
- Orden de búsqueda: S → I → D, Local → Network
- Criterios de pre-selección: stock > 0, minResLot, stock de implantación

---

<a id="sec3"></a>
## 3. Validación de candidatos contra reglas de compensación

> ⏳ **Pendiente de documentar**

Este flujo explicará:
- Cómo `validateRule` evalúa cada candidato contra todas las reglas aplicables
- El sistema de puntuación (`getLevelRestriction`) para seleccionar la regla más restrictiva
- Diferencia entre TotalStock y OverStock
- Verificación de proyección de stock vs MaxOutflow

---

<a id="sec4"></a>
## 4. Cálculo de MANP3/4/5 (mínimos de lote, compra, portes)

> ⏳ **Pendiente de documentar**

Este flujo explicará:
- Cómo `calcMinimums` evalúa cada propuesta contra las restricciones del proveedor
- Lote mínimo (MANP3) desde `IN_SUPP_ITEM_MOV`
- Compra mínima (MANP4) desde `IN_SUPP_LOCATION_COMPRA`
- Portes mínimos (MANP5) desde `IN_SUPP_LOCATION_PORTES`
- Cómo se generan las necesidades (`WRK_NECESSITIES`) con sus estados

---

<a id="sec5"></a>
## 5. Centralización ACCUM → TransferOrders

> ⏳ **Pendiente de documentar**

Este flujo explicará:
- Diferencia entre proveedores ACCUM, NO_ACCUM y ALL_WHOUSE
- Niveles de centralización REG y CEN
- Cómo se genera la CentralizationEntity
- Cómo se crean las TransferOrders del almacén central a los destinos

---

<a id="sec6"></a>
## 6. Exportación: BulkInsert + Stored Procedures

> ⏳ **Pendiente de documentar**

Este flujo explicará:
- Cómo se marcan las propuestas como TO_EXPORT
- El truncado de tablas OUT_
- El BulkInsert con `SqlBulkCopy`
- La ejecución de Stored Procedures que mueven datos a INTER_SO99
- Cómo se maneja el mapeo de nombres de tabla via `CONFIG_Tables_SO99`

---

<a id="sec7"></a>
## 7. Importación desde SO99 (backup, clean, load)

> ⏳ **Pendiente de documentar**

Este flujo explicará:
- Cómo se determinan los Plangroups a importar según frecuencia de planificación
- El proceso de backup en tablas BACK_
- La limpieza de datos obsoletos
- La importación de masterdata y propuestas
- La importación de TransferOrders

---

> **Documento en construcción** — Los flujos marcados como ⏳ Pendiente se irán completando progresivamente.
