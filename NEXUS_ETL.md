# NEXUS — Fleet Materiel Logistics BI Platform
## ETL Pipeline
**Defensie Helikopter Commando**
Versie 0.1 | 15-03-2026 | Status: In uitwerking

> **Scope:** Dit document beschrijft alleen de ETL pipeline.
> Voor de GUI applicaties zie `NEXUS_GUI.md`.
> Voor domeinachtergrond zie `NEXUS_CONTEXT.md`.

---

## 1. Projectoverzicht

### 1.1 Context
> TODO: Welke NH90 logistieke processen worden ondersteund?
> bijv. onderhoud, inkoop, vlieguren, onderdelen-tracking, inspecties

### 1.2 Doelstelling
> TODO: Wat moet de ETL pipeline uiteindelijk kunnen?
> Welk probleem lost het op t.o.v. de huidige situatie?

### 1.3 Stakeholders
> TODO: Wie beheert de ETL? Wie is afhankelijk van correcte data-levering?

---

## 2. Architectuur & Technologie Stack

### 2.1 Vastgestelde stack

| Onderdeel      | Keuze                        | Opmerking                                      |
|----------------|------------------------------|------------------------------------------------|
| Bronsysteem    | SAP                          | Materieel logistiek brondata                   |
| SAP spiegel    | Oracle DB (MLIC)             | Periodieke dump: zondag- en dinsdagnacht       |
| 3MS extract    | PostgreSQL (01_src)          | Defensie tool — PoRef-structuur en PO plan validatie |
| ETL taal       | Python                       | Nu: los script (VB). Straks: GitLab CI/CD      |
| CI/CD          | GitLab CI/CD (DBDAAP)        | In ontwikkeling — nog niet in productie        |
| Database       | PostgreSQL                   | Doeldatabase, op DBDAAP platform               |
| Versiebeheer   | GitLab                       |                                                |

### 2.2 Architectuurschema (data-flow)

**Huidige situatie (productie):**
```
SAP
 └─► Oracle DB (MLIC)          ← periodieke dump (zo/di nacht)
      └─► Python script (VB)   ← handmatig of gepland gestart
           ├─ leest query-tabel uit PostgreSQL (querynaam + SQL)
           ├─ voert SQL's uit op MLIC (Oracle)
           ├─ checkt via stored procedure of doeltabel bestaat
           │   └─ aanmaken of aanpassen indien nodig
           └─► PostgreSQL
                ├─ 01_src        (ruwe data uit MLIC + 3MS extract)
                ├─ 02_stage      (views, select * from src)
                ├─ 03_ssot_nh90  (views, soms aangepast, NH90-filter)
                └─ 04_mart       (eindproduct voor rapportage / GUI)

3MS (Defensie tool)
 └─► 3MS data extract → PostgreSQL 01_src  ← naast SAP/MLIC data
```

**Toekomstige situatie (in ontwikkeling — GitLab CI/CD op DBDAAP):**
```
SAP
 └─► Oracle DB (MLIC)           ← periodieke dump (zo/di nacht)
      └─► GitLab CI/CD (DBDAAP) ← automatisch getriggerd
           └─► Python ETL        ← zelfde logica, geautomatiseerd
                └─► PostgreSQL   ← zelfde laagstructuur
```
> ⚠️ Verbinding MLIC ↔ PostgreSQL via DBDAAP werkt al in testomgeving.
> Pipeline moet nog worden uitgewerkt, getest en gevalideerd voor productie-overgang.

### 2.3 Databronnen

| Bron          | Formaat        | Frequentie                    | Opmerking                        |
|---------------|----------------|-------------------------------|----------------------------------|
| SAP           | Via Oracle     | 2x per week (zo/di nacht)     | Gespiegeld naar MLIC             |
| MLIC (Oracle) | Relationeel    | Gelezen door ETL script       | SQL's opgeslagen in PostgreSQL   |
| 3MS           | Extract        | Bij wijziging AMP/PoRefs      | PoRef-structuur, PO plan validatie |
| IETP          | XML (S1000D)   | Bij nieuwe revisie van OEM    | Technische publicaties NH90      |

