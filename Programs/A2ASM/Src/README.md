# Aufgabe A2 – Speicher und Byte-Reihenfolge

## Grundprinzip: Little Endian

Der Prozessor speichert ein 16-Bit-Halfword im Little-Endian-Format: Das niederwertige Byte liegt an der kleineren Adresse. Daher wird `0xBEEF` zunächst als Bytes `EF BE` abgelegt. Ein Debugger, der die Bytes in aufsteigender Adresse zeigt, zeigt also `EF BE` – das ist nicht dasselbe wie die Registerdarstellung `0xBEEF`.

## Anw01 bis Anw06

Ausgangszustand von `VariableA` (`DCW 0xBEEF`):

- `[VariableA]` = `EF`
- `[VariableA + 1]` = `BE`
- `R0`, `R2` und `R3` sind vor den jeweiligen Zuweisungen nicht festgelegt; die folgenden Werte beschreiben die Effekte der Anweisungen.

| Zeile | Effekt auf Register | Effekt auf Memory ab `VariableA` |
|---|---|---|
| **Anw01** `ldr R0,=VariableA` | `R0` erhält die Adresse von `VariableA`. `R2` und `R3` bleiben unverändert. | Keine Änderung: Bytes `EF BE` (aufsteigende Adressen). |
| **Anw02** `ldrb R2,[R0]` | Lädt das Byte an `VariableA` in `R2`: `R2 = 0x000000EF`. `R0` und `R3` bleiben unverändert. | Keine Änderung: `EF BE`. |
| **Anw03** `ldrb R3,[R0,#1]` | Lädt das Byte an `VariableA + 1`: `R3 = 0x000000BE`. `R0` und `R2` bleiben unverändert. | Keine Änderung: `EF BE`. |
| **Anw04** `lsl R2,#8` | Verschiebt `R2` um 8 Bit nach links: `R2 = 0x0000EF00`. `R0` und `R3` bleiben unverändert. | Keine Änderung: `EF BE`. |
| **Anw05** `orr R2,R3` | Bitweises ODER: `R2 = 0x0000EF00 OR 0x000000BE = 0x0000EFBE`. `R0` und `R3` bleiben unverändert. | Keine Änderung: `EF BE`. |
| **Anw06** `strh R2,[R0]` | Schreibt die unteren 16 Bit von `R2` (`0xEFBE`) in den Speicher. Die Registerwerte werden nicht verändert. | Little Endian speichert `0xEFBE` als Bytes `BE EF`. Im Byte-Fenster erscheint damit `BE EF`, also die gewünschte sichtbare Reihenfolge „BEEF“. |

### Warum unterscheidet sich die Darstellung?

`0xBEEF` als 16-Bit-Zahlenwert wird im Little-Endian-Speicher als `EF BE` abgelegt. Die ersten sechs Anweisungen lesen diese beiden Bytes einzeln, bauen im Register den Wert `0xEFBE` auf und schreiben ihn zurück. Dadurch stehen anschließend die Bytes `BE EF` im Speicher.

## `VariableC`: „affe“ im Memory sichtbar machen

`ConstByteA EQU 0xAFFE` lädt den Zahlenwert `0xAFFE` in `R5`. Ein direktes `strh R5,[...]` würde im Little-Endian-Speicher die Bytes `FE AF` ablegen. Damit im Memory-Fenster `AF FE` (also „AFFE“ als Bytefolge) erscheint, müssen die Bytes vor dem Speichern vertauscht werden:

```asm
VariableC   DCW 0x0000

    mov     R5,#ConstByteA
    ldr     R0,=VariableC
    rev16   R5,R5
    strh    R5,[R0]
```

`REV16` vertauscht die beiden Bytes innerhalb des Halfwords: `R5` wird von `0x0000AFFE` zu `0x0000FEAF`. `strh` speichert diesen Wert Little Endian als Bytes `AF FE`. Das Memory-Fenster zeigt daher `AF FE`.

## Anw09 ff.: `12 34` im Memory-Fenster

`VariableB DCW 0x1234` liegt zunächst als Bytes `34 12` im Little-Endian-Speicher. Um in aufsteigender Adresse `12` gefolgt von `34` zu erhalten, werden die Bytes nach dem Laden vertauscht und der Wert wieder als Halfword gespeichert:

```asm
    ldr     R1,=VariableB
    ldrh    R6,[R1]
    rev16   R6,R6
    strh    R6,[R1]
```

Nach `ldrh` ist `R6 = 0x00001234`. Nach `rev16` ist `R6 = 0x00003412`. Das Speichern von `0x3412` in Little Endian ergibt die Bytes `12 34` an `VariableB` und `VariableB + 1`.

## Erwartete Ergebnisse

| Variable | Wert im Register vor dem Speichern | Bytes an aufsteigenden Adressen |
|---|---:|---|
| `VariableA` nach Anw06 | `R2 = 0xEFBE` | `BE EF` |
| `VariableC` | `R5 = 0xFEAF` nach `REV16` | `AF FE` |
| `VariableB` | `R6 = 0x3412` nach `REV16` | `12 34` |

**Debugger-Hinweis:** Die konkrete Anzeige hängt davon ab, ob das Memory-Fenster Bytes, Halfwords oder Words gruppiert und in welcher Spaltenreihenfolge es sie darstellt. Für diese Aufgabe die einzelnen Bytes an `VariableA`, `VariableB` bzw. `VariableC` und jeweils an Adresse `+1` prüfen.
