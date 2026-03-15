# NEXUS — Fleet Materiel Logistics BI Platform
## Domeincontext & Achtergrond
**Defensie Helikopter Commando**
Versie 2.0 | 15-03-2026

> **Doel van dit document:**
> Achtergrondkennis voor een nieuw LLM of nieuwe medewerker om snel de context
> te begrijpen zonder de volledige chathistorie te hoeven lezen.
> Voor het project zelf zie `NEXUS_ETL_v0.4.md` en `NEXUS_GUI_v0.4.md`.

---

## 1. Organisatie & context

**Defensie Helikopter Commando (DHC)**
Onderdeel van de Koninklijke Luchtmacht. Beheert en vliegt de NH90-helikoptervloot.

**PM — Program Management (LCW/Woensdrecht)**
Afdeling binnen het LCW, officieel onderdeel van "Air Depot en Program Management".
Voert de regie over de instandhouding van luchtwapensystemen voor de Koninklijke Luchtmacht.
PM bestaat uit aparte secties per wapensysteem. Voor dit project relevant:

| Sectie        | Status in project                                        |
|---------------|----------------------------------------------------------|
| PM NH90       | Huidige scope — primaire gebruiker van het platform      |
| PM Cougar     | Toekomstige scope                                        |
| PM Apache     | Toekomstige scope                                        |
| PM Chinook    | Toekomstige scope                                        |

> **Toekomstvisie:** het platform wordt nu gebouwd voor NH90, maar de architectuur
> wordt generiek opgezet zodat andere wapensystemen later kunnen worden toegevoegd.
> Dit sluit direct aan op de multi-type uitbreiding beschreven in `NEXUS_ETL_v0.4.md`.

PM NH90 — verantwoordelijk voor:
- Bewaken van systeem- en onderdelenbeschikbaarheid binnen instandhoudingsbudget
- Supply Chain Management (SCM): bevoorrading reserveonderdelen
- Regie over groot onderhoud en modificaties
- Afstemming met CAMO over onderhoudsstatussen en planning

**MLE-145 onderhoudsorganisaties (gebruikers van de data)**
MLE-145 is de Militaire Luchtvaarteis voor gecertificeerde onderhoudsorganisaties —
de Nederlandse militaire equivalent van EASA Part-145. Een MLE-145 goedkeuring van de MLA
is vereist om bevoegd onderhoud uit te voeren en vrijgiftes te tekenen op militaire luchtvaartuigen.

Er zijn twee -145 afdelingen die gebruik maken van de data uit het platform:

| Locatie | Type onderhoud                                              |
|---------|-------------------------------------------------------------|
| **MVKK De Kooy** | Lijnonderhoud NH90 — dagelijks, vluchtvóór/vluchtnà, kleine inspecties |
| **LCW Woensdrecht** | Hoger echelon onderhoud — grote inspecties, modificaties, componentrevisies |

Wat de -145 afdelingen nodig hebben uit het platform:
- Onderhoudsstatussen per toestel — wat vervalt wanneer
- AMP-taken en intervallen — planning werkpakketten en werkorders
- Configuratiedata — welke PNs/SNs gemonteerd, toegestane alternatieven
- Role equipment status — beschikbaarheid en montagelocatie
- Resturen/restcycles — prioritering en capaciteitsplanning

> De data uit het DHC data-platform (PostgreSQL) is directe werkinput voor
> zowel de lijn- als de hogere echelon onderhoudsorganisaties.

> **Samenvatting gebruikers / stakeholders:**
> - **CAMO DHC** (Gilze-Rijen + MVKK) — luchtwaardigheid individuele toestellen, uitvoeren SBPs
> - **CAMO PM** (LCW) — AMP/MIS, configuratiebeheer, reliability, uitbesteding (uitbesteed door CAMO DHC)
> - **-145 lijnonderhoud** (MVKK) — dagelijks onderhoud, vrijgiftes
> - **-145 hoger echelon** (LCW) — grote inspecties, modificaties
> - **PM NH90** (LCW) — instandhoudingsregie, budget, supply chain

