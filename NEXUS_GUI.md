# NEXUS — Fleet Materiel Logistics BI Platform
## Gebruikerslaag & GUI — Maintenance Viewer (MV3)
**Defensie Helikopter Commando**
Versie 0.2 | 15-03-2026 | Status: Actief in ontwikkeling

> **Scope:** Dit document beschrijft de Maintenance Viewer (PySide6 desktop app),
> de architectuur, databronnen, werkende functionaliteit en roadmap.
> Voor de ETL pipeline zie `NEXUS_ETL.md`.
> Voor domeinachtergrond zie `NEXUS_CONTEXT.md`.

---

## 1. Wat is de Maintenance Viewer

De Maintenance Viewer (MV3) is een PySide6 desktop-applicatie voor onderhoudspersoneel,
CAMO en PM van het DHC. De app maakt de onderhoudsstatus van de NH90-vloot inzichtelijk
en vervangt de bestaande MS Access versie.

**Kernprincipe — offline-first:**
Alle data wordt lokaal opgeslagen in een SQLite-database (`datasource/mv_data.db`).
De app werkt volledig zonder netwerkverbinding. Zodra het netwerk beschikbaar is,
worden referentiegegevens automatisch ververst.

**Waarom niet gewoon SAP raadplegen?**
SAP beheert PO-plannen per toestel en per teller (kalender, uren, landingen, etc.)
in afzonderlijke views. Een volledig beeld van één toestel kost al moeite; over
meerdere helikopters tegelijk is dat praktisch niet werkbaar. De Maintenance Viewer
combineert alle tellers van alle toestellen op één scherm.

**Repo:** `RobMaas01/MV_PySide6` (GitHub / DBDAAP GitLab)
**Stack:** Python 3.12 · PySide6 (Qt6) · pandas · SQLite · openpyxl

---

## 2. Architectuur

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Databronnen                                                             │
│                                                                          │
│  [SAP]              [PostgreSQL (DBDAAP)]          [GLIMS]              │
│  bronsysteem        via SAP→MLIC→ETL→PG            Leonardo Helicopters │
│                     2x per week verse data          na elke vlucht       │
│       │                      │                          │                │
│  Statusbord XLS         toekomst:                  toekomst:            │
│  dagelijks handmatig    directe koppeling          XML export           │
│       │                      │                          │                │
│       ▼                      ▼                          ▼                │
│  ┌────────────────────────────────────────────────────────────┐         │
│  │              SQLite  —  mv_data.db                         │         │
│  │         lokale database · offline beschikbaar              │         │
│  └────────────────────────────────────────────────────────────┘         │
│                              │                                           │
│                    DataLoader (QThread)                                  │
│                              │                                           │
│                    DataStore (in-memory DataFrames)                      │
│                              │                                           │
│        Home │ Overview │ Planning │ ECU │ Config* │ MIS*                │
│                                                                          │
│  * = QLabel-placeholder, nog niet geïmplementeerd                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Bestandsstructuur

```
MV_PySide6/
├── main.py                    Entry point — QApplication + DataLoader start
├── launcher.py                Tkinter splashscreen + Popen MV3_app.exe
├── mv_config.ini              Pad-configuratie (datasource, settings, internal)
├── assets/
│   └── NH90_taskbar.ico
├── data/
│   ├── app_config.py          Pad-resolutie (frozen/dev, env vars, ini)
│   ├── app_state_service.py   Service-laag — ontkoppelt UI van processor
│   ├── database.py            SQLite I/O + Excel-import + auto-migratie
│   ├── loader.py              QThread DataLoader
│   ├── planning_processor.py  Berekeningslogica planning-tab
│   ├── processor.py           Domeinlogica + JSON state beheer
│   └── store.py               DataStore (in-memory DataFrames, singleton)
├── datasource/
│   ├── mv_data.db             SQLite database (meegeleverd in repo)
│   ├── statusbord.xlsx        Laatste import (niet in git)
│   └── configuratie.xlsx      Referentiedata (niet in git)
├── settings/
│   ├── MV_UserVariabelen.json Gebruikersinstellingen per Windows-gebruiker
│   ├── MV_UserVariabelen.lock Schrijflock (advisory, stale na 5s automatisch verwijderd)
│   ├── MV_SystemVariabelen.json Systeemconfiguratie (kenmerken, cycles, engine-groepen)
│   └── last_change.txt        Wijzigingsvlag — gepolled door open sessies
└── ui/
    ├── theme.py               QSS stylesheet + Slate-kleurpalet
    ├── main_window.py         QMainWindow + QTabWidget + polling
    └── tabs/
        ├── home_tab.py
        ├── overview_tab.py
        ├── planning_tab.py
        ├── ecu_tab.py
        └── settings_tab.py
```

