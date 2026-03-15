# NEXUS — Van MPD naar PO plan en Taaklijst
## NH90 Onderhoudsplanning — SAP structuur
**Defensie Helikopter Commando**
Versie 1.0 | 15-03-2026

> **Doel van dit document:**
> Beschrijft de volledige keten van MPD naar SAP PO plan, taaklijst, melding en order
> voor de NH90-vloot. Achtergrondkennis voor NEXUS ETL en Maintenance Viewer ontwikkeling.
> Voor projectcontext zie `NEXUS_CONTEXT_v2.0.md`.

---

## 1. Gezagslijn — van MPD naar SAP

```
MPD (NHIndustries)
  └─► [via Type Management] CAMO PM stelt AMP op (goedgekeurd door MLA)
        └─► AMP  ←  echte master
              │
              ├─► MIS (Maintenance Inspection Schedule)
              │     └─► PoRefs — één per onderhoudstaak
              │
              ├─► 3MS (Defensie tool)
              │     ├─ beheert PoRefs uit AMP/MIS
              │     ├─ valideert of alle PO plannen correct zijn aangemaakt in SAP
              │     ├─ valideert of settings in PO plannen correct zijn
              │     └─► 3MS data extract → PostgreSQL (01_src)
              │
              └─► Taaklijsten (DMCs gaan direct van AMP naar SAP taaklijst)
                    ├─ 3MS raakt de DMC-taken niet
                    └─► SAP taaklijst (Routegroup + Routegroupteller)
```

---

## 2. AMP en MIS — structuur

Het AMP is het door de MLA goedgekeurde beheerdocument voor de NH90-vloot,
opgesteld door CAMO PM (LCW) op basis van de MPD. Het AMP bevat onder andere:
- Beleid, scope, procedures en revisiehistorie
- Verwijzingen naar brondocumenten (MPD, ADs, SBs)
- Levenlimieten (chapter 4 items)
- **MIS (Maintenance Inspection Schedule)** — het takenschema met alle PoRefs

### PoRef — de master boven de PO plannen

Een PoRef in de MIS groepeert alles wat bij één onderhoudstaak hoort:

| Gegeven | Beschrijving |
|---|---|
| Naam / omschrijving | Bijv. "100FH Engine inspectie" |
| DMC's | De taken die uitgevoerd moeten worden (bijv. 4 DMC-referenties) |
| PNs | De Part Numbers waarop PO plannen aangemaakt moeten worden |
| Taaklijstnummer | Routegroup + Routegroupteller |

> **PoRef naam = PO plan naam.** Alle PO plannen die vanuit dezelfde PoRef zijn
> aangemaakt hebben exact dezelfde naam, maar staan op verschillende
> equipment-objecten (serienummers). NEXUS groepeert op PO plan naam —
> geen parsing nodig.

---

## 3. Taaklijst — oplossing voor meerdere DMCs

Een SAP PO plan kan zelf niet meerdere taken/DMC's weergeven. Oplossing:
vanuit het AMP worden **SAP General Task Lists** aangemaakt die alle DMC's
van een PoRef bevatten als aparte operaties.

```
Taaklijst (SAP General Task List)
  Routegroup       (SAP: PLNNR) — groeperingsnummer
  Routegroupteller (SAP: PLNAL) — teller binnen de groep
  Operaties:
    0010  →  DMC-referentie taak 1
    0020  →  DMC-referentie taak 2
    0030  →  DMC-referentie taak 3
    0040  →  DMC-referentie taak 4
```

Het taaklijstnummer (Routegroup + Routegroupteller) staat in de PoRef in het AMP,
zodat bij het aanmaken van PO plannen het juiste taaklijstnummer meegegeven wordt.

---

## 4. Van AMP naar SAP — aanmaken PO plannen

Per PN in de PoRef worden PO plannen aangemaakt — één per individueel
equipment (serienummer) op dat PN.

**Voorbeeld:** PoRef "100FH Engine inspectie" met PN xxx1 en PN xxx2

```
PoRef / PO plan naam: "100FH Engine inspectie"
  ├─► PN xxx1 → equipment SN-A → PO plan "100FH Engine inspectie"
  ├─► PN xxx1 → equipment SN-B → PO plan "100FH Engine inspectie"
  └─► PN xxx2 → equipment SN-C → PO plan "100FH Engine inspectie"
```

### NH90 PO plan typen

| Type | SAP transactie | Interval | Call-instelling |
|---|---|---|---|
| **Single Cycle Plan** | IP41 | Één interval (tijd of prestatie) | Call Horizon (% of dagen) |
| **Multiple Counter Plan** | IP43 | Meerdere tellers tegelijk | Lead Float (dagen, absoluut) |

