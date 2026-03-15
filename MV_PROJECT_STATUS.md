# Maintenance Viewer (MV3) — Project Status
**Repo:** `RobMaas01/MV_PySide6`  |  **Branch:** master  |  **33 commits**  |  **15-03-2026**

---

## Wat is dit

Desktop-applicatie voor onderhoudspersoneel, CAMO en PM van het Defensie Helikopter Commando. Maakt de onderhoudsstatus van de NH90-vloot inzichtelijk op basis van het dagelijkse SAP-statusbord en referentiedata.

**Kernprincipe:** offline-first. Alle data wordt lokaal opgeslagen in SQLite — de app werkt volledig zonder netwerk.

---

## Architectuur

```
┌─────────────────────────────────────────────────────────────────┐
│  ETL-pipeline (aparte repo / buiten scope MV)                   │
│                                                                  │
│  SAP  →  MLIC (Oracle, 2×/week)  →  GitLab CI/CD (Python ETL)  │
│                                           │                      │
│                                     PostgreSQL (DBDAAP)          │
│                                     04_mart · configuratie       │
│                                     MIS · 3MS · ...             │
└─────────────────────────────────────┬───────────────────────────┘
                                      │ ← sync nog te bouwen (2×/week)
┌─────────────────────────────────────▼───────────────────────────┐
│  Maintenance Viewer (deze repo)                                  │
│                                                                  │
│  Statusbord XLS  ──(dagelijks handmatig)──────────►│            │
│                                                    │             │
│  PostgreSQL ──(toekomst, bij opstarten) ──────────►│            │
│                                                    │             │
│  GLIMS XML ──(toekomst, na elke vlucht) ──────────►│            │
│                                                    ▼             │
│                                           SQLite mv_data.db      │
│                                           (offline beschikbaar)  │
│                                                    │             │
│                         DataLoader (QThread) ◄─────┘            │
│                               │                                  │
│                          DataStore (DataFrames in RAM)           │
│                               │                                  │
│          ┌────────────────────┼────────────────────────┐        │
│       Home   Overview   Planning   ECU   Config*   MIS*         │
└─────────────────────────────────────────────────────────────────┘

* = QLabel placeholder, nog niet geïmplementeerd
```

### Drie databronnen in de app

| Bron | Inhoud | Hoe | Frequentie | Status |
|------|--------|-----|------------|--------|
| Statusbord XLS | PO-plan statussen, restwaarden per toestel | Home tab → bestandsdialoog → SQLite | Dagelijks handmatig | ✅ Werkt |
| PostgreSQL (DBDAAP) | Configuratie, MIS, 3MS, uitgebreide referentiedata | Nog te bouwen — connector + sync naar SQLite bij opstarten | Data 2× per week vers via ETL | 🔴 Nog te bouwen |
| GLIMS XML | Gevlogen uren, landingen, motorcycles na elke vlucht | Nog te bouwen — XML import via Home tab → SQLite | Na elke vlucht | 🔴 Nog te bouwen |

De statusbord XLS is een SAP-export. PostgreSQL wordt gevuld door de ETL-pipeline (aparte repo) en staat er volledig los van — de app haalt straks referentiedata op uit PostgreSQL zodra het netwerk beschikbaar is. GLIMS (Ground Logistic Information Management System, Leonardo Helicopters) exporteert na elke vlucht een XML met actuele tellerstanden — dit zorgt gedurende de dag voor een actueel beeld zonder te wachten op het volgende ochtend-statusbord.

---

## Bestandsstructuur

```
MV_PySide6/
├── main.py                    Entry point — QApplication + DataLoader start
├── launcher.py                Tkinter splashscreen + Popen MV3_app.exe + env-injectie
├── mv_config.ini              Pad-configuratie (datasource, settings, internal)
├── assets/
│   └── NH90_taskbar.ico
├── data/
│   ├── app_config.py          Pad-resolutie (frozen/dev, env vars, ini)
│   ├── app_state_service.py   Service-laag — ontkoppelt UI van processor-aanroepen
│   ├── database.py            SQLite I/O + Excel-import + auto-migratie
│   ├── glims_import.py        GLIMS XML import → SQLite (nog te bouwen)
│   ├── loader.py              QThread DataLoader
│   ├── planning_processor.py  Berekeningslogica planning-tab
│   ├── processor.py           Domeinlogica — prepare_statusbord, cal/cyc/bijz/ecu
│   └── store.py               DataStore (in-memory DataFrames, module-level singleton)
├── datasource/
│   ├── mv_data.db             SQLite database (meegeleverd in repo)
│   ├── statusbord.xlsx        Laatste import (niet in git)
│   └── configuratie.xlsx      Referentiedata (niet in git)
├── settings/
│   ├── MV_UserVariabelen.json Gebruikersinstellingen per Windows-user
│   ├── MV_SystemVariabelen.json Systeemconfiguratie (kenmerken, cycles, engine-groepen)
│   └── last_change.txt        Wijzigingsvlag — gepolled elke 2s door open sessies
└── ui/
    ├── theme.py               QSS stylesheet + kleurpalet (Slate-tinten)
    ├── main_window.py         QMainWindow + QTabWidget + QTimer polling
    └── tabs/
        ├── home_tab.py        Import, helikopter-selectie, work modes, statistieken
        ├── overview_tab.py    Onderhoudsstatus per helikopter (cal/uren/cyclus)
        ├── planning_tab.py    Vooruitblik inspecties + Excel export
        ├── ecu_tab.py         ECU 1/2 per toestel + filter + export
        └── settings_tab.py    Info/documentatie-tab
```