---

## 3. Opstartpad en launcher

**`launcher.py` (MV3.exe):**
Klein opstartprogramma gebouwd met Tkinter (ingebouwd in Python, geen extra dependency).
Toont een splashscreen, start `MV3_app.exe` via `subprocess.Popen` (niet-blokkerend),
wacht via Windows API `WaitForInputIdle` tot de PySide6-app gereed is, en vernietigt
dan het splashscreen. Injecteert `MV3_DATASOURCE` en `MV3_SETTINGS` als environment
variables zodat de app het gedeelde netwerkpad gebruikt.

**`main.py` (MV3_app.exe):**
Start de `QApplication`, zet de Windows App User Model ID zodat het NH90-icoon in de
taakbalk verschijnt, toont het hoofdvenster gemaximaliseerd, en start een `DataLoader`
als `QThread` die de SQLite-data op de achtergrond inlaadt zonder de UI te blokkeren.

**`mv_config.ini`:**
Pad-configuratie naast de exe. Ondersteunt zowel absolute als relatieve paden.
Als het bestand niet bestaat, wordt het automatisch aangemaakt met standaardwaarden.

```ini
[paths]
internal   = _internal
datasource = datasource
settings   = settings
```

---

## 4. Multi-sessie synchronisatie

Meerdere gebruikers kunnen de app tegelijk openen met hetzelfde gedeelde
`datasource/`-pad op het netwerk. Twee mechanismen zorgen voor synchronisatie:

**`last_change.txt` — databron-polling:**
Na elke succesvolle statusbord-import raakt `database.py` dit bestand aan via
`touch_meta()` (schrijft lege inhoud, update alleen de modificatietijd).
Alle open sessies pollen elke 2 seconden of de `mtime` is veranderd.
Bij wijziging: volledig herladen inclusief herberekening van alle DataFrames
(`refresh_planning=True`).

**`MV_UserVariabelen.json` mtime — instellingen-polling:**
Dezelfde QTimer bewaakt ook de modificatietijd van de gebruikersinstellingen.
Bij wijziging door een andere sessie: alleen de Overview-tab verversen zonder
herberekening (`refresh_planning=False`). Eigen opslag wordt gemarkeerd zodat
de poller dat niet opnieuw als externe wijziging oppikt.

**Praktisch voorbeeld:**
CAMO importeert 's ochtends het nieuwe statusbord op zijn laptop. Een technicus
heeft de app open op een andere laptop — die ziet binnen 2 seconden automatisch
de nieuwe data verschijnen zonder de app opnieuw te hoeven starten.

**Schrijf-synchronisatie JSON:**
`MV_UserVariabelen.json` wordt geschreven via een `.lock`-bestand (advisory
file-lock). Stale locks ouder dan 5 seconden worden automatisch verwijderd.
Schrijven zelf gaat atomisch via een tijdelijk `.tmp`-bestand.

---

## 5. Tabs — werkende functionaliteit

### 5.1 Home-tab ✅

- Statusbord XLS importeren via bestandsdialoog
- Kolomvalidatie — foutmelding bij ontbrekende kolommen, bestand gekopieerd naar datasource-map
- Helikopter-selectie per work mode
- Work modes: `Flight MVKK` / `Out of area 1` / `Out of area 2` / `Out of area 3` / `BVP`
- BVP is user-specifiek (per Windows-gebruikersnaam), andere modes zijn gedeeld
- Instellingen opgeslagen met 400ms debounce-timer, atomisch via `.lock`
- Statistieken: aantal rijen statusbord en configuratie
- Knop "Import statusboard" met voortgang en bevestigingsmelding (rijen oud/nieuw)

### 5.2 Overview-tab ✅