> **3MS** is een Defensie tool die de PoRefs uit het AMP beheert en valideert of
> de SAP PO plannen correct zijn aangemaakt en geconfigureerd. De 3MS data wordt
> als extract geladen in PostgreSQL (01_src) en is beschikbaar als databron voor NEXUS.
> De 3MS data bevat de PoRef-structuur — welke PO plannen bij welke AMP-taak horen.
> Zie `NEXUS_MPD_naar_POplan.md` voor de volledige context.

> **IETP — Interactive Electronic Technical Publication (S1000D)**
> De NH90 IETP bestaat uit Data Modules (DM), elk geïdentificeerd via een unieke
> Data Module Code (DMC). Elke DM is een zelfstandige XML-file.
> Drie series zijn relevant voor NEXUS:
>
> | Serie | DM type | Inhoud | Gebruik in NEXUS |
> |-------|---------|--------|-----------------|
> | **IPD** | Illustrated Parts Data | Stuklijsten — PNs en posities per assembly | Configuratie / PN-lijst Maintenance Viewer |
> | **AMP ch.4** | Airworthiness Limitations | Harde luchtwaardigheidslimieten — Life Limited Parts, nooit overschrijden | AMP inspectieplanning |
> | **AMP ch.5** | Time Limits / Scheduled Maintenance | Geplande onderhoudstaken met intervallen (uren/cycli/kalender) | AMP inspectieplanning |
> | **DM (proced)** | Procedural | Onderhoudsprocedures en werkkaarten — elk een eigen DMC | Referentie vanuit AMP-taken naar procedure |

---

## 3. Huidige status — wat hebben we al

> Statussen: ✅ Klaar | 🔄 Basis werkt | ⚠️ Onvolledig | ⏳ Nog te doen

### 3.1 ETL Pipeline

> **Belangrijk onderscheid:**
> - **Productie (nu):** Los Python script — "VB script", handmatig of gepland gestart
> - **In ontwikkeling:** GitLab CI/CD op DBDAAP — verbinding werkt, pipeline nog niet af

| Status | Onderdeel                    | Huidige situatie                                              |
|--------|------------------------------|---------------------------------------------------------------|
| ✅     | Brondefinitie in DB          | Query-tabel in PostgreSQL met querynaam + SQL                 |
| ✅     | Extractie uit Oracle (MLIC)  | Script leest query-tabel, voert SQL's uit op MLIC            |
| ✅     | Load naar PostgreSQL         | Data wordt opgeslagen in 01_src                              |
| ✅     | Tabel-beheer via stored proc | SP checkt of tabel bestaat, maakt aan of past aan            |
| ✅     | 3MS extract in PostgreSQL    | 3MS data beschikbaar in 01_src naast SAP/MLIC data           |
| 🔄     | Laagstructuur src→stage→ssot | Werkt via views (select *), soms aangepaste views            |
| 🔄     | Views aanmaken via SP's      | Mechanisme werkt, nog niet volledig consistent               |
| 🔄     | GitLab CI/CD (DBDAAP)        | In ontwikkeling — verbinding werkt, pipeline nog niet af     |
| ⚠️     | NH90-filter in SQL           | Hardcoded filter in de SQL's — niet generiek                 |
| ⚠️     | Error handling               | Onduidelijk wat er gebeurt bij falende SQL of verbinding     |
| ⚠️     | Logging                      | Geen gestructureerde logging van runs en fouten              |
| ⏳     | Migratie script → CI/CD      | Testen, valideren en overstap productiescript plannen        |
| ⏳     | Multi-type ondersteuning     | Plan: later ook andere helikoptertypes ophalen               |
| ⏳     | Unit tests                   | Geen geautomatiseerde tests                                  |
| ⏳     | Documentatie                 | Weinig tot geen inline documentatie                          |

### 3.2 PostgreSQL laagstructuur