**Het project — NEXUS**
Fleet Materiel Logistics BI Platform. Ontsluit SAP-logistieke data via een ETL pipeline
naar PostgreSQL, met gebruikersinterfaces en BI-rapportage voor onderhoudspersoneel,
CAMO, -145 organisaties en PM.
Doel: vlootbrede inzichtelijkheid van onderhoudsstatussen, configuratie en planning
voor alle helikoptertypes van DHC — nu NH90, later ook Cougar, Apache en Chinook.
Vervangt en verbetert de huidige situatie van losse Access-tools en handmatige processen.

---

## 2. NH90 vloot & operationele context

### Varianten Nederlandse NH90 vloot
| Variant | Aantal | Thuisbasis       | Primaire taak                              |
|---------|--------|------------------|--------------------------------------------|
| NFH (NATO Frigate Helicopter)  | 12 + 3 besteld (2030) | MVKK De Kooy (Den Helder) | Maritieme gevechtstaak, ASW, schepen        |
| TNFH (Tactical NFH)            | 8     | Gilze-Rijen (DHC)         | Transport, land en zee                      |

> Groot onderhoud voor beide varianten: Logistiek Centrum Woensdrecht.
> Lijnonderhoud: MVKK De Kooy en Gilze-Rijen.
> Binationale samenwerking met België via BNLC (Binationale Logistieke Cel NH90) op Woensdrecht.

### Operationele inzet
| Missie                  | Beschrijving                                                                 |
|-------------------------|------------------------------------------------------------------------------|
| ASW                     | Onderzeebootbestrijding met HELRAS sonar en Mark 46 torpedo's                |
| Drugsvangsten           | Caribisch gebied — vanaf stationsschepen (Zr.Ms. Groningen, Amsterdam etc.) NH90 onderschept smokkelbootjes, schakelt motoren uit met gericht vuur, boarding door mariniers |
| Boarding                | Afzetten gespecialiseerde mariniersteams op verdachte schepen                |
| SAR                     | Search and Rescue, ook kustwachttaak                                         |
| Transport               | Special forces, tot 14 passagiers of 6 liggende patiënten, Karel Doorman    |
| NAVO-operaties          | Internationale oefeningen en operaties                                       |

### Sensoren & role equipment (publiek bekend)
- HELRAS — Helicopter Long Range Active Sonar (onderzeebootdetectie)
- European Navy Radar — 360° oppervlakteradar
- FLIR — Forward Looking Infra-Red (nacht/slechtzicht)
- ESM — Electronic Support Measure
- Sonarboeien
- Mark 46 torpedo's (2 stuks)
- Machinegeweren
- Raketwaarschuwingssysteem (landoperaties)
- Sensor operator stations (1 of 2, verwijderbaar voor transporttaak)

> Dit zijn de publiek bekende role equipment items. De exacte SAP-configuratie en
> alle gemonteerde SNs worden bijgehouden in het DHC data-platform.

---

## 3. Technische infrastructuur

**SAP**
Bronsysteem voor alle materieel-logistieke data: vlieguren, onderhoudsstatus, stuklijsten,
serienummers, role equipment, magazijnvoorraad.

**SAP PO plannen (Preventief Onderhoud plannen)**
In SAP PM (Plant Maintenance) wordt preventief onderhoud beheerd via PO plannen
(Engels: Maintenance Plans). Een PO plan definieert welke onderhoudstaken op welk
interval worden uitgevoerd voor een specifiek technisch object.

**NH90 gebruikt twee typen PO plannen:**

| Type | SAP term | Wanneer | Voorbeeld NH90 |
|---|---|---|---|
| **Single Cycle Plan** | IP41 | Één interval, één dimensie | Kalenderinspectie elke 12 maanden |
| **Multiple Counter Plan** | — | Meerdere tellers tegelijk | Vlieguren + landingen + hoistlifts |

