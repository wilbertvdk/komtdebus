# Ontwerp: gesloten haltes in de reisinformatie

Status: ontwerp, nog niet gebouwd. De meetcode staat in een aparte repo; dit document beschrijft
wat daar moet komen en hoe `data.json` en `index.html` worden uitgebreid.

## De vraag

Een halte is weken dicht, maar staat nog in de dienstregeling, in de apps en op het scherm in de
bus. Reizigers plannen er een reis naartoe, of worden er opgehaald, en de bus stopt er niet.

De meting beantwoordt per gesloten halte, per dag:

> Bood de reisinformatie deze halte nog aan, en zo ja, stond er op tijd bij dat de bus er niet stopt?

Dit staat los van de bestaande ritmeting. Een rit die netjes rijdt maar een gesloten halte overslaat,
telt daar als "gereden". Deze meting kijkt per haltepassage.

## 1. Lijst met haltesluitingen

Een handmatig bijgehouden bestand `haltesluitingen.json` in deze (publieke) repo. Zo kan iedereen
zien op welke grond een halte als gesloten geldt. De meetcode leest het bestand.

```json
{
  "sluitingen": [
    {
      "id": "pr-soesterberg-2026",
      "halte": "Soesterberg, P+R Soesterberg",
      "stop_ids": [],
      "lijnen": ["202", "302"],
      "periodes": [
        {
          "van": "2026-08-10",
          "tot": "2026-08-28",
          "bron": "rws-2026-08"
        }
      ],
      "waarnemingen": [
        {
          "datum": "2026-10-22",
          "tijd": "20:55",
          "soort": "eigen waarneming",
          "wat": "P+R afgesloten; bus lijn 202 stopte niet bij de halte"
        }
      ],
      "bronnen": [
        {
          "id": "rws-2026-08",
          "soort": "officieel",
          "uitgever": "Rijkswaterstaat",
          "titel": "Rijkswaterstaat vernieuwt P+R Soesterberg vanaf 10 augustus",
          "url": "https://www.rijkswaterstaat.nl/nieuws/archief/2026/08/rijkswaterstaat-vernieuwt-p-r-soesterberg-vanaf-10-augustus-2026",
          "geraadpleegd": null,
          "archief": null
        }
      ]
    }
  ]
}
```

Regels:

- **Bronsoorten:** `officieel` (vervoerder, provincie, gemeente, wegbeheerder), `eigen waarneming`
  en `reizigersmelding`. De pagina toont de soort altijd bij de sluiting.
- **Een periode alleen met een officiële bron.** Een waarneming geldt alleen voor die datum en tijd.
  Uit één waarneming volgt geen begin- of einddatum.
- **Geen bron, geen sluiting.** Een sluiting zonder periode en zonder waarneming wordt niet gemeten.
- **`geraadpleegd` en `archief` worden ingevuld zodra iemand de bron zelf heeft gelezen.** Dat is
  een datum plus een kopie (zie 4). Een bron met `geraadpleegd: null` staat wel in de lijst, maar
  de pagina noemt hem "nog niet gecontroleerd".
- **`stop_ids`** zijn de halte-ID's uit `stops.txt` van de OVapi-GTFS. Een halte heeft er meestal
  één per richting. De meetcode zoekt ze eenmalig op naam op; daarna staan ze vast in het bestand.

## 2. Welke dagen tellen als gesloten

Een dag is een **meetdag** voor een sluiting als:

- de dag binnen een periode met een officiële bron valt, of
- er die dag een waarneming is.

Bij een waarneming tellen alleen de haltepassages rond het tijdstip mee (standaard: een uur ervoor
tot een uur erna). Bij een officiële periode tellen alle haltepassages van die dag mee.

## 3. Wat er per meetdag gemeten wordt

Er komen geen nieuwe bronnen bij. Alleen de bestaande feeds van OVapi worden ook per halte bekeken.

| Bron | Hoe vaak | Wat we eruit halen |
|---|---|---|
| Dienstregeling (GTFS) | elke dag | Alle geplande stops (`stop_times`) bij de `stop_ids`, per lijn en rit |
| Ritupdates (GTFS-RT) | elke 5 minuten | Per stop: staat er `schedule_relationship = SKIPPED`, en vanaf wanneer |
| Serviceberichten (GTFS-RT alerts) | elke 5 minuten | Berichten die de halte of de lijn noemen, met tekst en geldigheid |
| Voertuigposities | elke 2 minuten | Alleen ter controle: kwam de bus langs de halte |