| Schema           | Doel                                 | Huidige invulling                                      |
|------------------|--------------------------------------|--------------------------------------------------------|
| `01_src`         | Ruwe data uit MLIC + 3MS extract     | Tabellen aangemaakt/bijgehouden via SP                 |
| `02_stage`       | Eerste opschoning / standaard views  | Views: grotendeels `SELECT * FROM src`                 |
| `03_ssot_nh90`   | Single Source of Truth voor NH90     | Views, soms aangepast, NH90-filter actief              |
| `04_mart`        | Eindproduct voor rapportage / GUI    | TODO: hoe ver is dit uitgewerkt?                       |
| `etl_run_log`    | Run-historie en monitoring           | Nog te bouwen                                          |

---

## 4. Verbeteringen & advies

### 4.1 🔴 Prioriteit HOOG — Foutafhandeling & logging

**Probleem:** Als een SQL faalt, een verbinding wegvalt of een tabel niet aangemaakt kan worden,
is het onduidelijk wat er gebeurt. Bij een periodieke nachtrun zonder toezicht is dit risicovol.

**Advies:**
- Implementeer gestructureerde logging met Python `logging` module (of `loguru`)
- Log per query: starttijd, eindtijd, aantal rijen geladen, status (ok/fout)
- Sla logs op in een aparte PostgreSQL tabel `etl_run_log` — run-historie opvraagbaar
- Stuur een notificatie bij fouten (e-mail of GitLab pipeline alert)
- Voeg `try/except` blokken toe rondom elke query-executie met zinvolle foutmeldingen

```sql
-- Voorbeeld structuur etl_run_log tabel
CREATE TABLE etl_run_log (
    id             SERIAL PRIMARY KEY,
    run_id         UUID,
    query_naam     TEXT,
    schema_doel    TEXT,
    tabel_doel     TEXT,
    gestart_op     TIMESTAMP,
    beeindigd_op   TIMESTAMP,
    rijen_geladen  INTEGER,
    status         TEXT,  -- 'ok' | 'fout' | 'waarschuwing'
    foutmelding    TEXT
);
```

---

### 4.2 🔴 Prioriteit HOOG — Migratie script → GitLab CI/CD

**Situatie:** CI/CD op DBDAAP werkt technisch, maar is nog niet productierijp.
Zolang het op een handmatig script draait: risico op vergeten runs, geen audit trail,
afhankelijkheid van één persoon.

**Advies:**
- Definieer **acceptatiecriteria**: wanneer is de pipeline "klaar genoeg"?
  Bijv: 3 succesvolle volledige testruns, logging werkt, alert bij mislukken
- Draai script en CI/CD **parallel** gedurende minimaal 2 weken — vergelijk output
- Stel een **go/no-go datum** in, niet open einde laten
- Zorg dat CI/CD de bestaande stored procedure aanroept — geen dubbele logica
- Zet een **fallback procedure** klaar: wat als CI/CD faalt, wie start dan handmatig?

---

### 4.3 🔴 Prioriteit HOOG — Multi-type uitbreiding (NH90 → generiek)

**Probleem:** NH90-filter zit hardcoded in de SQL's. Bij uitbreiding naar andere types
moeten alle SQL's worden aangepast — foutgevoelig en onderhoudsgevoelig.

**Advies:**
- Voeg kolom `toesteltype` toe aan de query-tabel in PostgreSQL
- Pas de Python ETL aan zodat hij filtert op toesteltype als parameter
- Gebruik placeholder in SQL (bijv. `WHERE toesteltype = :type`)
- Keuze voor schema-strategie:
  - Optie A: aparte schemas per type (`03_ssot_nh90`, `03_ssot_cougar`)
  - Optie B: één SSOT met `toesteltype`-kolom, filter in mart-laag

---

### 4.4 🟡 Prioriteit MIDDEL — Laagconsistentie src → stage → ssot

**Probleem:** `SELECT *` in views breekt stil als een bronkolom verdwijnt of hernoemd
wordt in MLIC — geen fout, gewoon lege of verkeerde data stroomafwaarts.

**Advies:**
- Definieer per view expliciet de kolommen die je nodig hebt
- Voeg in de stage-laag eenvoudige validaties toe: NOT NULL checks, type-checks
- Documenteer per tabel de "contract"-kolommen (data dictionary)
- Overweeg een lichte validatiecheck die rijen telt en vergelijkt met vorige run