---

## Status per component

### Volledig werkend ✅

| Component | Wat het doet |
|-----------|-------------|
| `launcher.py` | Tkinter splashscreen, lanceert `MV3_app.exe` via Popen, injecteert `MV3_DATASOURCE` en `MV3_SETTINGS` als env vars |
| `main.py` | QApplication, Windows App User Model ID (taakbalk-icoon), laadt DataLoader als QThread |
| `app_config.py` | Leest `mv_config.ini`, ondersteunt zowel frozen (PyInstaller) als dev, absolute én relatieve paden, lazy singleton |
| `database.py` | SQLite I/O, generieke `import_excel_to_table`, `import_statusbord` met kolomvalidatie + kopiëren naar datasource-map, `_flag_touch()` na import |
| `store.py` | DataStore met auto-migratie (als tabel ontbreekt, importeer Excel automatisch), module-level singleton |
| `loader.py` | QThread achtergrondlader — blokkeert UI niet |
| `processor.py` | `prepare_statusbord`, `get_calendar_inspections`, `get_cycle_inspections`, `get_bijzonderheden`, `get_ecu_status`, JSON state (atomic write + `.lock` + stale-lock cleanup) |
| `app_state_service.py` | Ontkoppelt UI van directe processor-aanroepen, centraliseert lezen/schrijven user/system state |
| `main_window.py` | QTabWidget, `_LoginTracker` thread, statusbord-fingerprint cache, `last_change.txt` polling (2s), refresh-keten, `_refresh_overview(refresh_planning=True/False)` |
| **Home tab** | XLS import met validatie, helikopter-selectie per work mode, work modes (Flight MVKK / Out of area 1–3 / BVP), user-specifieke BVP-selectie, save-timer (400ms debounce), statistieken |
| **Overview tab** | Cal/uren/cyclus-inspecties per toestel naast elkaar, bijzonderheden, hide-completed, fingerprint-cache, Excel export met opmaak |
| **Planning tab** | Vooruitblik tot datum/uren/weken, intended use per missie-type, multi-kolom filtering (dropdown), sorteren, Excel export |
| **ECU tab** | ECU 1/2 toggle, SN uit configuratiedata, zelfde filter/sorteer-laag als Planning, export naar Excel (2 sheets) |
| **Info/Settings tab** | Applicatiedocumentatie (architectuur, stack, instellingen, berekeningslogica) |

### Placeholder — tab bestaat, inhoud ontbreekt ⏳

| Tab | Situatie |
|-----|----------|
| Configuration | `QLabel("Configuration")` — data beschikbaar in PostgreSQL (configuratie), tab-klasse ontbreekt |
| MIS | `QLabel("MIS")` — data beschikbaar in PostgreSQL (3MS/MIS), tab-klasse ontbreekt |
| Part. insp. | `QLabel("Part. insp.")` — invulling nog te bepalen |

### Nog te bouwen 🔴

