# Power BI Datenmodell – IT Tickets (ITIL 4)

Dieses Paket liefert ein praxistaugliches Datenmodell inkl. DAX-Messgrößen und einer Hierarchie für **Kategorie → Hardware → Software**. Es ist so gestaltet, dass du mit den vorhandenen Spalten sofort starten kannst und trotzdem sauber zu einem sternförmigen Modell (Star Schema) kommst.

---

## 1) Zielbild: Star Schema

**Faktentabelle**

* `Fact_Tickets`

  * Schlüssel: `TicketID` (falls nicht vorhanden, in Power Query generieren)
  * Attribute (aus deiner Liste):

    * `Störung` (Typ)
    * `Umgebung`
    * `Erstellt am`
    * `aufgenommen von`
    * `Anwender`
    * `Anw-Dst`, `Anw-Dst2`, `Anw-Dst3`
    * `Kategorie`
    * `Betreff`
    * `Hardware`
    * `Software`
    * `akt. Sachgeb.`
    * `aktueller SB`
    * `Status`
    * `Reaktionsebene`
    * **Optional empfohlen**: `Geschlossen am` (falls nicht vorhanden, später ersetzbar durch „heute“ für offene Tickets)

**Dimensionen**

* `Dim_Date` (Kalender)
* `Dim_Org` (Organisation)
* `Dim_User` (Anwender; optional, wenn PII minimiert werden soll, nur Hash/ID)
* `Dim_Agent` (Mitarbeitende/Sachbearbearbeitung)
* `Dim_ServiceArea` (Sachgebiet/Team)
* `Dim_Environment` (Umgebung)
* `Dim_Status` (Statuswerte)
* `Dim_ReactionLevel` (Reaktionsebene; 1st/2nd/3rd Level)
* `Dim_Classification` (**Hierarchie**: Kategorie → Hardware → Software)

**Kernbeziehungen**

* `Fact_Tickets[Erstellt am]` → `Dim_Date[Date]` (Viele-zu-Eins, Richtung: beidseitig **aus**)
* `Fact_Tickets[Geschlossen am]` (optional über zweites Datumsfeld mit **Rolle** „Geschlossenes Datum“, siehe Measures)
* `Fact_Tickets[Anw-Dst3]` → `Dim_Org[Org_L3]` (plus L2/L1 als Spalten in Dim\_Org)
* `Fact_Tickets[aktueller SB]` → `Dim_Agent[AgentName]`
* `Fact_Tickets[akt. Sachgeb.]` → `Dim_ServiceArea[ServiceArea]`
* `Fact_Tickets[Umgebung]` → `Dim_Environment[Environment]`
* `Fact_Tickets[Status]` → `Dim_Status[Status]`
* `Fact_Tickets[Reaktionsebene]` → `Dim_ReactionLevel[Level]`
* **Wichtig**: `Fact_Tickets[ClassificationKey]` → `Dim_Classification[ClassificationKey]` (siehe Abschnitt 2)

---

## 2) Klassifikations-Hierarchie (Kategorie → Hardware → Software)

### 2.1 Power Query (Staging) – Schlüssel erzeugen

Erzeuge in **Power Query** eine staging-Query `stg_Tickets` (aus Excel/SharePoint/CSV/SQL), bereinige Spalten (Trim/Clean) und konstruiere einen stabilen Klassifikationsschlüssel.

```m
// in Power Query: Klassifikations-Key bauen
let
    Source = stg_Tickets,
    ToText = Table.TransformColumnTypes(Source,{
        {"Kategorie", type text}, {"Hardware", type text}, {"Software", type text}
    }),
    Trimmed = Table.TransformColumns(ToText,{
        {"Kategorie", each Text.Upper(Text.Trim(_))},
        {"Hardware", each Text.Upper(Text.Trim(_))},
        {"Software", each Text.Upper(Text.Trim(_))}
    }),
    Filled = Table.ReplaceValue(Trimmed, null, "(UNBEKANNT)", Replacer.ReplaceValue,{"Kategorie","Hardware","Software"}),
    ClassificationKey = Table.AddColumn(Filled, "ClassificationKey", each
        Text.Combine({[Kategorie],[Hardware],[Software]}, "||"), type text)
in
    ClassificationKey
```

### 2.2 Dimension `Dim_Classification`

Erzeuge die Dimension als eindeutige Liste:

```m
let
    Source = stg_Tickets,
    ToText = Table.TransformColumnTypes(Source,{
        {"Kategorie", type text}, {"Hardware", type text}, {"Software", type text}
    }),
    Trimmed = Table.TransformColumns(ToText,{
        {"Kategorie", each Text.Upper(Text.Trim(_))},
        {"Hardware", each Text.Upper(Text.Trim(_))},
        {"Software", each Text.Upper(Text.Trim(_))}
    }),
    Filled = Table.ReplaceValue(Trimmed, null, "(UNBEKANNT)", Replacer.ReplaceValue,{"Kategorie","Hardware","Software"}),
    AddKey = Table.AddColumn(Filled, "ClassificationKey", each Text.Combine({[Kategorie],[Hardware],[Software]}, "||"), type text),
    Distinct = Table.Distinct(Table.SelectColumns(AddKey,{"ClassificationKey","Kategorie","Hardware","Software"}))
in
    Distinct
```

In Power BI Desktop richte im Modell eine **Hierarchie** auf `Dim_Classification` ein:

* **Classification**

  * Stufe 1: `Kategorie`
  * Stufe 2: `Hardware`
  * Stufe 3: `Software`

`Fact_Tickets[ClassificationKey]` verknüpft sich 1\:n mit `Dim_Classification[ClassificationKey]`.

---

## 3) Kalender & Rollen

### 3.1 `Dim_Date` (DAX)

```DAX
Dim_Date =
ADDCOLUMNS(
    CALENDAR(date(2018,1,1), date(2035,12,31)),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "MMM"),
    "MonthNo", MONTH([Date]),
    "YearMonth", FORMAT([Date], "YYYY-MM"),
    "Week", WEEKNUM([Date],2),
    "Quarter", QUARTER([Date])
)
```

> Tipp: Lege für „Erstellt am“ die Standardbeziehung an. Für „Geschlossen am“ verwende **Measures** mit `USERELATIONSHIP` (siehe 4.3), so brauchst du keine zweite aktive Beziehung.

---

## 4) DAX Measures (Kern-KPIs)

> **Namenskonvention**: Präfixe wie `#` für Zähler, `∆` für Zeiten sind optional – hier einfache, klare Namen.

### 4.1 Basiszähler

```DAX
Tickets Gesamt := COUNTROWS('Fact_Tickets')

Tickets Offen :=
CALCULATE(
    [Tickets Gesamt],
    KEEPFILTERS('Fact_Tickets'[Status] IN {"OFFEN","IN BEARBEITUNG"})
)

Tickets Geschlossen :=
CALCULATE(
    [Tickets Gesamt],
    KEEPFILTERS('Fact_Tickets'[Status] IN {"GELÖST","GESCHLOSSEN"})
)

Tickets 1st Level :=
CALCULATE(
    [Tickets Gesamt],
    KEEPFILTERS('Fact_Tickets'[Reaktionsebene] = "1ST")
)

First Time Resolution % :=
DIVIDE(
    CALCULATE([Tickets Gesamt], KEEPFILTERS('Fact_Tickets'[Reaktionsebene] = "1ST"), KEEPFILTERS('Fact_Tickets'[Status] IN {"GELÖST","GESCHLOSSEN"})),
    [Tickets Geschlossen]
)
```

> Passe Status-/Level-Codes an deine echten Werte an (Groß/Kleinschreibung egal, wenn du vorher in PQ normalisierst).

### 4.2 Durchlauf- und Alterszeiten

```DAX
// Für geschlossene Tickets: Geschlossen am - Erstellt am
Bearbeitungszeit (Tage) :=
VAR EndDate = COALESCE( MAX('Fact_Tickets'[Geschlossen am]), TODAY() )
VAR StartDate = MAX('Fact_Tickets'[Erstellt am])
RETURN
DATEDIFF(StartDate, EndDate, DAY)

Ø Bearbeitungszeit (alle) :=
AVERAGEX(
    'Fact_Tickets',
    VAR EndDate = COALESCE('Fact_Tickets'[Geschlossen am], TODAY())
    VAR StartDate = 'Fact_Tickets'[Erstellt am]
    RETURN DATEDIFF(StartDate, EndDate, DAY)
)

Ø Bearbeitungszeit (geschl.) :=
CALCULATE(
    [Ø Bearbeitungszeit (alle)],
    KEEPFILTERS('Fact_Tickets'[Status] IN {"GELÖST","GESCHLOSSEN"})
)

Backlog-Alter (Ø Tage, offen) :=
CALCULATE(
    AVERAGEX(
        'Fact_Tickets',
        DATEDIFF('Fact_Tickets'[Erstellt am], TODAY(), DAY)
    ),
    KEEPFILTERS('Fact_Tickets'[Status] IN {"OFFEN","IN BEARBEITUNG"})
)
```

### 4.3 Zeitbezug über „Geschlossen am“ (USERELATIONSHIP)