---

### 4.5 🟡 Prioriteit MIDDEL — Stored procedure beheer

**Advies:**
- Zet alle SP-definities als `.sql` bestanden in de GitLab repo
- Voeg versienummers toe aan SP's, log welke versie gebruikt is bij een run
- Overweeg **Flyway** of **Liquibase** voor gecontroleerde schema-wijzigingen
- Test SP's in een apart test-schema vóór productie

---

### 4.6 🟢 Prioriteit LAAG — Code structuur & onderhoudbaarheid

**Advies — splits het script in modules:**
```
etl/
├── config.py          # verbindingsstrings, parameters (uit env vars)
├── extract.py         # verbinding MLIC, query uitvoeren
├── load.py            # wegschrijven naar PostgreSQL
├── schema_manager.py  # stored procedure aanroepen
├── logger.py          # logging setup
└── run.py             # hoofdscript, orkestratie
```
- Gebruik environment variables voor credentials — nooit hardcoded in code of CI config
- Voeg type hints toe aan functies
- Schrijf een README: hoe werkt de pipeline, hoe test je lokaal

---

### 4.7 🟢 Prioriteit LAAG — Testen

**Advies:**
- Begin klein: pytest tests voor de transformatielogica
- Gebruik een test-schema in PostgreSQL voor integratietests (geen productiedata)
- Voeg een CI/CD stap toe die tests draait vóór de echte ETL-run
- Mock de Oracle-verbinding in unit tests — onafhankelijk van MLIC beschikbaarheid

---

## 5. Toekomstige architectuur (multi-type)

```
SAP
 └─► Oracle DB (MLIC)
      └─► GitLab CI/CD (DBDAAP)
           └─► Python ETL  ← leest query-tabel met toesteltype-parameter
                └─► PostgreSQL
                     ├─ 01_src           (alle types, ruw + 3MS extract)
                     ├─ 02_stage         (gevalideerde views, expliciete kolommen)
                     ├─ 03_ssot_<type>   (per toesteltype, of 1 ssot met type-kolom)
                     ├─ 04_mart          (aggregaties, KPI's voor rapportage)
                     └─ etl_run_log      (run-historie, monitoring)

3MS extract → 01_src  (los van SAP/MLIC stroom)
```

---

## 6. Open vragen

- [ ] Hoe wordt de `04_mart` laag gevuld — ook via views of Python transformaties?
- [ ] Multi-type: aparte schemas per type of één SSOT met type-kolom?
- [ ] Als een nachtrun faalt — wie krijgt een melding, wat is de maximale vertraging?
- [ ] Worden credentials nu veilig beheerd (niet hardcoded in script of CI config)?
- [ ] Draait het VB-script nu handmatig of via een planner (bijv. Windows Task Scheduler)?
- [ ] Hoe frequent wordt de 3MS extract vernieuwd in PostgreSQL?

---

## 7. Prioriteitenlijst

| Prio | Onderwerp                      | Impact | Haalbaarheid | Actie                              |
|------|--------------------------------|--------|--------------|------------------------------------|
| 1    | Logging & foutafhandeling      | Hoog   | Middel       | etl_run_log tabel + try/except     |
| 2    | Migratie script → CI/CD        | Hoog   | Middel       | Parallel draaien + go/no-go datum  |
| 3    | Multi-type generiek maken      | Hoog   | Middel       | Parameter in query-tabel           |
| 4    | SELECT * → expliciete kolommen | Middel | Laag         | Per view uitwerken                 |
| 5    | SP's in versiebeheer           | Middel | Hoog         | .sql files in repo + Flyway        |
| 6    | Code modularisering            | Middel | Hoog         | Splitsen in modules                |
| 7    | Testen                         | Middel | Middel       | pytest + test-schema               |

---

## 8. Versiehistorie

| Versie | Datum      | Wijziging                         |
|--------|------------|-----------------------------------|
| 0.1    | 15-03-2026 | Geconsolideerde versie — 3MS als databron toegevoegd, bestandsnamen bijgewerkt, versiehistorie gereset |