| Item | Omschrijving |
|------|-------------|
| **PostgreSQL connector** | `psycopg2` (of `psycopg`) verbinding vanuit de app met DBDAAP. Online check bij opstarten. |
| **Sync PostgreSQL → SQLite** | Configuratie, MIS en 3MS ophalen en naar SQLite schrijven. Frequentie: 2× per week, automatisch bij online opstarten. SQLite blijft altijd de bron voor de app zelf. |
| **Configuration-tab** | Stuklijst per toestel: SAP-structuur, WC (As-Built) en TC (PVS/As-Designed) naast elkaar. Data is al beschikbaar in PostgreSQL. |
| **MIS-tab** | PoRef-overzicht: interval, drempelwaarde, volgende vervaldatum. Data beschikbaar via 3MS-extract in PostgreSQL. |
| **PyInstaller .exe build** | `MV3_app.exe` + `MV3.exe` (launcher). Spec-bestand, `_internal` bundel, versiecheck bij opstarten. |
| **Logging naar bestand** | `logging.FileHandler` naast de exe. Gestructureerde log van imports, laadfouten en PostgreSQL-verbindingsproblemen. |
| **Foutafhandeling PostgreSQL** | Duidelijke melding bij geen verbinding. App start gewoon op met SQLite-data, geen crash. |
| **GLIMS XML import** | Na elke vlucht MDS-vluchtdata via memory plug in GLIMS laden → XML exporteren → inlezen in app. Geeft actuele tellerstand gedurende de dag, zonder te wachten op het dagelijkse SAP-statusbord. |

---

## Directe volgende stap: PostgreSQL connector

Dit is de meest impactvolle toevoeging — het ontgrendelt Configuration-tab en MIS-tab tegelijk.

### Aanpak

**1. Dependency toevoegen**
```
pip install psycopg2-binary
```
Toevoegen aan `requirements.txt` (nog aan te maken).

**2. Configuratie uitbreiden** — `mv_config.ini`
```ini
[paths]
internal   = _internal
datasource = datasource
settings   = settings

[database]
host     = <dbdaap-host>
port     = 5432
dbname   = nexus
user     = mv_app
password = <wachtwoord>
```

Of via env var `MV3_PG_DSN` (veiliger voor gedeelde installaties).

**3. Nieuwe module** — `data/pg_sync.py`
```python
"""
Synchronisatie PostgreSQL → SQLite.
Wordt aangeroepen bij opstarten als netwerk beschikbaar is.
"""
import logging
import sqlite3
import psycopg2
import pandas as pd
from data.app_config import get_datasource_dir

log = logging.getLogger(__name__)

PG_TABLES = [
    # (postgresql-query of tabelnaam, sqlite-tabel)
    ("SELECT * FROM mv.configuratie",  "configuratie"),
    ("SELECT * FROM mv.mis",           "mis"),
    ("SELECT * FROM mv.tms",           "3ms"),
]

def sync_from_postgres(pg_dsn: str) -> dict[str, int | str]:
    """Haal data op uit PostgreSQL en schrijf naar SQLite. Geeft resultaat per tabel."""
    results = {}
    try:
        pg = psycopg2.connect(pg_dsn)
    except Exception as e:
        return {"error": str(e)}
    try:
        sqlite_db = str(get_datasource_dir() / "mv_data.db")
        for query, table in PG_TABLES:
            try:
                df = pd.read_sql(query, pg)
                conn = sqlite3.connect(sqlite_db)
                df.to_sql(table, conn, if_exists="replace", index=False)
                conn.close()
                results[table] = len(df)
                log.info("Gesynchroniseerd: %s (%d rijen)", table, len(df))
            except Exception as e:
                results[table] = f"FOUT: {e}"
                log.warning("Sync mislukt voor %s: %s", table, e)
    finally:
        pg.close()
    return results
```

**4. Aanroepen in `loader.py`**

Bij opstarten online check → `pg_sync.sync_from_postgres()` als QThread → daarna normale `DataStore.load()`.

**5. Frequentie bewaken**

Sla `pg_sync_last` op in `MV_SystemVariabelen.json`. Sync alleen als ouder dan 3 dagen (instelbaar).

---

## Open vragen

- [ ] Welke PostgreSQL schema/tabelnamen zijn van toepassing voor configuratie, MIS en 3MS?
- [ ] Hoe worden credentials beheerd — `mv_config.ini`, env var of Windows Credential Manager?
- [ ] Is PostgreSQL bereikbaar via MULAN zowel op MVKK als aan boord?
- [ ] Wat is de go/no-go datum voor de PostgreSQL connector?
- [ ] Moeten Part. insp. en de overige placeholder-tabs worden ingevuld of weggelaten?
- [ ] Exact XML-schema van de DHC GLIMS-installatie — voorbeeld-export nodig
- [ ] Wordt per vlucht één XML gegenereerd of cumulatief per dag?
- [ ] GLIMS XML beschikbaar via gedeeld netwerkpad (MULAN) of handmatig kopiëren?

---

## Installatie (ontwikkeling)

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

## GLIMS XML import — achtergrond en aanpak

### Wat is GLIMS

GLIMS (Ground Logistic Information Management System) is het Health & Usage Monitoring Systems (HUMS) platform van Leonardo Helicopters voor de NH90. Het is een geharde laptop die na elke vlucht de MDS-data (Monitoring and Diagnostic System) ontvangt via een memory plug uit het toestel.