Niet alle onderdelen zijn bevestigd. Zo is nog niet duidelijk of OVapi serviceberichten voor U-OV
publiceert. Dat moet de eerste stap van de bouw zijn.

### Indeling per geplande haltepassage

Elke geplande stop bij een gesloten halte komt in één groep, net als bij de ritten:

| Groep | Betekenis |
|---|---|
| **Uit de planning** | De halte staat niet in de dienstregeling van die dag. Dit is de goede situatie. Telt per dag, niet per passage. |
| **Overgeslagen, op tijd gemeld** | Staat in de planning, maar de ritupdate meldde de stop als overgeslagen vóór de geplande tijd. |
| **Overgeslagen, te laat gemeld** | De melding kwam pas ná de geplande tijd. |
| **Alleen een bericht** | Geen overgeslagen stop, wel een servicebericht over de halte. Planners plannen de reis nog gewoon; alleen wie het bericht leest, weet het. |
| **Niet gemeld** | Staat in de planning, geen overgeslagen stop en geen bericht. De halte bestaat voor de reiziger nog gewoon. |
| **Geen meting** | Te weinig ritupdates rond de geplande tijd. Telt niet mee. |

Een rit die als geheel is geannuleerd, telt hier niet mee. Die zit al in de ritmeting.

De norm uit de modelbijlage Datasets OV (95% op tijd gemeld, eis K17.2) gaat over vervallen
ritten. Of die ook voor vervallen haltes geldt, staat nog niet vast. Tot dat duidelijk is, toont de
pagina de percentages zonder norm.

### Wat het busscherm en de apps laten zien

Dat meten we niet rechtstreeks. Apps als 9292 en Google Maps gebruiken deze feeds, dus de feeds
zijn een goede benadering. Een screenshot van een app of het busscherm blijft bewijs per incident,
en gaat via het meldformulier.

## 4. Archief van meldingen

Aankondigingen verdwijnen zodra de sluiting voorbij is. Een melding van U-OV/Keolis over deze halte
gaf op 27 oktober 2026 al "pagina niet gevonden". Daarom bewaren we zelf wat er gemeld was.

- **Serviceberichten uit de feed:** elk bericht dat de halte of de lijn noemt, wordt bewaard met
  `eerst_gezien`, `laatst_gezien`, de ruwe inhoud en de tekst. Dit gebeurt automatisch, elke 5 minuten.
- **Dienstregeling:** per meetdag bewaren we de regels uit `stop_times` voor de gesloten halte.
  OVapi biedt alleen de actuele dienstregeling, dus zonder eigen kopie kan niemand later aantonen
  wat er gepland stond.
- **Webpagina's (U-OV, Keolis, provincie):** handmatig, als pdf of schermafbeelding met datum en
  SHA-256-hash. Het bestand gaat in het archief van de meetcode, de hash in `bronnen[].archief`.

Bewijs met persoonsgegevens, zoals berichten van reizigers, namen of adressen, komt nooit in deze
publieke repo. Daar staat alleen de waarneming zelf (datum, tijd, halte, wat er gebeurde).

## 5. Uitbreiding van `data.json`

Een nieuw blok `haltesluitingen`, naast `lijnen` en `dagen`. De bestaande velden blijven gelijk.

```json
"haltesluitingen": [
  {
    "id": "pr-soesterberg-2026",
    "halte": "Soesterberg, P+R Soesterberg",
    "lijnen": ["202", "302"],
    "grond": [
      { "soort": "officieel", "van": "2026-08-10", "tot": "2026-08-28", "titel": "…", "url": "…", "gecontroleerd": false },
      { "soort": "eigen waarneming", "datum": "2026-10-22", "tijd": "20:55", "wat": "…" }
    ],
    "meetdagen": 1,
    "dagen_uit_planning": 0,
    "passages": {
      "gepland": 14,
      "gemeten": 12,
      "aantal": {
        "overgeslagen, op tijd": 0,
        "overgeslagen, te laat": 0,
        "alleen bericht": 0,
        "niet gemeld": 12,
        "geen meting": 2
      },
      "percentage": {
        "overgeslagen, op tijd": 0.0,
        "overgeslagen, te laat": 0.0,
        "alleen bericht": 0.0,
        "niet gemeld": 100.0
      }
    },
    "berichten": [
      { "eerst_gezien": "…", "laatst_gezien": "…", "kop": "…", "tekst": "…", "geldig_van": "…", "geldig_tot": "…" }
    ]
  }
]
```