> **Strategy Plan wordt niet gebruikt bij NH90.** Dit SAP-type (IP42) is bedoeld voor
> meerdere intervallen op één object via een Maintenance Strategy. Bij de NH90 worden
> taken met verschillende intervallen als losse Single Cycle Plans opgezet.

**Structuur van een PO plan:**
```
PO plan (per equipment / NH90 serienummer)
  └─ Plan type              ← Single Cycle of Multiple Counter (geen Strategy Plan)
  └─ Taaklijst              ← gedeeld, beschrijft de uit te voeren taken (DMC-referentie)
  └─ Equipment              ← specifiek per toestel
  └─ Teller (counter)       ← vlieguren, landingen, hoistlifts etc.
  └─ Call horizon           ← bepaalt wanneer melding aangemaakt wordt (% of dagen vóór plan date)
       └─ Onderhoudsmelding ← automatisch gegenereerd in SAP op call date (vóór vervaldatum)
            └─ Order        ← handmatig aangemaakt vanuit de melding
```

**Call horizon / Lead Float — wanneer verschijnt de melding:**
De melding wordt in SAP **niet** aangemaakt op het moment van vervallen (plan date),
maar eerder — bepaald door een instelling in het PO plan:

| Plan type | Instelling | Eenheid |
|---|---|---|
| **Single Cycle Plan** | Call Horizon | % van cyclus of aantal dagen |
| **Multiple Counter Plan** | Lead Float (preliminary buffer) | Aantal dagen (absoluut) |

| Term | Betekenis |
|---|---|
| **Plan date** | Datum waarop het onderhoud uitgevoerd moet worden |
| **Call date** | Datum waarop SAP de melding aanmaakt — vóór de plan date |

Voorbeeld Single Cycle: cyclus 100 FH, call horizon 80% → melding verschijnt op 80 FH (20 FH vóór vervaldatum).
Voorbeeld Multiple Counter: lead float 14 dagen → melding verschijnt 14 dagen vóór vervaldatum.

> **SAP vs MLIC:** de melding verschijnt eerst in SAP. MLIC is een periodieke spiegel
> (2x per week — zo/di nacht). Wat NEXUS via MLIC ziet is altijd een momentopname
> met enige vertraging — de melding komt pas bij de volgende MLIC-refresh beschikbaar.

**Relatie AMP → SAP PO plannen:**
Het AMP is de echte master — goedgekeurd door de MLA en de autoriteitsbron voor
alle onderhoudstaken. Vanuit het AMP lopen twee aparte paden naar SAP:

```
AMP (Excel)  ←  echte master, goedgekeurd door MLA
  │
  ├─► 3MS (Defensie tool)
  │     ├─ PoRefs opgenomen uit AMP
  │     ├─ valideert of alle PO plannen correct zijn aangemaakt in SAP
  │     ├─ valideert of settings in PO plannen correct zijn
  │     └─► SAP PO plannen  ←  aangemaakt op basis van PoRef + PNs uit AMP
  │
  └─► Taaklijsten (direct uit AMP)
        ├─ DMC-taken gaan rechtstreeks van AMP naar SAP taaklijst
        ├─ 3MS raakt de DMC-taken niet
        └─ Taaklijst gekoppeld aan PO plan via Routegroup + Routegroupteller
```

**3MS** is een Defensie tool die de PoRefs uit het AMP beheert en als
validatiemechanisme fungeert tussen AMP en SAP. Het is geen master —
het AMP blijft de autoriteitsbron. 3MS controleert of SAP correct de
AMP weerspiegelt: of alle verwachte PO plannen bestaan en of de
instellingen (interval, taaklijst, tellers) correct zijn.

**Belangrijke kenmerken voor NEXUS:**
- Een PO plan wordt aangemaakt **per individueel equipment** — er bestaat **geen master
  PO plan** die uitgerold wordt naar meerdere NH90's. Elke heli heeft zijn eigen set PO plannen.
- Gedeelde elementen zijn mogelijk via **taaklijsten** (taakomschrijvingen),
  maar het plan zelf is altijd gekoppeld aan één specifiek toestel.