Wenn du ein zweites Datumsfeld `Geschlossen am` hast, kannst du Trends nach **Schließdatum** berechnen, ohne die aktive Beziehung umzuschalten:

```DAX
Tickets nach Schließdatum :=
CALCULATE(
    [Tickets Gesamt],
    USERELATIONSHIP('Fact_Tickets'[Geschlossen am], 'Dim_Date'[Date])
)

Ø Bearbeitungszeit nach Schließmonat :=
CALCULATE(
    [Ø Bearbeitungszeit (geschl.)],
    USERELATIONSHIP('Fact_Tickets'[Geschlossen am], 'Dim_Date'[Date])
)
```

### 4.4 Service- & Team-Performance

```DAX
Tickets je Reaktionsebene % :=
VAR total = [Tickets Gesamt]
RETURN
DIVIDE( [Tickets 1st Level], total )

Workload aktueller SB (offen) :=
CALCULATE(
    [Tickets Offen],
    ALLEXCEPT('Fact_Tickets','Fact_Tickets'[aktueller SB])
)

SLA Erfüllung % (Beispiel) :=
// Annahme: SLA = 5 Arbeitstage für 1st Level; passe Formel an deine SLA-Regeln an
VAR LimitDays = 5
VAR ClosedInSLA =
    CALCULATE(
        COUNTROWS('Fact_Tickets'),
        KEEPFILTERS('Fact_Tickets'[Status] IN {"GELÖST","GESCHLOSSEN"}),
        FILTER('Fact_Tickets',
            DATEDIFF('Fact_Tickets'[Erstellt am], COALESCE('Fact_Tickets'[Geschlossen am], TODAY()), DAY) <= LimitDays
        )
    )
RETURN DIVIDE(ClosedInSLA, [Tickets Geschlossen])
```

---

## 5) Power Query – empfohlene Bereinigungen

* **Trim/Clean** auf alle Textspalten
* **Normalisieren** (z. B. alles UPPERCASE) für `Status`, `Reaktionsebene`, `Kategorie`, `Hardware`, `Software`, `Umgebung`
* **Fehlende Werte** auf `(UNBEKANNT)` setzen
* **TicketID** generieren (falls fehlt):

```m
let
    Source = stg_Tickets,
    AddIndex = Table.AddIndexColumn(Source, "TicketID", 1, 1, Int64.Type)
in
    AddIndex
```

---

## 6) Empfohlene Visuals (Start-Dashboard)

1. **KPI-Karten**: `Tickets Offen`, `Tickets Geschlossen`, `Ø Bearbeitungszeit (alle)`
2. **Säulen** (by `Dim_Date[YearMonth]`): `Tickets Gesamt`, `Tickets nach Schließdatum`
3. **Matrix**: `Dim_Classification`-Hierarchie in den Zeilen, Werte: `Tickets Gesamt`, `Ø Bearbeitungszeit (alle)`
4. **Heatmap**: Achsen `Reaktionsebene` × `Status`, Wert `Tickets Gesamt`
5. **Treemap**: `Dim_Org` (L1→L3) nach `Tickets Offen`
6. **Tabelle**: `aktueller SB`, `Workload aktueller SB (offen)`, `Backlog-Alter (Ø Tage, offen)`
7. **Slicer**: `Umgebung`, `Reaktionsebene`, `Status`, `YearMonth`

---

## 7) Governance & ITIL 4 Fit

* **Beginne dort, wo du stehst**: Nutze dein bestehendes Status-/Level-Vokabular, normalisiere konsequent.
* **Halte es einfach & praktisch**: Eine einzige Klassifikationsdimension mit 3 Stufen vermeidet Snowflaking.
* **Optimiere & automatisiere**: PQ-Regeln für Standardisierung (z. B. Mapping „IN PROGRESS“ → „IN BEARBEITUNG“).
* **Transparenz**: Dokumentiere Statusdefinitionen (wann zählt „geschlossen“?).

---

## 8) Nächste Schritte (Hands-on)

1. Quellen anbinden (Excel/SharePoint/CSV/SQL) → `stg_Tickets`
2. PQ-Bereinigung + `ClassificationKey` + `TicketID`
3. `Dim_Classification`, `Dim_Date`, weitere Dims erstellen
4. Beziehungen gemäß Abschnitt 1
5. Measures aus Abschnitt 4 anlegen
6. Visuals aus Abschnitt 6 platzieren

> Wenn du mir eine Beispiel-CSV/Excel-Struktur gibst (Spaltennamen + 5–10 Zeilen), erstelle ich dir die komplette Power Query und alle DAX-Measures exakt passend zu deinen echten Werten.
