Status-Log (Empfehlung)

Tabellen

Tickets (aktuelle Stammdaten, enthält TicketID, Erstellt am, …)

StatusLog (Verlaufsdaten: TicketID, Status, StatusDatum)

Ziel

Erstes Datum, an dem der Status „Gelöst/Geschlossen“ erreicht wurde - Geschlossen am

Merge zurück in die Faktentabelle

1) Status-Text normalisieren (StatusLog)
```m
let
  Source = StatusLog,
  Types  = Table.TransformColumnTypes(Source,{{"TicketID", type text},{"Status", type text},{"StatusDatum", type datetime}}),
  Trim   = Table.TransformColumns(Types,{{"Status", each Text.Upper(Text.Trim(_))}}),
  Map    = Table.ReplaceValue(Trim, "IN PROGRESS", "IN BEARBEITUNG", Replacer.ReplaceText, {"Status"})
in
  Map

3) „Geschlossen am“ aus dem Log ermitteln
let
  Clean       = #"StatusLog Normalisiert",
  ClosedSet   = Table.SelectRows(Clean, each [Status] = "GELÖST" or [Status] = "GESCHLOSSEN"),
  MinClosed   = Table.Group(ClosedSet, {"TicketID"}, {{"Geschlossen am", each List.Min([StatusDatum]), type datetime}})
in
  MinClosed

4) Mit Tickets mergen → Fact
let
  Tix        = Tickets,
  Merge      = Table.NestedJoin(Tix, {"TicketID"}, MinClosed, {"TicketID"}, "Closed", JoinKind.LeftOuter),
  Expanded   = Table.ExpandTableColumn(Merge, "Closed", {"Geschlossen am"}, {"Geschlossen am"})
in
  Expanded
```

Ergebnis: fact_Incidents enthält nun Geschlossen am (nullable).

Variante B – Kein Log, nur Momentaufnahme

Wenn es z. B. Letzte Änderung am gibt und Status aktuell „Geschlossen“ ist:
```m

let
  Tix       = Tickets,
  Types     = Table.TransformColumnTypes(Tix,{{"Erstellt am", type datetime},{"Letzte Änderung am", type datetime},{"Status", type text}}),
  Norm      = Table.TransformColumns(Types,{{"Status", each Text.Upper(Text.Trim(_))}}),
  AddClosed = Table.AddColumn(Norm, "Geschlossen am", each if [Status] = "GESCHLOSSEN" or [Status] = "GELÖST" then [Letzte Änderung am] else null, type datetime)
in
  AddClosed
```

Hinweis: Das ist nur eine Näherung (nimmt an, dass die letzte Änderung das Schließdatum ist).

Variante C – „Erledigt-Flag“

Hast du ein Bool/Flag Erledigt und ein Feld Erledigt am? Dann einfach kopieren:
```m
let
  AddClosed = Table.AddColumn(Tickets, "Geschlossen am", each if [Erledigt] = true then [Erledigt am] else null, type datetime)
in
  AddClosed

DAX (mit neuem Feld nutzbar)
Bearbeitungszeit (Tage) :=
AVERAGEX (
  'fact_Incidents',
  DATEDIFF('fact_Incidents'[Erstellt am], COALESCE('fact_Incidents'[Geschlossen am], TODAY()), DAY)
)

Tickets nach Schließdatum :=
CALCULATE ( [Anzahl_Tickets], USERELATIONSHIP('fact_Incidents'[Geschlossen am], 'dim_Datum'[Date]) )
```
Praxis-Tipps

Lege Geschlossen am nur für Tickets mit Status „Gelöst/Geschlossen“; sonst null.

Nutze in Visuals:
– Eröffnungs-Trends über Erstellt am (aktive Beziehung)
– Abschluss-Trends über Geschlossen am via USERELATIONSHIP.

Dokumentiere im Modell: „Geschlossen am (simuliert aus Status-Log / letzter Änderung)“.