- SAP genereert automatisch een **onderhoudsmelding** wanneer een taak vervalt.
  Een order wordt vervolgens handmatig aangemaakt vanuit de melding.
- De **tellers** (vlieguren, landingen, hoistlifts) worden gevoed vanuit de vliegregistratie
  in SAP — dit is de basis voor de resturen/restcycles berekening.
- PO plan-statussen (resturen, restcycles, vervaldatum per taak) zijn de primaire bron
  voor het statusoverzicht en de simulatie in de NEXUS Maintenance Viewer.
- **Chapter 4 items** (Airworthiness Limitations) zijn aparte PO plannen met een harde
  limiet — SAP blokkeert in principe verdere inzet na overschrijding.
- **Chapter 5 items** (Scheduled Maintenance) hebben een tolerance-window —
  SAP berekent de volgende vervaldatum op basis van laatste uitvoering + interval.

> **NEXUS meerwaarde:** SAP geeft geen vlootbreed overzicht over alle PO plannen
> tegelijk. NEXUS trekt via de ETL alle toestel-specifieke PO plan-data samen in
> PostgreSQL, gecombineerd met 3MS-data voor de PoRef-structuur. Dit maakt
> vlootbrede statusoverzichten en simulaties mogelijk die in SAP zelf niet bestaan
> — en offline beschikbaar op laptop aan boord of out of base.

**MLIC (Oracle DB)**
Periodieke spiegel van SAP-data. Wordt bijgewerkt op zondag- en dinsdagnacht.
Staat los van SAP — is een read-only kopie voor rapportage- en analysedoeleinden.

**3MS (Defensie tool)**
Defensie tool die de PoRefs uit het AMP beheert en valideert of de SAP PO plannen
correct zijn aangemaakt en geconfigureerd. De 3MS data wordt als extract geladen
in PostgreSQL en is daarmee beschikbaar als databron voor NEXUS.

Wat 3MS levert aan NEXUS (via PostgreSQL):
- PoRef-lijst — de gestructureerde master van alle AMP-taken
- Koppeling PoRef → PO plannen — welke PO plannen bij welke AMP-taak horen
- PO plan settings — intervallen, taaklijstnummer, PNs

> NEXUS hoeft de PoRef-groepering niet zelf te reconstrueren uit de PO plan naam —
> die relatie ligt gestructureerd klaar in de 3MS-data in PostgreSQL.

**DBDAAP**
Intern Defensie platform waarop PostgreSQL draait en GitLab CI/CD wordt uitgevoerd.

**PostgreSQL laagstructuur**
```
01_src        Ruwe data zoals uit MLIC komt (tabellen)
              + ruwe data uit 3MS extract
02_stage      Eerste opschoning via views (grotendeels SELECT * FROM src)
03_ssot_nh90  Single Source of Truth voor NH90 — views, soms aangepast, NH90-filter
04_mart       Eindproduct voor rapportage en GUI-applicaties
etl_run_log   (nog te bouwen) run-historie en monitoring
```

**ETL werking**
- Een query-tabel in PostgreSQL bevat querynamen + bijbehorende SQL
- De ETL leest deze tabel, voert de SQL's uit op MLIC (Oracle) en laadt de resultaten in 01_src
- 3MS data extract wordt eveneens geladen in 01_src — naast de SAP/MLIC data
- Een stored procedure checkt of de doeltabel bestaat en maakt hem aan of past hem aan
- Momenteel: los Python script (handmatig of gepland gestart)
- In ontwikkeling: GitLab CI/CD op DBDAAP — verbinding werkt al, pipeline nog niet productierijp

---

## 3. Luchtwaardigheid — sleutelbegrippen

### MPD — Maintenance Planning Document
Document uitgegeven door NHIndustries dat alle onderhoudstaken
en intervallen beschrijft voor de NH90-vloot. Bij DHC wordt de MPD beschouwd als
**approved data** — aangeboden via de TC-houder (Type Management) aan de CAMO.