- Per toestel: kalender-, uren- en cyclusinspecties naast elkaar
- Restwaarden direct zichtbaar vanuit statusbord
- Bijzonderheden handmatig invoerbaar per inspectie (met datum of tellerwaarde)
- "Hide completed inspecties" schakelaar in de tabbalk (per gebruiker)
- Export naar Excel met opmaak (één klik, opent direct in Excel via `os.startfile`)
- Statusbord-fingerprint cache — herlaadt alleen bij gewijzigde data, niet bij elke filterwissel
- ATL Report knop per toestel (PDF export)
- Serienummers en tellerstand opvraagbaar per toestel via info-dialogen

### 5.3 Planning-tab ✅

- Inspecties filteren op periode (van/tot datum), vlieguren en weken aan boord
- Intended use invoer per missie-type (cycli per periode)
- Multi-kolom filtering via dropdown per kolom (multi-select popup)
- Sorteren op elke kolom
- Export naar Excel met stylesheeting (opent direct in Excel)
- Aircraft-dropdown met per-toestel refresh via debounce-timer

### 5.4 ECU-tab ✅

- ECU 1 / ECU 2 toggle per toestel
- Serienummer opgehaald uit configuratiedata
- Zelfde filter- en sorteerlaag als Planning-tab
- Export naar Excel met ECU 1 en ECU 2 op aparte sheets
- Aircraft-dropdown identiek aan Planning-tab

### 5.5 Info/Settings-tab ✅

- Applicatiedocumentatie: architectuur, stack, instellingen, berekeningslogica
- Statisch — toont technische uitleg van de app zelf

### 5.6 Configuration-tab ⏳ PLACEHOLDER

Toont nu alleen `QLabel("Configuration")`. Tab-klasse ontbreekt.
Data beschikbaar in PostgreSQL (configuratie-tabel).

Doel: stuklijst per toestel — SAP-structuur, Werkelijke Configuratie (As-Built / WC)
en Toegestane Configuratie (As-Designed / PVS / TC) naast elkaar.

### 5.7 Part. insp.-tab ⏳ PLACEHOLDER

Toont nu alleen `QLabel("Part. insp.")`. Invulling nog te bepalen.

### 5.8 MIS-tab ⏳ PLACEHOLDER

Toont nu alleen `QLabel("MIS")`. Tab-klasse ontbreekt.
Data beschikbaar in PostgreSQL (3MS-extract).

Doel: overzicht van alle MIS-taken met PoRef, interval, drempelwaarde
en volgende vervaldatum.

---

## 6. Databronnen

### 6.1 Statusbord XLS — dagelijks handmatig ✅

SAP-export als Excel-bestand. Elke ochtend handmatig gedownload en via de
Home-tab geïmporteerd. Na import wordt `last_change.txt` geraakt zodat alle
open sessies herladen.

Vereiste kolommen: `ID/tactisch teken`, `PO-plan`, `Tekst PO-plan`,
`Ref.equipment`, `Ref.func.plaats`, `Restwaarde teller`.

### 6.2 PostgreSQL (DBDAAP) — toekomst ⏳

Directe koppeling vanuit de app nog te bouwen (`psycopg2`).
De data in PostgreSQL is 2x per week vers via de ETL-pipeline
(SAP → MLIC → Python ETL → PostgreSQL). De app haalt bij opstarten
de referentiedata op als het netwerk beschikbaar is.

Te halen data: configuratie, MIS-taken, 3MS PoRef-structuur.
SQLite blijft de lokale cache en offline fallback.

**Aanpak:**
```python
# data/pg_sync.py (nog te bouwen)
import psycopg2, pandas as pd, sqlite3
from data.app_config import get_datasource_dir

PG_TABLES = [
    ("SELECT * FROM mv.configuratie", "configuratie"),
    ("SELECT * FROM mv.mis",          "mis"),
    ("SELECT * FROM mv.tms",          "3ms"),
]

def sync_from_postgres(pg_dsn: str) -> dict:
    # verbinden, ophalen, naar SQLite schrijven
    ...
```

Configuratie via `mv_config.ini` sectie `[database]` of env var `MV3_PG_DSN`.

### 6.3 GLIMS XML — toekomst ⏳

GLIMS (Ground Logistic Information Management System) is het HUMS-platform
van Leonardo Helicopters voor de NH90. Na elke vlucht wordt MDS-data
(Monitoring and Diagnostic System) via een memory plug in GLIMS geladen.

GLIMS exporteert een XML met:
- Gevlogen uren
- Landingen
- Motorcycles (starts per motor / ECU)
- Overige cyclische tellers