GLIMS voert analyses uit op:
- Health & Usage Monitoring (HUM)
- Transmission Vibration Monitoring (TVM)
- Engine Vibration Monitoring (EVM)

En exporteert per vlucht een XML-bestand met de actuele tellerstand:
- Gevlogen uren
- Landingen
- Motorcycles (starts per motor)
- Overige cyclische tellers

### Waarom relevant voor de Maintenance Viewer

Het dagelijkse SAP-statusbord wordt 's ochtends handmatig geëxporteerd. Gedurende de dag veranderen de tellers door vluchten — maar die wijzigingen zijn pas de volgende ochtend zichtbaar in de app.

Met de GLIMS XML-import wordt na elke vlucht de actuele tellerstand ingelezen. Daarmee is de resterende capaciteit tot het volgende onderhoud gedurende de hele dag actueel, ook als er meerdere vluchten op één dag zijn.

### Aanpak implementatie

**1. XML-formaat bepalen**
Het exacte GLIMS XML-schema is operatorafhankelijk. Eerst het formaat in kaart brengen op basis van een voorbeeld-export van de DHC GLIMS-installatie.

**2. Nieuwe module** — `data/glims_import.py`
```python
import xml.etree.ElementTree as ET
import sqlite3
from pathlib import Path

def import_glims_xml(path: Path) -> dict:
    """
    Leest GLIMS XML-export en schrijft tellers naar SQLite tabel 'glims_tellers'.
    Geeft {'aircraft': str, 'tellers': int, 'error': str | None}.
    """
    try:
        tree = ET.parse(path)
        root = tree.getroot()
        # Parse afhankelijk van GLIMS XML-schema DHC
        # Velden: aircraft_id, vlucht_datum, uren, landingen, motorcycles, ...
        ...
    except Exception as e:
        return {'aircraft': '', 'tellers': 0, 'error': str(e)}
```

**3. SQLite tabel** — `glims_tellers`
Aparte tabel naast `statusbord` — de GLIMS-data is een delta bovenop het statusbord, niet een vervanging.

**4. Integratie in Overview-tab**
Bij het berekenen van restwaarden: als er een recentere GLIMS-meting bestaat voor een toestel (timestamp na statusbord-import), gebruik dan de GLIMS-tellerstand in plaats van de statusbord-waarde.

**5. Import via Home-tab**
Knop "Import GLIMS XML" naast de bestaande statusbord-importknop. Zelfde patroon: bestandsdialoog → validatie → SQLite → `_flag_touch()` → herladen.

### Open vragen GLIMS
- [ ] Wat is het exacte XML-schema van de DHC GLIMS-installatie?
- [ ] Wordt per vlucht één XML gegenereerd of cumulatief per dag?
- [ ] Hoe is het XML-bestand toegankelijk — gedeeld pad op MULAN of handmatig kopiëren?
- [ ] Welke tellers staan in de XML — zijn dat dezelfde als in het statusbord?

---

## Domeinbegrippen

| Term | Betekenis |
|------|-----------|
| Statusbord | SAP-export (XLS) met PO-plan statussen en restwaarden per toestel |
| PO-plan | SAP Maintenance Plan — definieert onderhoudstaak per toestel per interval |
| PoRef | Referentie van een MIS-taak (= PO-plan naam) |
| MIS | Maintenance Inspection Schedule — takenschema binnen het AMP |
| AMP | Aircraft Maintenance Programme |
| WC | Werkelijke Configuratie (As-Built) |
| TC | Toegestane Configuratie (As-Designed / PVS) |
| ECU | Engine Control Unit — per toestel twee exemplaren (ECU 1 en ECU 2) |
| 3MS | Defensie tool die PoRefs beheert en SAP PO-plannen valideert |
| MLIC | Oracle-spiegel van SAP, 2× per week bijgewerkt |
| GLIMS | Ground Logistic Information Management System (Leonardo Helicopters) — ruggedized laptop die MDS-data via memory plug ontvangt na elke vlucht. Voert HUMS-analyse uit (Health & Usage, TVM, EVM) en exporteert XML met gevlogen uren, landingen, motorcycles en overige tellers. |
| DBDAAP | Intern Defensie platform (PostgreSQL + GitLab CI/CD) |
| MULAN | Defensie bedrijfsnetwerk |
| Work mode | Selectieprofiel: Flight MVKK / Out of area 1–3 / BVP (user-specifiek) |
| BVP | Bijzonder VliegProgramma — user-specifieke helikopterselectie |