De MPD is **generiek**: van toepassing op de gehele wereldwijde NH90-vloot en bevat
taken voor alle mogelijke configuraties, modificatiestatussen en serienummers.
Het is geen operationeel document — het is de grondstof voor het AMP.

**Gezagslijn MPD → AMP → SAP:**
```
NHIndustries (OEM)
  └─► MPD  ← approved data, uitgegeven door industrie
        └─► Type Management (TC-houder)
              └─► aanbieden aan CAMO
                    └─► CAMO stelt AMP op (goedgekeurd door MLA)
                          └─► SAP PO plannen (per equipment / serienummer)
```

> **Nuance t.o.v. civiele luchtvaart:** in de civiele context wordt de MPD formeel
> als "niet-goedgekeurd" fabrikantsdocument beschouwd. Bij DHC/NH90 geldt de MPD
> als approved data binnen de militaire gezagslijn via Type Management.

### AMP — Aircraft Maintenance Programme
Het door de MLA goedgekeurde document dat alle geplande onderhoudstaken beschrijft
voor de NH90, opgesteld door de CAMO op basis van de MPD. Basis voor onderhoudsplanning
en de statusoverzichten in de Maintenance Viewer.

Het AMP is **toestel-specifiek**: de CAMO filtert de generieke MPD-taken op
modificatiestatus, serienummer en operationele relevantie. Wat overblijft is de
goedgekeurde takenlijst voor de DHC NH90-vloot, vertaald naar SAP PO plannen.

De NH90 AMP volgt de standaard ATA chapter-indeling:

| Chapter | Inhoud | Karakter |
|---------|--------|----------|
| **Chapter 4** | Airworthiness Limitations — Life Limited Parts (LLPs), harde limieten | **Nooit overschrijden** — safety-kritisch |
| **Chapter 5** | Time Limits / Scheduled Maintenance — geplande taken met intervallen (uren/cycli/kalender) | Toegestane variatie (tolerance) mogelijk |

> In de Maintenance Viewer worden chapter 4 en chapter 5 items visueel anders
> gemarkeerd vanwege het verschil in kritikaliteit.

### IETP — Interactive Electronic Technical Publication
De NH90 IETP is de digitale technische bibliotheek van NHIndustries,
opgebouwd volgens de **S1000D** standaard (Europese XML-standaard voor technische publicaties).

**Structuur:** elke Data Module (DM) is een zelfstandige XML-file, geïdentificeerd
via een unieke **Data Module Code (DMC)**. De DMC is de primaire sleutel.

Drie series zijn relevant voor NEXUS:

| Serie | DM type | Inhoud | Gebruik in NEXUS |
|-------|---------|--------|-----------------|
| **IPD** | Illustrated Parts Data | Stuklijsten — PNs en posities per assembly | Configuratie / PN-lijst |
| **AMP ch.4** | Airworthiness Limitations | Harde limieten, LLPs | AMP inspectieplanning |
| **AMP ch.5** | Scheduled Maintenance | Geplande taken met intervallen | AMP inspectieplanning |
| **DM (proced)** | Procedural | Onderhoudsprocedures / werkkaarten | DMC-referentie vanuit AMP-taken |

> **Relatie AMP → procedure:**
> Een AMP-taak verwijst via de DMC naar de bijbehorende onderhoudsprocedure
> die de technicus opzoekt in de IETP. NEXUS slaat alleen de DMC-referentie op —
> niet de procedure-inhoud zelf. De IETP is de authoritative bron voor procedures.

> **IPD vs IPC:** in S1000D heet de stuklijst IPD (Illustrated Parts Data),
> niet IPC (Illustrated Parts Catalog — dat is de ATA-term). Zelfde concept, andere naam.

### IPC — Illustrated Parts Catalog
Fabrikantdocument (NHIndustries) met de volledige stuklijst:
welke Part Numbers (PNs) horen op welke positie in de helikopter.
Bevat ook pre-modificatie PNs die na een uitgevoerde modificatie niet meer van toepassing zijn.
De IPC is dus niet direct bruikbaar als operationele lijst — moet worden gefilterd.