**Meerwaarde:** het dagelijkse SAP-statusbord wordt 's ochtends éénmalig
geëxporteerd. Gedurende de dag veranderen de tellers door vluchten.
Met de GLIMS XML-import is de resterende capaciteit na elke vlucht actueel —
zonder te wachten op het volgende ochtend-statusbord.

**Aanpak (nog te bouwen):**
- `data/glims_import.py` — XML parsing via `xml.etree.ElementTree`
- Aparte SQLite-tabel `glims_tellers` naast `statusbord`
- Import via Home-tab knop "Import GLIMS XML"
- Overview-tab gebruikt GLIMS-meting als die recenter is dan het statusbord

**Open vragen GLIMS:**
- [ ] Exact XML-schema van de DHC GLIMS-installatie?
- [ ] Wordt per vlucht één XML gegenereerd of cumulatief per dag?
- [ ] XML beschikbaar via gedeeld pad op MULAN of handmatig kopiëren?

---

## 7. Deployment omgevingen

| Omgeving | Netwerk | PostgreSQL | Strategie |
|----------|---------|------------|-----------|
| MVKK thuisbasis | MULAN bedrijfsnetwerk | ✅ Altijd | Online — live data |
| Aan boord schip | MULAN via satelliet | ⚠️ Kan wegvallen | Online met SQLite fallback |
| Out of base | MULAN indien beschikbaar | ⚠️ Situatieafhankelijk | Online met SQLite fallback |
| Operationeel black hole | Geen netwerk | ❌ Geen verbinding | Volledig offline — SQLite only |

**Operationeel black hole:** geen verbinding met het thuisnetwerk, kan dagen
tot weken duren. De app moet volledig functioneel blijven op de lokale SQLite.

---

## 8. Gebruikers

| Gebruikersgroep | Locatie | Primair gebruik |
|-----------------|---------|-----------------|
| CAMO DHC | Gilze-Rijen + MVKK De Kooy | Statusoverzicht, planning, luchtwaardigheid |
| CAMO PM (LCW) | Woensdrecht | Configuratie, TC-beheer |
| -145 lijnonderhoud | MVKK De Kooy | Dagelijkse onderhoudsstatus per toestel |
| -145 hoger echelon | LCW Woensdrecht | Zware inspecties, MIS-planning |
| PM NH90 (LCW) | Woensdrecht | Vlootplanning, operationele inzet |

---

## 9. Status per component

### Volledig werkend ✅

| Component | Wat het doet |
|-----------|-------------|
| `launcher.py` | Tkinter splashscreen, Popen MV3_app.exe, env-injectie |
| `main.py` | QApplication, Windows App ID, DataLoader thread |
| `app_config.py` | mv_config.ini parsing, frozen/dev pad-resolutie, lazy singleton |
| `database.py` | SQLite I/O, import_statusbord met validatie + kopiëren, auto-migratie Excel |
| `store.py` | DataStore met auto-migratie, module-level singleton |
| `loader.py` | QThread achtergrondlader |
| `processor.py` | prepare_statusbord, get_calendar/cycle_inspections, get_bijzonderheden, get_ecu_status |
| `app_state_service.py` | Ontkoppelt UI van directe processor-aanroepen |
| `main_window.py` | QTabWidget, fingerprint-cache, last_change.txt + user_vars polling (2s), refresh-keten |
| `planning_processor.py` | Berekeningslogica Planning-tab |
| **Home-tab** | XLS import, heli-selectie, work modes, BVP per user, statistieken |
| **Overview-tab** | Cal/uren/cyclus per toestel, bijzonderheden, hide-completed, fingerprint-cache, Excel/PDF export |
| **Planning-tab** | Vooruitblik, intended use, multi-kolom filter, sort, Excel export |
| **ECU-tab** | ECU 1/2 toggle, SN, filter/sort, dual-sheet Excel export |
| **Info/Settings-tab** | Applicatiedocumentatie |
| Atomic JSON write + .lock + last_change.txt polling | Multi-sessie synchronisatie |

### Placeholder — tab bestaat, inhoud ontbreekt ⏳

| Tab | Situatie |
|-----|----------|
| Configuration | `QLabel` — data in PostgreSQL, tab-klasse ontbreekt |
| Part. insp. | `QLabel` — invulling nog te bepalen |
| MIS | `QLabel` — data in PostgreSQL (3MS), tab-klasse ontbreekt |

### Nog te bouwen 🔴