De cijfers in dit voorbeeld zijn verzonnen en dienen alleen om de structuur te laten zien.

Ontbreekt het blok of is het leeg, dan toont de pagina de sectie niet. Zo blijft de huidige
`data.json` geldig.

## 6. Uitbreiding van de pagina

- **Nieuwe sectie "Gesloten haltes"**, tussen "Resultaten per lijn" en "Per dag". Per sluiting een
  kaart met:
  - de halte en de lijnen;
  - de grond: de periode met bron, of de waarneming, met de bronsoort erbij ("officieel",
    "eigen waarneming", "nog niet gecontroleerd");
  - een balk met de haltepassages per groep, in dezelfde stijl als bij de ritten;
  - het aantal dagen dat de halte wel uit de planning was gehaald;
  - de bewaarde berichten, inklapbaar.
- **"Hoe we meten":** een alinea over haltes, met de groepen uit 3.
- **"Meld wat jij meemaakt":** een punt erbij: "de halte was dicht, maar stond wel in de app of op het scherm".
- **Lijn 202:** toevoegen aan `LIJN_NAMEN`, de titel en de introtekst zodra de lijn gemeten wordt.
  De lijnnaam haal je uit de GTFS (`routes.txt`), niet uit het geheugen.

## 7. Lijn 202

De halte P+R Soesterberg is het eerste geval, en lijn 202 stopt daar volgens eigen waarneming ook.
Het is daarom zinvol om lijn 202 mee te meten, zowel voor de ritten als voor de haltes.
De ritmeting werkt al per lijn, dus het gaat vooral om configuratie. Controleer in de GTFS welke
lijnen de halte bedienen, en neem die allemaal op in `lijnen` van de sluiting.

## 8. Het eerste geval: P+R Soesterberg

Wat bekend is, en hoe zeker:

| Wat | Bron | Status |
|---|---|---|
| P+R en bushaltes dicht van 10 t/m 28 augustus 2026 | Rijkswaterstaat | Gevonden via zoekmachine, nog niet zelf gelezen |
| Omleiding van 10 t/m 30 augustus, halte vervalt | U-OV | Gevonden via zoekmachine, nog niet zelf gelezen. Einddatum wijkt af van Rijkswaterstaat |
| P+R en fietspad dicht op 24 en 25 oktober 2026 | Rijkswaterstaat (volgens zoekresultaat) | Nog niet zelf gelezen |
| Halte vervalt van 10 oktober 07:00 t/m 26 oktober 05:00 (lijn 302, 315, 376) | U-OV/Keolis | Zonder jaartal, pagina op 27 oktober 2026 niet meer te vinden. **Niet opnemen tot bevestigd** |
| 22 oktober 2026, rond 20:55: P+R afgesloten, bus lijn 202 stopte niet | Eigen waarneming | Vastgelegd |

Alleen de waarneming van 22 oktober staat al vast. De officiële periode voor oktober volgt
hopelijk uit het antwoord van U-OV en de provincie op de klacht.

**Terugwerkend meten:** het meten begint pas zodra de code draait. Voor 22 oktober is er alleen een
kans als er ergens een archief van de dienstregeling en ritupdates van die dag bestaat, bijvoorbeeld
bij het NDOV Loket. Dat is nog niet uitgezocht.

## 9. Volgorde van bouwen

1. Controleren welke feeds OVapi heeft: zitten er serviceberichten voor U-OV in, en worden
   overgeslagen stops (`SKIPPED`) in de ritupdates gebruikt?
2. `stop_ids` voor P+R Soesterberg opzoeken en lijn 202 aan de meting toevoegen.
3. `haltesluitingen.json` aanmaken met de bronnen uit 8.
4. Archief: dagelijks `stop_times` voor gesloten haltes bewaren, en serviceberichten elke 5 minuten.
5. Indeling per haltepassage bouwen, met tests op vaste voorbeeldbestanden (per groep minstens één
   geval, plus een dag waarop de halte uit de planning is).
6. `data.json` uitbreiden (5) en de pagina aanpassen (6).

## 10. Open vragen

- Geldt de 95%-norm (K17.2) ook voor haltes die vervallen, of alleen voor ritten?
- Wat is de officiële sluitingsperiode in oktober, en geldt die ook voor lijn 202?
- Heeft OVapi of het NDOV Loket een archief van de dienstregeling van 22 oktober 2026?