### Service Bulletin (SB)
Technisch document uitgegeven door de fabrikant/OEM. Beschrijft een wijziging, verbetering
of vervanging aan het vliegtuig of een component. Kan optioneel, aanbevolen of verplicht zijn.
Verplicht wanneer gekoppeld aan een Airworthiness Directive (AD).
Een SB kan een oud PN vervangen door een nieuw PN — na uitvoering is het pre-mod PN
niet meer toegestaan op dat specifieke toestel.

### AD — Airworthiness Directive / Luchtwaardigheidsaanwijzing
Juridisch bindende instructie van de luchtvaartautoriteit (voor Defensie: MLA).
Verplicht uitvoering van een specifieke actie (inspectie, modificatie, vervanging).
Vaak gebaseerd op een SB van de fabrikant.

### MLA — Militaire Luchtvaart Autoriteit
Onafhankelijke toezichtorganisatie binnen Defensie. Stelt militaire luchtvaartregelgeving op
namens de minister van Defensie en houdt toezicht op naleving.
Equivalent van EASA maar dan voor de militaire context.

### Type Management (TM)
Defensie-organisatie die de typeautoriteit beheert over een wapensysteem (in dit geval de NH90).
Verwerkt OEM-data (SBs, IPC-revisies, technische documentatie) en geeft formele goedkeuring
voor wijzigingen aan de configuratie, inclusief alternatieve of vervangende PNs.
Geen PN-wijziging zonder TM-goedkeuring gebaseerd op OEM approved data.

### CAMO — Continued Airworthiness Management Organisation
De CAMO-taken voor de NH90 zijn verdeeld over twee organisaties:

**CAMO DHC** (Gilze-Rijen + MVKK) — verantwoordelijk voor:
- Luchtwaardigheid van de individuele toestellen
- Uitvoeren van SBPs (Service Bulletin Procedures)
- Dagelijks onderhoud en vrijgiftes

**CAMO PM** (LCW/Woensdrecht) — taken uitbesteed door CAMO DHC:
- AMP (incl. MIS — Maintenance Inspection Schedule)
- Configuratiebeheer (incl. beoordelen/uitgeven SBPs, bijhouden modificatiestatus helis)
- Reliability
- Uitbesteding onderhoud
- Introductie nieuwe vliegtuigen

> CAMO PM beheert dus het AMP en de MIS (met de PoRefs), en is daarmee
> de primaire stakeholder voor alles wat met onderhoudsplanning en
> PoRef-structuur in NEXUS te maken heeft.

**Belangrijke nuance — PN-lijst beheer:**
De CAMO beheert de operationele PN-lijst, maar binnen strikte grenzen:
- Kan alleen PNs opnemen die door TM zijn goedgekeurd (op basis van OEM data)
- Kan de lijst inkorten: bijv. ongebruikte approved PNs weglaten, of pre-mod PNs verwijderen
  als alle toestellen al post-mod zijn
- Kan nooit zelfstandig een PN toevoegen buiten de TM-goedgekeurde set

**De gezagslijn voor een PN-wijziging:**
```
OEM approved data → Type Management goedkeuring → CAMO selectie → operationele PN-lijst
```

---

## 4. Configuratiebeheer

### SAP configuratiestaten — IPPE

SAP gebruikt de module **iPPE (Integrated Product and Process Engineering)** voor
configuratiebeheer van complexe producten met veel varianten, zoals de NH90.
Binnen SAP/iPPE worden twee configuratiestaten gebruikt, bij DHC bekend als **WC/TC**:

| Afkorting | Voluit | SAP term | Inhoud | In NEXUS |
|-----------|--------|----------|--------|----------|
| **WC** | **Werkelijke Configuratie** | As-Built | Wat er daadwerkelijk op de heli zit — gemonteerde PNs en SNs per positie, inclusief alle wijzigingen na modificaties en onderhoud | Scherm 4 Maintenance Viewer |
| **TC** | **Toegestane Configuratie** | As-Designed / PVS (Product Variant Structure) | Wat er op de heli mag — de CAMO-beheerde operationele PN-lijst, gebaseerd op TM/OEM goedgekeurde data | Scherm 4 Maintenance Viewer |