| Item | Prioriteit | Omschrijving |
|------|-----------|-------------|
| PostgreSQL connector | Hoog | `psycopg2` koppeling, sync bij opstarten, SQLite als fallback |
| Sync PostgreSQL → SQLite | Hoog | Configuratie, MIS, 3MS ophalen en cachen |
| GLIMS XML import | Hoog | XML parser, aparte SQLite-tabel, integratie Overview-tab |
| Configuration-tab | Hoog | WC/TC stuklijst per toestel |
| MIS-tab | Hoog | PoRef-overzicht met intervallen en vervaldatums |
| PyInstaller packaging | Middel | MV3_app.exe + MV3.exe, spec-bestand, versiecheck |
| Logging naar bestand | Middel | FileHandler naast de exe, PostgreSQL-verbindingsfouten |
| Foutafhandeling PostgreSQL | Middel | Geen crash bij geen verbinding, duidelijke melding |

---

## 10. Installatie (ontwikkeling)

```bash
pip install pyside6 pandas openpyxl
python main.py
```

Toekomstig (productie):
```bash
pip install pyside6 pandas openpyxl psycopg2-binary pyinstaller
pyinstaller MV3.spec
```

---

## 11. Open vragen

- [ ] Welk PostgreSQL schema/tabelnamen voor configuratie, MIS en 3MS?
- [ ] Credentials beheer — mv_config.ini, env var of Windows Credential Manager?
- [ ] Is PostgreSQL bereikbaar via MULAN zowel op MVKK als aan boord?
- [ ] Exact XML-schema van DHC GLIMS-installatie?
- [ ] GLIMS XML via gedeeld netwerkpad of handmatig kopiëren?
- [ ] Moeten Part. insp. en overige placeholders worden ingevuld of verwijderd?
- [ ] Go/no-go datum voor PostgreSQL connector?

---

## 12. Domeinbegrippen

| Term | Betekenis |
|------|-----------|
| AMP | Aircraft Maintenance Programme — door MLA goedgekeurd onderhoudsplan |
| MIS | Maintenance Inspection Schedule — takenschema binnen het AMP |
| PoRef | Referentie van een onderhoudstaak (= PO-plan naam in SAP) |
| PO-plan | SAP Maintenance Plan — onderhoudstaak per toestel per interval |
| WC | Werkelijke Configuratie (As-Built) — wat er op de heli zit |
| TC | Toegestane Configuratie (As-Designed/PVS) — wat er mag zitten |
| ECU | Engine Control Unit — per NH90 twee exemplaren (ECU 1 en ECU 2) |
| MLIC | Oracle-spiegel van SAP, 2x per week bijgewerkt (zo/di nacht) |
| 3MS | Defensie tool die PoRefs beheert en SAP PO-plannen valideert |
| GLIMS | Ground Logistic Information Management System (Leonardo Helicopters) — ruggedized laptop, ontvangt MDS-data via memory plug na elke vlucht, exporteert XML met uren/landingen/motorcycles |
| MDS | Monitoring and Diagnostic System — boordcomputer die vliegdata registreert |
| HUMS | Health and Usage Monitoring System — bewaakt component-slijtage en trillingen |
| CAMO | Continued Airworthiness Management Organisation |
| MLA | Militaire Luchtvaart Autoriteit |
| DHC | Defensie Helikopter Commando |
| LCW | Logistiek Centrum Woensdrecht |
| MVKK | Marine Vliegkamp Valkenburg De Kooy (Den Helder) |
| MULAN | Defensie bedrijfsnetwerk |
| DBDAAP | Intern Defensie platform (PostgreSQL + GitLab CI/CD) |
| Work mode | Selectieprofiel: Flight MVKK / Out of area 1–3 / BVP |
| BVP | Bijzonder VliegProgramma — user-specifieke helikopterselectie |
| last_change.txt | Wijzigingsvlag in settings-map — mtime gepolled elke 2s door open sessies |

---

## 13. Versiehistorie

| Versie | Datum | Wijziging |
|--------|-------|-----------|
| 0.1 | 15-03-2026 | Initieel op basis van NEXUS_CONTEXT + NEXUS_GUI docs |
| 0.2 | 15-03-2026 | Volledig herschreven op basis van broncode-analyse MV_PySide6 repo — werkelijke tabs, databronnen, architectuur, polling-mechanismen, GLIMS-context toegevoegd |