> Strategy Plan (IP42) wordt **niet** gebruikt bij de NH90.

---

## 5. Van PO plan naar melding — call horizon / lead float

De onderhoudsmelding wordt in SAP **niet** aangemaakt op de vervaldatum (plan date),
maar eerder — op de **call date**:

| Term | Betekenis |
|---|---|
| **Plan date** | Datum waarop het onderhoud uitgevoerd moet worden |
| **Call date** | Datum waarop SAP de melding aanmaakt — vóór de plan date |
| **Call Horizon** | Instelling Single Cycle Plan: % van cyclus of aantal dagen |
| **Lead Float** | Instelling Multiple Counter Plan: absoluut aantal dagen |

**Voorbeelden:**
- Single Cycle, cyclus 100 FH, call horizon 80% → melding op 80 FH (20 FH vóór vervaldatum)
- Multiple Counter, lead float 14 dagen → melding 14 dagen vóór vervaldatum

---

## 6. Van melding naar order

```
PO plan
  └─► op call date: SAP genereert automatisch Onderhoudsmelding
        - verschijnt in SAP op call date, niet op plan date
        - bevat taaklijstnummer (Routegroup + Routegroupteller)
        └─► Order (handmatig aangemaakt vanuit de melding)
              - taaklijst gekopieerd naar de order
              - operaties (DMC-taken) als aparte regels in de order
              - technicus voert taken uit en tekent per operatie af
```

> **SAP vs MLIC:** de melding verschijnt eerst in SAP. MLIC is een periodieke spiegel
> (2x per week — zo/di nacht). Wat NEXUS via MLIC ziet is altijd een momentopname
> met enige vertraging — de melding komt pas bij de volgende MLIC-refresh beschikbaar.

---

## 7. Volledig overzicht

```
MPD (NHIndustries)
  └─► [via TM] CAMO PM stelt AMP op (goedgekeurd door MLA)
        └─► AMP
              └─► MIS
                    └─► PoRef (= PO plan naam)
                          ├─ DMCs, PNs, taaklijstnummer
                          │
                          ├─► 3MS → valideert SAP PO plannen
                          │     └─► 3MS extract → PostgreSQL (01_src)
                          │
                          └─► SAP taaklijst (Routegroup + Routegroupteller)
                                └─► PO plan per equipment/SN
                                      │  Single Cycle  → call horizon
                                      │  Multiple Counter → lead float
                                      ▼ op call date
                                  Onderhoudsmelding (in SAP)
                                      └─► Order (handmatig)
                                            └─ Operaties: DMC taak 1..n
                                                  │
                                                  ▼
                                      SAP → MLIC (2x/week)
                                                  │
                                                  ▼
                                              PostgreSQL (01_src)
                                                  │
                                                  ▼
                                              NEXUS
                                      (combineert SAP + 3MS data)
```

---

## 8. NEXUS implicaties

**Groepering op PoRef-niveau**
PO plannen zijn in SAP individueel per equipment. PoRef naam = PO plan naam.
NEXUS groepeert via 3MS-data in PostgreSQL — PoRef-structuur ligt gestructureerd
klaar. Geen parsing van de PO plan naam nodig.

**Taaklijst als bron voor DMC-informatie**
De operaties in de taaklijst (Routegroup + Routegroupteller) bevatten de
DMC-referenties. NEXUS haalt via het taaklijstnummer in melding/order de
bijbehorende DMC-taken op.

**Mapping niet 1-op-1**
Één PoRef → meerdere PO plannen (per equipment/SN).
Één PO plan → één taaklijst → meerdere DMC-operaties.
NEXUS moet beide relaties kennen voor een correct vlootoverzicht.

**Call date vs plan date**
Meldingen in MLIC kunnen al aanwezig zijn voordat de plan date bereikt is.
NEXUS moet onderscheid maken tussen call date (aanmaakdatum melding) en
plan date (werkelijke vervaldatum).

---

## 9. Open punten

- [ ] SAP veldnaam / tabelnaam PO plan naam in MLIC (vermoedelijk WARPL of KTEXT)
- [ ] Type melding bevestigen — is dit inderdaad M4?
- [ ] Staan DMC-referenties als tekst in de taaklijst-operaties, of als apart veld?
- [ ] Welke call horizon / lead float waarden worden standaard gebruikt bij NH90 PO plannen?

---

## 10. Versiehistorie

| Versie | Datum | Wijziging |
|---|---|---|
| 1.0 | 15-03-2026 | Initieel — samengesteld uit chathistorie sessie 15-03-2026 |