> **As-Maintained wordt niet gebruikt** — de werkelijke staat wordt in SAP
> bijgehouden als As-Built (WC), ook na onderhoud en modificaties.

### As-Built (werkelijke configuratie)
Elke NH90 heeft een eigen SAP equipment-structuur met alle gemonteerde componenten
en hun serienummers (SNs). Dit wordt As-Built bijgehouden in SAP en weerspiegelt
de actuele fysieke toestand van het toestel na alle in/uitbouwen en modificaties.

### As-Designed / PVS (toegestane configuratie)
De CAMO-beheerde operationele subset van door TM/OEM goedgekeurde PNs per positie,
vastgelegd als Product Variant Structure in SAP/iPPE.
Niet gelijk aan de IPD/IPC — gefilterd op:
- Actuele modificatiestatus (pre-mod PNs verwijderd als alle toestellen post-mod zijn)
- Operationele relevantie (niet alle approved PNs hoeven op de lijst)

### Role equipment
Missiespecifieke installaties die afhankelijk van de taak worden gemonteerd of verwijderd.
Voorbeelden: sonar-installatie (ASW), rescue hoist, extra brandstoftanks, wapensystemen.
In SAP geregistreerd als equipment-objecten die in/uitgebouwd kunnen worden.
Als niet gemonteerd: opgeslagen in voorraad als SAP equipment-object.
Toekomstig: eigen onderhoudsintervallen (resturen/restcycles) vanuit PostgreSQL.

### SAP PO plannen (Preventief Onderhoud plannen)

**PO plan** = SAP PM term voor **Maintenance Plan** — definieert wat er onderhouden
moet worden, hoe vaak (intervallen in vlieguren, cycli of kalenderdagen) en welke
taken worden gegenereerd. SAP maakt automatisch onderhoudsmeldingen/orders aan
als een taak vervalt.

**Belangrijke beperkingen van SAP PO plannen:**

1. **Geen master PO plan mogelijk**
   In SAP PM kun je geen master PO plan aanmaken die je uitrolt naar meerdere
   equipment-objecten. Elke NH90 heeft zijn eigen losse PO plannen.
   Taaklijsten kunnen wel gedeeld worden, maar het plan zelf is altijd
   gekoppeld aan één specifiek toestel.

2. **Geen bundeling van DMC's per PO plan**
   In het AMP kan één taak (bijv. 100 FH inspectie) meerdere DMC's bevatten —
   een bundel van procedures die samen bij dat interval horen.
   In SAP kan een PO plan maar één order/taak genereren — meerdere DMC's
   worden opgesplitst in losse PO plannen of via taaklijsten samengevoegd.

**Implicatie voor NEXUS:**
De mapping van AMP → SAP PO plannen is **niet 1-op-1**, maar via de **PoRef** te reconstrueren.

- Een PoRef in het AMP definieert één onderhoudstaak, met de bijbehorende DMC's, PNs en taaklijst.
- Vanuit één PoRef worden meerdere PO plannen aangemaakt — één per equipment (serienummer) op de genoemde PNs.
- De **PoRef naam is gelijk aan de PO plan naam**. Alle PO plannen die bij dezelfde AMP-taak horen hebben dus dezelfde naam, maar staan op verschillende equipment-objecten.
- NEXUS groepeert PO plannen naar AMP-taakniveau door op **PO plan naam** te groeperen — geen parsing of apart veld nodig.

Dit is mede waarom NEXUS waarde toevoegt — SAP geeft zelf geen vlootbreed
geaggregeerd overzicht van alle PO plan-statussen over alle toestellen heen.



## 5. Gebruikerstools — context

### Maintenance Viewer (PySide6 — actief in ontwikkeling)
Vervangt de bestaande MS Access versie. Repo: `RobMaas01/MV_PySide6`.
Stack: Python 3.12 · PySide6 · pandas · SQLite · openpyxl.

