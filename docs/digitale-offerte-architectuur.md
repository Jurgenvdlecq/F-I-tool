# Digitale occasionofferte met F&I — architectuur

Prototype: `digitale-offerte.html` (één bestand, geen backend). Openen in de browser is genoeg.

## Doelproces

```
Verkoper maakt offerte  →  PDF uploaden  →  gegevens uitlezen  →  controlescherm ("Klopt dit?")
   →  "Maak interactieve offerte"  →  unieke link naar klant  →  klant gebruikt pagina
   →  verkoper ziet interessesignalen
```

## Bouwstenen in het prototype en hoe ze later doorgroeien

| Onderdeel | Nu (prototype) | Pilot / later |
|---|---|---|
| **Offertedata** | `DEMO_OFFERTE`-object; eigen offertes via `?o=<token>` uit localStorage | `GET /api/offertes/{token}`; token is lang en niet te raden |
| **Opslag** | `opslag`-adapter rond localStorage | Zelfde interface, maar met API-calls |
| **Tracking** | `registreer(type, data)` schrijft **ruwe events** weg | `navigator.sendBeacon('/api/offertes/{token}/events', …)` |
| **Inzichten** | `berekenInzichten(events)` leidt score, niveaus en advies af | Dezelfde functie op de server; weging aanpasbaar zonder dataverlies |
| **Foto** | SVG-illustratie | `voertuig.fotoUrl` vullen; de pagina schakelt automatisch over |
| **PDF-offerte** | Placeholder in "Uw offerte" | `pdfUrl` vullen; wordt in een iframe getoond en via "Open PDF-offerte" geopend |
| **PDF uitlezen** | Gesimuleerd in de verkoperweergave (stap 1–3) | Lokaal met PDF.js + vaste veldherkenning per offertesjabloon (AVG: geen externe AI-API) |
| **Verzekering** | Gesimuleerde aanvraag met referentie | Koppeling met verzekeraar/volmacht of lead naar de binnendienst |
| **Inruil** | Kenteken via RDW open data (`m9d7-ebf2`), verwachte waarde telt indicatief mee | Taxatieverzoek naar de inkoop; getaxeerde waarde vooraf meegeven via `offerte.inruil` |
| **Financieringsaanvraag** | 4 stappen, voorgevuld met de berekening; gesimuleerd verstuurd, inkomensgegevens niet bewaard | Doorzetten naar de F&I-specialist of het portaal van de financier |
| **Digitaal akkoord** | Naam + handtekening (canvas) + keuze betaalwijze | Eenvoudige elektronische handtekening; juridisch laten toetsen; koopovereenkomst blijft via het bestaande proces |
| **Taal** | NL/EN-wissel; `offerte.taal` bepaalt de standaard | Taal meegeven vanuit het offertesysteem |
| **WhatsApp** | `wa.me`-link met voorgevuld bericht incl. berekening en inruil; `verkoper.whatsapp` | Zakelijk WhatsApp-nummer per verkoper of vestiging |

## Offertecontract (velden)

`nummer, datum, geldigTot, taal, klant{aanhef, voorletters, achternaam, email}, verkoper{naam, functie, vestiging, telefoon, whatsapp, email}, voertuig{merk, model, uitvoering, bouwjaar, kilometerstand, brandstof, transmissie, vermogenPk, kenteken, occasionnummer, kleur, carrosserie, fotoUrl}, prijs{verkoopprijs, inbegrepen[]}, usps[], financiering{rentePct, aanbetalingPct, looptijd, looptijden[], minPct, maxPct, slottermijn, minTeFinancieren}, inruil (optioneel, al getaxeerd), pdfUrl`

## Events

`pagina_geopend, tab_bekeken{tab, bron}, aanbetaling_gewijzigd{van, naar, bedrag}, looptijd_gewijzigd{van, naar}, financiering_interesse, contactvoorkeur, uitleg_bekeken, verzekering_gestart, verzekering_ingevuld, verzekering_bereken_onvolledig, verzekering_berekend, specificaties_bekeken, pdf_geopend, offerte_geprint, offerte_gedeeld, contact_geopend{modus}, contact_verstuurd{modus}, inruil_gestart, kenteken_opgezocht{gevonden}, inruil_toegevoegd, inruil_verwijderd, aanvraag_gestart{bron}, aanvraag_afgebroken{stap}, aanvraag_ingediend{soort}, akkoord_gestart, akkoord_gegeven{betaalwijze, verzekeringsvoorstel}, whatsapp_geklikt{bron}, taal_gewijzigd{taal}`

Elk financiering-event bevat een `stand` (aanbetaling %, €, looptijd, maandbedrag). Daarmee zijn de "laatste keuze" en de "laatste berekening" altijd te reconstrueren.

Sliderbewegingen worden gebundeld: één event per beweging (van → naar). Een sleepbeweging levert dus niet tientallen events op.

## Interessescore

Een gewogen som, omgerekend naar 0–100. De weging staat in `WEGING` en is ook zichtbaar in de verkoperweergave. Alleen een tab openen telt licht (maximaal 3 punten per onderdeel). Doorrekenen, een formulier afmaken of op een CTA klikken telt zwaar. Hierdoor levert één klik nooit "hoge interesse" op. Een verstuurde financieringsaanvraag geeft minimaal 60 punten, een digitaal akkoord minimaal 70.

De verkoperweergave is bewust geen CRM: alleen signalen en een advies voor de volgende stap, met de nadruk op F&I-kansen.

## Berekening

Annuïtair, zonder slottermijn: `M = P · r / (1 − (1 + r)^−n)`, met `r = 8,99% / 12`. Bij de standaardwaarden (25% aanbetaling, 60 maanden) is dat € 544,00 per maand. Het JKP wordt getoond als `(1 + r)^12 − 1`, met daarbij de verplichte zin "Let op! Geld lenen kost geld."
