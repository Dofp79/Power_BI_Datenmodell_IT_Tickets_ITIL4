# Power BI – "Geschlossen am" simulieren

In vielen ITSM-Datenquellen fehlt ein explizites Feld **„Geschlossen am“**. Mit Power Query (M) und DAX kannst du dieses Feld ableiten. Je nach Datenlage gibt es drei Varianten.

---

## Variante A – Mit Status-Log (Empfohlen)

### Tabellen
- **Tickets**: Stammdaten (enthält `TicketID`, `Erstellt am`, …)
- **StatusLog**: Verlaufsdaten (`TicketID`, `Status`, `StatusDatum`)

### Ziel
- Erstes Datum, an dem ein Ticket den Status **„Gelöst“** oder **„Geschlossen“** erreicht → `Geschlossen am`
- Merge zurück in die Faktentabelle

### 1) Status-Text normalisieren
```m
let
  Source = StatusLog,
  Types  = Table.TransformColumnTypes(Source,{{"TicketID", type text},{"Status", type text},{"StatusDatum", type datetime}}),
  Trim   = Table.TransformColumns(Types,{{"Status", each Text.Upper(Text.Trim(_))}}),
  Map    = Table.ReplaceValue(Trim, "IN PROGRESS", "IN BEARBEITUNG", Replacer.ReplaceText, {"Status"})
in
  Map
```

### 2) „Geschlossen am“ ermitteln
```m
let
  Clean       = #"StatusLog Normalisiert",
  ClosedSet   = Table.SelectRows(Clean, each [Status] = "GELÖST" or [Status] = "GESCHLOSSEN"),
  MinClosed   = Table.Group(ClosedSet, {"TicketID"}, {{"Geschlossen am", each List.Min([StatusDatum]), type datetime}})
in
  MinClosed
```

### 3) Mit Tickets mergen
```m
let
  Tix        = Tickets,
  Merge      = Table.NestedJoin(Tix, {"TicketID"}, MinClosed, {"TicketID"}, "Closed", JoinKind.LeftOuter),
  Expanded   = Table.ExpandTableColumn(Merge, "Closed", {"Geschlossen am"}, {"Geschlossen am"})
in
  Expanded
```

> Ergebnis: `fact_Incidents` enthält nun ein Feld **„Geschlossen am“** (nullable).

---

## Variante B – Ohne Log (Momentaufnahme)

Falls es nur ein Feld **„Letzte Änderung am“** gibt und der aktuelle Status **„Geschlossen“** ist:

```m
let
  Tix       = Tickets,
  Types     = Table.TransformColumnTypes(Tix,{{"Erstellt am", type datetime},{"Letzte Änderung am", type datetime},{"Status", type text}}),
  Norm      = Table.TransformColumns(Types,{{"Status", each Text.Upper(Text.Trim(_))}}),
  AddClosed = Table.AddColumn(Norm, "Geschlossen am", each if [Status] = "GESCHLOSSEN" or [Status] = "GELÖST" then [Letzte Änderung am] else null, type datetime)
in
  AddClosed
```

> ⚠️ Hinweis: Dies ist nur eine **Näherung** (setzt voraus, dass die letzte Änderung = Schließdatum ist).

---

## Variante C – Mit Erledigt-Flag

Falls es ein Bool-Feld `Erledigt` und ein Datum `Erledigt am` gibt:

```m
let
  AddClosed = Table.AddColumn(Tickets, "Geschlossen am", each if [Erledigt] = true then [Erledigt am] else null, type datetime)
in
  AddClosed
```

---

## DAX mit dem neuen Feld

Sobald `Geschlossen am` existiert, kannst du es in Measures nutzen:

```DAX
Bearbeitungszeit (Tage) :=
AVERAGEX (
  'fact_Incidents',
  DATEDIFF('fact_Incidents'[Erstellt am], COALESCE('fact_Incidents'[Geschlossen am], TODAY()), DAY)
)

Tickets nach Schließdatum :=
CALCULATE ( [Anzahl_Tickets], USERELATIONSHIP('fact_Incidents'[Geschlossen am], 'dim_Datum'[Date]) )
```

---

## Praxis-Tipps

- Lege `Geschlossen am` **nur** für Tickets mit Status „Gelöst/Geschlossen“; sonst `null`.
- Nutze in Visuals:
  - **Eröffnungs-Trends** über `Erstellt am` (aktive Beziehung)
  - **Abschluss-Trends** über `Geschlossen am` via `USERELATIONSHIP`
- Dokumentiere im Modell: *„Geschlossen am (simuliert aus Status-Log / letzter Änderung)“*