**Werkende tabs:**
- **Home** — statusbord XLS importeren, helikopter-selectie, work modes (Flight MVKK / Out of area 1-3 / BVP)
- **Overview** — resturen/restdagen/restcycles per heli, kalender/uren/cyclus-inspecties, bijzonderheden, export
- **Planning** — vliegprofiel invoeren voor een periode → welk onderhoud wordt geraakt, export
- **ECU** — ECU 1/2 status per toestel met serienummers

**Placeholder (tab bestaat, inhoud ontbreekt):**
- Configuration — WC/TC stuklijst per toestel
- MIS — AMP-taken met intervallen en vervaldatums
- Part. insp. — invulling nog te bepalen

**Databronnen:**
- Statusbord XLS — dagelijks handmatig uit SAP, importeren via Home-tab
- PostgreSQL — toekomst: directe koppeling bij opstarten (configuratie, MIS, 3MS)
- GLIMS XML — toekomst: na elke vlucht, gevlogen uren/landingen/motorcycles

**Offline-first:** app werkt volledig op lokale SQLite-database, ook zonder netwerk.
Multi-sessie synchronisatie via `last_change.txt` polling (2s) op gedeeld netwerkpad.
Zie `NEXUS_GUI.md` voor volledige technische details.

### MS Access tools (legacy)
Drie tools: Maintenance Viewer (wordt vervangen), Logkaart Tool, Logistieke Viewer.
Blokkade: 32-bit MS Office kan geen directe PostgreSQL-connectie maken.
Toekomst afhankelijk van organisatiebrede upgrade naar Windows 11 + 64-bit Office.
Bij 64-bit Office: PostgreSQL ODBC-connectie mogelijk → Access-tools kunnen live data tonen.

### Power BI & Excel Power Query
Rapportage-tools die data uit de 04_mart laag van PostgreSQL tonen.
Gebruikers kunnen zelf periodieke rapporten refreshen zonder technische kennis.

---

## 6. Startprompt voor nieuwe chat

Kopieer dit als eerste bericht in een nieuwe chat:

```
Ik werk verder aan NEXUS — Fleet Materiel Logistics BI Platform voor het Defensie Helikopter Commando (DHC).

Context:
- NH90 helikopter materieel logistiek (toekomst: ook Cougar, Apache, Chinook)
- ETL: SAP → Oracle MLIC (2x/week) + 3MS extract → Python script → PostgreSQL (schemas: 01_src, 02_stage, 03_ssot_nh90, 04_mart)
- CI/CD: GitLab op DBDAAP platform — in ontwikkeling, nog niet productierijp
- GUI: PySide6 Maintenance Viewer (Home/Overview/Planning/ECU werkend, Config/MIS placeholder), offline-first via SQLite, MS Access legacy tools (Logkaart Tool, Logistieke Viewer)
- Rapportage: Power BI + Excel Power Query
- Gebruikers: CAMO DHC (Gilze-Rijen + MVKK), CAMO PM (LCW), -145 lijnonderhoud (MVKK), -145 hoger echelon (LCW), PM NH90 (LCW)

Projectdocumenten:
- NEXUS_ETL.md — ETL pipeline status en advies
- NEXUS_GUI.md — GUI schermen, architectuur, werkende functionaliteit en roadmap
- NEXUS_CONTEXT.md — domeincontext achtergrond (dit document)
- NEXUS_MPD_naar_POplan.md — MPD → AMP → MIS → PoRef → PO plan → taaklijst → melding → order

We zijn bezig met: [ vul in wat je wil uitwerken ]
```

---

## 7. Versiehistorie

| Versie | Datum      | Wijziging                         |
|--------|------------|-----------------------------------|
| 0.1    | 15-03-2026 | Geconsolideerde versie — samengesteld uit chathistorie sessie 15-03-2026 |
| 2.1    | 15-03-2026 | Sectie 5 bijgewerkt o.b.v. broncode-analyse MV_PySide6 — werkende tabs, GLIMS, databronnen; startprompt bestandsnamen gecorrigeerd |
