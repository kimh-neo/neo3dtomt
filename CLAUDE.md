# Neo Tomt3D

Internt verktøy for Neo hytter: plasserer en hytte/et hus fra SketchUp (GLB) på riktig tomt i 3D,
med terreng, trær, veier, nabobygg, skjæring/fylling og solstudie. Brukes av fire personer på PC og Mac.

## Les dette først

- `docs/BYGGEOPPSKRIFT.md` – hele planen, steg 1–12. Følg rekkefølgen der.
- `docs/OPPSETT.md` – GitHub Pages, Actions og utviklingsmiljø (allerede satt opp).
- `referanse/forhandsvisning.html` – fungerende prototype i én fil (three.js 0.160). Mye av logikken
  kan flyttes rett over til TypeScript-modulene. Data og bilder er fjernet fra denne kopien.

## Arbeidsmåte

- Ta ett steg fra byggeoppskriften om gangen. Lag en egen gren og en pull request per steg.
- Avslutt hvert steg med at `npm run build` og `npm test` går grønt.
- Spør før du legger til nye biblioteker utover dem som står i steg 1.
- Skriv kode, kommentarer og grensesnitt på norsk bokmål. Variabelnavn kan være norske uten æøå
  (for eksempel `hoyde`, `paa`, `traer`).

## Faste regler

- All geometri i meter, lokalt koordinatsystem med origo i tomtens nullpunkt (EPSG:25833, UTM33).
  I three.js: x = øst, y = høyde, z = −nord.
- Ingen server og ingen proxy. Kartverket (api.kartverket.no, ws.geonorge.no) og Overpass svarer
  direkte fra nettleseren.
- Kundefiler (.skp, .glb, .pdf, prosjekt.json) skal ALDRI inn i repoet. Repoet er offentlig.
  `.gitignore` sperrer dem; ikke fjern de linjene.
- Ingen API-nøkler eller passord i koden.
- Vis «© Kartverket» og «© OpenStreetMap-bidragsytere» i verktøyet.
- Ikke bygg inn ortofoto (Norge i bilder) før lisensen er avklart.

## Fasit for tester (standardinnstillinger)

| Prosjekt | Tomteareal | Skjæring / fylling (massebalanse) | Avstand til grense |
| --- | --- | --- | --- |
| Trondheim 5001, gnr 513 bnr 24, Sikt hus, 270° | 1 078 m² | 113 / 109 m³ | 7,0 m |
| Stor-Elvdal 3423, gnr 54 bnr 465, Kjerne liten, 180° | 3 670 m² | 51 / 57 m³ | 19,8 m |

Standard: arbeidsrom 1,5 m, sokkel 0,35 m, helning normal (1:1,5 / 1:2), maks skråning 8 m.
Små avvik (noen få m³) er forventet når terrenget hentes på nytt eller interpolasjonen endres.

## Kommandoer

```bash
npm run dev      # lokal utvikling på http://localhost:5173/neo-tomt3d/
npm run build    # bygger til dist/
npm test         # Vitest-enhetstester
```
