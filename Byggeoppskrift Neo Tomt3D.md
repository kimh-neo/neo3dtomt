# Byggeoppskrift: Neo Tomt3D

Oct 15, 2026 · @Kim

## Oversikt

Neo Tomt3D er en nettside som henter tomt, terreng, trær, veier og nabobygg fra åpne kilder, og plasserer huset fra SketchUp på tomta i 3D. Alt kjører i nettleseren. Det trengs ingen server, fordi Kartverket svarer direkte fra nettsider (testet 15.10.2026), og kundefilene leses lokalt fra Dropbox.

Oppskriften bygger på forhåndsvisningen vi allerede har laget, og følger samme rekkefølge som den: først data, så 3D-scenen, så husmodellen og beregningene, til slutt grensesnitt og publisering. Oppsett av GitHub, utviklingsmiljø og publisering står i det egne dokumentet «Oppsett: Neo Tomt3D på GitHub Pages», og forutsettes gjort.

Hvert steg nedenfor har tre deler: hva steget skal gjøre, hvilke filer det hører hjemme i, og koden eller algoritmen som gjør jobben. Kodebitene er TypeScript og bruker three.js 0.160.

&#91;embedded content: arkitektur · kilder, moduler i nettleseren, resultat\]

Dataene går gjennom fem moduler i nettleseren, fra henting til grensesnitt. Kundens filer leses fra og lagres tilbake til Dropbox-mappen, og ingenting sendes til en server.

## Steg 1: Prosjektstruktur og biblioteker

Start med Vite-prosjektet fra oppsettsdokumentet, og legg til disse bibliotekene:

```bash
npm install three@0.160.0 proj4 geotiff pdfjs-dist
npm install -D @types/three @types/proj4 typescript
```

Vi bruker three.js direkte i stedet for Giro3D. Forhåndsvisningen viser at det holder, og det gir full kontroll over terreng, skygger og skjæring/fylling.

Mappestruktur (hver fil har én jobb):

```text
src/
  main.ts                oppstart, kobler alt sammen
  config.ts              API-adresser, standardverdier
  geo/utm.ts             koordinatomregning (EUREF89 UTM33 ↔ grader)
  data/eiendom.ts        tomtegrense fra Kartverket
  data/hoyde.ts          DTM1/DOM1 fra Høydedata
  data/osm.ts            veier og bygg fra OpenStreetMap
  terreng/grid.ts        høydegitter, interpolasjon
  terreng/mesh.ts        3D-flate, normaler, shader med koter
  terreng/jordarbeid.ts  skjæring, fylling, støttemur, masser
  scene/vegetasjon.ts    trær (instanced meshes)
  scene/omgivelser.ts    veier, nabobygg, navnelapper
  scene/hus.ts           last inn GLB, glass, grunnmur
  scene/sol.ts           solposisjon, himmel, lys
  scene/render.ts        renderer, kamera, etterbehandling
  plan/pdfplan.ts        plantegning fra PDF på bakken
  prosjekt/schema.ts     prosjekt.json (type og validering)
  prosjekt/lagring.ts    åpne og lagre i Dropbox-mappen
  ui/panel.ts            stegene i venstre kolonne
  ui/kontroller.ts       glidebrytere, knapper, av/på
```

Felles regel: all geometri regnes i meter i et lokalt koordinatsystem med origo i tomtens senter (UTM33 avrundet). I three.js er x = øst, y = høyde og z = −nord.

## Steg 2: Finn tomta

Målet er å gå fra adresse eller kommune + gnr/bnr til en tomtegrense i UTM33 og et senterpunkt alt annet måles fra.

1. **Adresse → gnr/bnr:** Adresse-API-et (`https://ws.geonorge.no/adresser/v1/sok?sok=…`) gir kommunenummer, gnr, bnr og koordinat. Vis treffene i en liste, og la brukeren velge.
2. **Gnr/bnr → teig:** Eiendoms-API-et gir tomtegrensen som GeoJSON-polygon:

```ts
export async function hentTeig(kommune: string, gnr: number, bnr: number) {
  const url = `https://api.kartverket.no/eiendom/v1/geokoding?kommunenummer=${kommune}` +
    `&gardsnummer=${gnr}&bruksnummer=${bnr}&utkoordsys=25833&omrade=true`;
  const fc = await (await fetch(url)).json();
  const f = fc.features.find((x: any) => x.properties['hovedområde']) ?? fc.features[0];
  return {
    ring: f.geometry.coordinates[0] as [number, number][], // UTM33, meter
    noyaktighet: f.properties['nøyaktighetsklasseteig'],    // Grønt / Gult / Rødt
  };
}
```

3. **Nullpunkt:** Senter = tomtens tyngdepunkt, avrundet til hel meter. Lagre det i prosjektfilen. Alle andre data hentes i et kvadrat på 240 × 240 m rundt dette punktet.
4. **Kommunenummer:** Bruk 2024-nummereringen (for eksempel 3423 Stor-Elvdal, 5001 Trondheim). Hent listen fra `https://ws.geonorge.no/kommuneinfo/v1/kommuner` til en nedtrekksliste, så ingen må huske tallene.
5. **Ett koordinatsystem:** Bruk EPSG:25833 overalt, også i Trøndelag. Målestokkfeilen er under 0,05 % og har ingen praktisk betydning her.

Sjekk: Tomtearealet regnet fra ringen skal stemme med matrikkelen. Forhåndsvisningen ga 3 670 m² for 54/465 og 1 078 m² for 513/24.

## Steg 3: Terreng

Målet er en 3D-flate på 240 × 240 m med riktig høyde (NN2000), fine høydekurver og en flate som kan endres når vi planerer tomta.

**Hent høyder.** Høydedata-API-et tar inntil 50 punkter per kall. Et rutenett på 61 × 61 punkter (hver 4. meter) er 75 kall og tar et par sekunder med 6–8 kall samtidig:

```ts
const BASE = 'https://ws.geonorge.no/hoydedata/v1/punkt?koordsys=25833&geojson=false&punkter=';
export async function hentHoyder(pts: [number, number][], kilde: 'dtm1' | 'dom1' = 'dtm1') {
  const base = kilde === 'dom1' ? BASE.replace('/punkt?', '/datakilder/dom1/punkt?') : BASE;
  const z = new Float32Array(pts.length); let neste = 0;
  const arbeider = async () => {
    while (neste < pts.length) {
      const k = neste; neste += 50;
      const del = pts.slice(k, k + 50);
      const svar = await (await fetch(base + encodeURIComponent(JSON.stringify(del)))).json();
      svar.punkter.forEach((p: any, i: number) => (z[k + i] = p.z));
    }
  };
  await Promise.all(Array.from({ length: 8 }, arbeider));
  return z;
}
```

DTM-svaret har også feltet `terreng` (Skog, ÅpentOmråde …). Lagre det per punkt, og bruk det til å farge bakken.

**Senere oppgradering:** WCS-tjenesten for Nasjonal høydemodell kan levere hele utsnittet som én GeoTIFF med 1 m oppløsning. Den leses med `geotiff.js` og gir skarpere terreng. Sjekk den nøyaktige tjenesteadressen i Kartkatalogen før den tas i bruk.

**Bygg flaten.**

1. Glatt 4 m-gitteret opp til 0,5 m med bikubisk interpolasjon (Catmull–Rom). Det gir 481 × 481 punkter og myke skråninger.
2. Lag én `BufferGeometry` med posisjon, normal, UV og et ekstra attributt `aMod` (endring i terrenget og flagg for planert flate og støttemur, se steg 8).
3. Regn normaler direkte fra gitteret (finitte differanser). Det er raskere enn `computeVertexNormals()` og kan gjøres bare for området som endres.
4. Bakketeksturen (canvas 1536 px) males én gang: skogbunn med lyng og lav der terrengtypen er Skog, gress der den er ÅpentOmråde, og mørkere barnåler under hvert tre.
5. Koter tegnes i shaderen fra høyden, så de alltid er skarpe og følger endret terreng: tynn linje hver meter, tykk hver femte.

```glsl
float h = vWPos.y + H0;
float l1 = 1.0 - smoothstep(0.0, 1.3 * fwidth(h), abs(fract(h + 0.5) - 0.5));
float h5 = h / 5.0;
float l5 = 1.0 - smoothstep(0.0, 1.8 * fwidth(h5), abs(fract(h5 + 0.5) - 0.5));
diffuseColor.rgb *= 1.0 - 0.22 * l1 - 0.30 * l5;
```

6. Lag sidevegger ned til en felles bunn, så utsnittet ser ut som en modell på et bord.
7. Lag også en grov, usynlig kopi av terrenget (4 m-gitteret). Den brukes når man drar huset med musa, fordi treff mot 460 000 trekanter er for tregt.

## Steg 4: Vegetasjon (med av/på)

Trærne finnes ved å sammenligne overflatemodellen (DOM1, toppen av alt) med terrengmodellen (DTM1, bakken). Der forskjellen er stor, står det noe høyt.

1. Hent DTM1 og DOM1 i et rutenett hver 2. meter (121 × 121 punkter, ca. 300 kall per modell).
2. Regn ut høyde over bakken: `d = DOM − DTM` for hvert punkt.
3. Et tre er et punkt der `d > 3 m` og `d` er større enn alle de åtte nabopunktene (lokalt toppunkt). Trehøyden er `d`.
4. Fjern «trær» som egentlig er bygg eller kjøretøy: alt innenfor 2 m fra et bygningsomriss (steg 5), og alt innenfor veibredden + 1,2 m.
5. Velg treslag: i forhåndsvisningen er omtrent 30 % gran (lagdelte kjegler) og resten furu (høy stamme, rund krone). Bruk et fast tilfeldig frø, så skogen ser lik ut hver gang.
6. Tegn alle trær med tre `InstancedMesh` (stamme, furukrone, grankrone) og én tegneoperasjon hver. Sett `frustumCulled = false`, ellers forsvinner trær i kanten av bildet.
7. Skjul et enkelt tre (for eksempel når det må felles) ved å sette matrisen til skala 0.

**Av/på-knapp.** All vegetasjon skjules og vises med én linje:

```ts
export function visVegetasjon(på: boolean) {
  [stammer, furu, gran].forEach(m => (m.visible = på));
  requestRender();
}
```

Tellingen av trær som må felles fortsetter selv om trærne er skjult. Et tre må felles hvis det står innenfor 3 m fra huset, eller der terrenget endres mer enn 10 cm (steg 8).

Begrensning: DOM1 kan være noen år gammel. Hogst og nye trær etter flyfotograferingen vises ikke.

## Steg 5: Veier og nabobygg (med av/på)

Kartverkets detaljerte bygningsdata (FKB) krever avtale. OpenStreetMap har veier og bygningsomriss for Norge, og byggene har ofte bygningsnummer fra matrikkelen (`ref:bygningsnr`).

**Hent data.** Én spørring mot Overpass-API-et henter alt i et område rundt tomta, i grader:

```ts
const q = `[out:json][timeout:25];(way[highway](${s},${w},${n},${e});way[building](${s},${w},${n},${e}););out geom;`;
const svar = await fetch('https://overpass-api.de/api/interpreter', {
  method: 'POST', body: 'data=' + encodeURIComponent(q),
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
});
```

Overpass er ofte opptatt (svarer 502). Prøv på nytt opptil seks ganger med 3, 6, 9 … sekunders pause. Mellomlagre svaret i prosjektfilen, så et prosjekt ikke trenger nettet for å åpnes igjen.

**Regn om koordinater.** Gjør om fra grader til UTM33 med proj4, og trekk fra nullpunktet:

```ts
import proj4 from 'proj4';
proj4.defs('EPSG:25833', '+proj=utm +zone=33 +ellps=GRS80 +units=m +no_defs');
const [x, y] = proj4('EPSG:4326', 'EPSG:25833', [lon, lat]);
```

Klipp veilinjene mot kanten av utsnittet (Liang–Barsky), så de ikke stikker ut av modellen.

**Veier.** Tegn hver vei som et bånd som følger terrenget: punkt for hver meter, høyden glattet over ±3 m (en vei er planert), og en litt bredere, mørkere kant under. Bredde etter type: hovedvei 5–6 m, `service` 4 m, sti 1,1 m. Vis veinavn som en HTML-lapp på punktet nærmest tomta, én lapp per navn.

**Nabobygg.**

1. Byggehøyde: mål DOM1 hver meter innenfor omrisset, og DTM1 langs omrisset. Høyde = høyeste DOM − laveste DTM, men minst 3,8 m for bolig og hytte og 2,8 m for uthus.
2. Form: vegger rett opp fra omrisset, og et enkelt saltak (27°) med mønet langs den lengste veggen. Takformen er ikke målt, og det skal stå i grensesnittet.
3. Bygg med tyngdepunkt inne på egen tomt merkes som eksisterende. De er skjult som standard, med en egen Vis-knapp, og teller ikke i avstandskravet når de er skjult.

**Av/på-knapp.** Nabobygg og navnelappene deres skjules sammen. Veier vises alltid, fordi de sier noe om adkomst. Avstandskontrollen (steg 7) regnes uansett.

```ts
export function visNabobygg(på: boolean) {
  nabobygg.filter(b => !b.egen).forEach(b => b.meshes.forEach(m => (m.visible = på)));
  lapper.filter(l => l.bygg && !l.bygg.egen).forEach(l => (l.el.hidden = !på));
  requestRender();
}
```

## Steg 6: Husmodellen fra SketchUp

I forhåndsvisningen bygde vi husene for hånd fra PDF-tegningene. I verktøyet lastes den ekte modellen inn som GLB, og da får vi riktig form uten manuelt arbeid.

**Regler for eksporten fra SketchUp (felles for alle fire):**

1. Origo (der aksene møtes) legges midt på huset i plan, med z = 0 i ferdig gulv.
2. Langsiden ligger langs den røde aksen. Grønn akse er «oppover i planen».
3. Terreng, mennesker, biler og hjelpegeometri slettes eller skjules. Vindusruter får et materiale med «glass» i navnet.
4. Fjern ubrukte komponenter og materialer. Fil → Eksporter → 3D-modell → GLTF Binary (\*.glb).

SketchUp er Z-opp, glTF er Y-opp, og eksporten snur aksene automatisk. Rød akse blir three.js x, og grønn akse blir −z. Det er samme lokale system som i forhåndsvisningen, så plassering og rotasjon virker uendret.

**Last inn filen** (dra og slipp eller filvelger, filen forlater aldri maskinen):

```ts
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
export async function lastHus(fil: File) {
  const buf = await fil.arrayBuffer();
  const gltf = await new GLTFLoader().parseAsync(buf, '');
  const hus = gltf.scene;
  hus.traverse((o: any) => {
    if (!o.isMesh) return;
    o.castShadow = o.receiveShadow = true;
    const navn = (o.material?.name ?? '').toLowerCase();
    const gjennomsiktig = o.material?.transparent || o.material?.opacity < 1;
    if (gjennomsiktig || /glass|glas|vindu|window/.test(navn)) o.material = glassMat;
  });
  return hus;
}
```

**Lyse, gjennomsiktige vinduer.** Ett felles glassmateriale gir lyse ruter med litt refleks av himmelen, og man ser inn i rommet bak:

```ts
export const glassMat = new THREE.MeshPhysicalMaterial({
  color: 0xe6eff3, roughness: 0.04, metalness: 0,
  transparent: true, opacity: 0.32, depthWrite: false,
  clearcoat: 1, clearcoatRoughness: 0.05, envMapIntensity: 1.2,
});
```

Har modellen ikke innvendige vegger, legg inn en enkel lys «innvendig kasse» (varm grå, `0xd9d2c6`), så man ikke ser rett gjennom huset.

**Fotavtrykk.** Kontrollene i steg 7 og planeringen i steg 8 trenger husets omriss som rektangler i lokale koordinater:

- Standard: én boks rundt modellen (`new THREE.Box3().setFromObject(hus)`), målt ved takfoten.
- For hus med tilbygg (som Sikt hus): flere rektangler i `prosjekt.json` under `modell.fotavtrykk`, for eksempel hovedhus og grovinngang hver for seg.

**Grunnmur.** Tegnes for hvert rektangel: en vegg fra gulvnivå og ned til terrenget minus 25 cm, rundt hele omkretsen. Ved skjæring/fylling blir den automatisk lav, fordi terrenget ligger rett under gulvet.

Hold filen under ca. 50 MB. Store teksturer gjør verktøyet tregt på eldre maskiner.

## Steg 7: Plassering og kontroller

Husets plassering er fire tall: øst, nord, retning og gulvhøyde. Alt annet regnes ut fra dem hver gang noe endres.

**Retning.** Brukeren velger hvilken himmelretning hovedfasaden (stua) vender mot, i grader (180 = sør). Rotasjonen i scenen er `θ = π − retning`. Gjør om mellom lokale og globale koordinater med:

```ts
const tilVerden = (lx: number, ly: number) =>
  [E + lx * Math.cos(θ) - ly * Math.sin(θ), N + lx * Math.sin(θ) + ly * Math.cos(θ)];
```

**Dra huset.** Ved museklikk på huset: slå av kamerakontrollen, finn treffpunktet på den grove terrengflaten (steg 3), og flytt huset dit, med samme forskyvning som der man tok tak. Oppdater høyst én gang per skjermbilde (`requestAnimationFrame`), ellers blir det hakkete.

**Følg kotene.** Regn ut terrengets helning der huset står, og snu huset slik at hovedfasaden vender nedover bakken. På Sandsæter gikk fallet under huset fra 1,9 m til 0,9 m med dette.

**Gulvhøyde.** Tre regler som brukeren velger mellom:

- 30 cm over høyeste terreng under huset (vanlig ved pilarer og grunnmur).
- Massebalanse, der det graves bort omtrent like mye som det fylles (steg 8).
- Fast kote, satt med glidebryteren. Ved flytting beholdes høyden over terrenget.

**Kontroller som vises med grønt, gult eller rødt:**

| Kontroll | Hvordan den regnes | Grønn når |
| --- | --- | --- |
| Innenfor tomta | Alle hjørner av fotavtrykket ligger inne i teigen (punkt-i-polygon) | Ja |
| Avstand til tomtegrense | Korteste avstand fra hjørnene til teigens kanter | ≥ 4 m |
| Avstand til nabobygg | Korteste avstand mellom fotavtrykket og nabobyggenes omriss | ≥ 8 m |
| Avstand til vei | Korteste avstand fra fotavtrykket til veiens senterlinje | ≥ 4 m |
| Trær som må felles | Trær innen 3 m fra huset eller der terrenget endres | færre enn 15 |
| Grunnmur / pilarer | Gulvhøyde − laveste terreng under huset | ≤ 2,5 m |

Avstandskravene er tommelfingerregler (plan- og bygningsloven § 29-4 og brannkravet i TEK17). Reguleringsplanen for området kan ha andre krav, og det må stå i verktøyet.

## Steg 8: Skjæring, fylling og støttemur

Ved «Planert tomt» lages en flat flate (planum) under og rundt huset, med skråninger ut til eksisterende terreng. Beregningen gjøres på 0,5 m-rutenettet fra steg 3.

**Parametere** (alle i prosjektfilen):

- Planum = gulvhøyde − sokkel (standard 0,35 m).
- Arbeidsrom rundt huset, m (standard 1,5).
- Helning skjæring / fylling: bratt 1:1 / 1:1,5, normal 1:1,5 / 1:2, slak 1:2 / 1:3.
- Maks skråningsbredde før støttemur, m (standard 8).

**Ny høyde for hvert punkt.** La d være avstanden fra punktet til fotavtrykket utvidet med arbeidsrommet, z₀ terrenget, P planum, og c og f horisontal lengde per meter høyde i skjæring og fylling:

```latex
z = \begin{cases} P & d = 0 \\ \min\left(\max\left(z_0,\; P - \frac{d}{f}\right),\; P + \frac{d}{c}\right) & 0 < d \le L_{maks} \\ z_0 & d > L_{maks} \end{cases}
```

**Bare sammenhengende skråninger.** Formelen alene «fyller» også daler langt unna, der skråningen tilfeldigvis ligger over terrenget. På 513/24 ga det 2 330 m³ i stedet for omtrent 110 m³. Løsningen er et bredde-først-søk fra planumet og utover. Et punkt telles bare hvis det henger sammen med planumet gjennom punkter som også endres, og avstanden øker utover:

```ts
// kø starter med alle punkter der d === 0
while (kø.length) {
  const k = kø.shift()!;
  for (const nb of naboer4(k)) {
    if (sett[nb]) continue;
    if (Math.abs(dz[nb]) > 0.005 && d[nb] >= d[k]) { sett[nb] = 1; kø.push(nb); }
  }
}
// punkter som ikke er sett: dz = 0
```

**Støttemur.** Der skråningen fortsatt ikke har møtt terrenget ved Lₘₐₖₛ, stopper den med en vertikal kant. Det er støttemuren. Punktene i ringen like innenfor Lₘₐₖₛ med høydeforskjell over 0,3 m gir lengden (antall × 0,5 m, et overslag) og største høyde. De merkes i `aMod`, så shaderen farger dem som betong.

**Masser.** Summer for hvert punkt `dz × 0,25 m²`: negative verdier er skjæring, positive er fylling. Vis også høyeste skjæring og fylling, berørt areal og støttemur.

**Massebalanse.** Netto (fylling − skjæring) vokser jevnt med gulvhøyden. Finn gulvet der netto er null med halveringssøk: 34 runder mellom laveste terreng − 3 m og høyeste + 3 m, og rund av til nærmeste 5 cm. Hver runde kjører beregningen uten å tegne, så det tar noen millisekunder.

**Oppdater skjermen.** Tilbakestill forrige område til originalt terreng, regn det nye, og send bare de berørte radene til grafikkortet (`attribute.addUpdateRange`). Regn normaler bare der. Da kan huset dras rundt i sanntid.

**Farger.** To visninger i shaderen: «Naturlig» (grus på planum, brun jord i skråningene, grå støttemur) og «Skjæring/fylling» (rød = gravd bort, blå = fylt på, sterkere farge jo dypere, full farge ved 1,5 m).

Vis tydelig at dette er et overslag fra terrengmodellen, ikke en prosjektert masseberegning.

## Steg 9: Plantegning fra PDF på bakken

Plantegningen lagt rett på terrenget viser kunden hvor huset kommer, før det er bygget i 3D. Den brukes også til å sjekke at GLB-en og tegningen stemmer.

1. **Les PDF-en** med pdf.js, og tegn valgt side (vanligvis A.01) til et canvas med 200 dpi. Ved 1:100 blir det 78,74 piksler per meter.

```ts
import * as pdfjs from 'pdfjs-dist';
pdfjs.GlobalWorkerOptions.workerSrc = new URL('pdfjs-dist/build/pdf.worker.min.mjs', import.meta.url).href;
const doc = await pdfjs.getDocument({ data: await fil.arrayBuffer() }).promise;
const side = await doc.getPage(1);
const vp = side.getViewport({ scale: 200 / 72 });
const c = Object.assign(document.createElement('canvas'), { width: vp.width, height: vp.height });
await side.render({ canvasContext: c.getContext('2d')!, viewport: vp }).promise;
```

2. **Gjør papiret gjennomsiktig.** For hver piksel: alfa = (250 − lysstyrke) / 170, avgrenset til 0–1, og farge = mørk blæk. Da vises bare strekene på bakken.
3. **Plasser tegningen** med to punkter: brukeren klikker to hjørner i PDF-en (for eksempel ytterhjørnene på langveggen) og de samme to hjørnene på huset. To punkter gir flytting, rotasjon og målestokk. Lagre punktene i `prosjekt.json` under `pdf.punkter`.
4. **Drapere på terrenget.** Lag et plan med 64 × 28 ruter i tegningens størrelse, og sett høyden i hvert hjørnepunkt til terrenget + 5 cm. Bruk `depthWrite: false` og `polygonOffset`, så tegningen ikke flimrer mot bakken. Flytt og roter den sammen med huset.
5. **Visning:** tre valg, «3D-modell», «Plan på bakken» og «Begge». I plan-modus kan man dra i tegningen for å flytte huset.

Samme teknikk virker for en situasjonsplan: da plasseres den med to punkter med kjente koordinater i stedet for husets hjørner.

## Steg 10: Sol, lys og bildekvalitet

**Solposisjon.** Regn ut solhøyde og retning for tomtens bredde- og lengdegrad og valgt dato og klokkeslett. En forenklet astronomisk formel holder til solstudier (feil under ca. 1 grad):

```ts
export function solposisjon(utcMs: number, lat: number, lon: number) {
  const d = utcMs / 864e5 + 2440587.5 - 2451545.0, rad = Math.PI / 180;
  const g = (357.529 + 0.98560028 * d) * rad, q = 280.459 + 0.98564736 * d;
  const L = (q + 1.915 * Math.sin(g) + 0.020 * Math.sin(2 * g)) * rad, e = (23.439 - 3.6e-7 * d) * rad;
  const ra = Math.atan2(Math.cos(e) * Math.sin(L), Math.cos(L)), dek = Math.asin(Math.sin(e) * Math.sin(L));
  const H = ((18.697374558 + 24.06570982441908 * d) % 24) * 15 * rad + lon * rad - ra, f = lat * rad;
  const hoyde = Math.asin(Math.sin(f) * Math.sin(dek) + Math.cos(f) * Math.cos(dek) * Math.cos(H));
  const asimut = Math.atan2(-Math.sin(H), Math.tan(dek) * Math.cos(f) - Math.sin(f) * Math.cos(H));
  return { hoyde, asimut: (asimut + 2 * Math.PI) % (2 * Math.PI) };
}
```

Klokkeslettet er norsk tid. Bruk `Intl.DateTimeFormat` med `timeZone: 'Europe/Oslo'` for sommertid, i stedet for faste datoer. Finn soloppgang og solnedgang ved å sjekke solhøyden hvert 5. minutt gjennom døgnet. Vis at fjell lenger unna enn utsnittet ikke er med.

**Lys.**

- Sol: `DirectionalLight` med skygger, plassert 180 m ut i solretningen og rettet mot huset. Skyggekameraet dekker ±55 m rundt huset. Skyggekart 4096 px på store skjermer og 2048 på mobil.
- Himmel: en enkel shader på en stor kule, med farge som går fra horisont til senit og en solskive. Fargene følger solhøyden: blått om dagen, oransje ved lav sol, mørkt om natten.
- Miljølys: lag et miljøkart fra himmelen med `PMREMGenerator.fromScene()` hver gang sola flyttes. Det gir naturlig fyllys og refleks i vinduene.

**Bildekvalitet.**

- Materialer: `MeshStandardMaterial` overalt, fasadebilder som tekstur, glass som i steg 6.
- Tonekurve: `ACESFilmicToneMapping` med eksponering 0,92. Farger som skrives i shaderen må være i lineært fargerom (sRGB 110 tilsvarer lineært ca. 0,15).
- Etterbehandling: `EffectComposer` med `RenderPass`, `GTAOPass` (skygge i hjørner og under trær) og `OutputPass`, og et render-target med `samples: 4` for kantutjevning.
- Knappen «Høy kvalitet» slår av etterbehandlingen på svake maskiner.
- Tegn bare et nytt bilde når noe har endret seg (kamera, sol, plassering). Da bruker verktøyet nesten ikke strøm når det står i ro.

**Kameravisninger:** perspektiv (fra hjørnet med terrassen), rett ovenfra, og «Utsikt fra stua» (øyehøyde 1,65 m like utenfor stuevinduet, rettet utover). Et kompass i hjørnet roterer med kameraet.

## Steg 11: Brukergrensesnitt og prosjektfil

**Oppsett av skjermen.** Samme som forhåndsvisningen: arbeidsflyten i en kolonne til venstre (Eiendom, Terreng, Modell, Plassering, Terrenginngrep, Sol), 3D-vinduet til høyre. På mobil ligger 3D-vinduet øverst og panelet under.

Knapper nederst i 3D-vinduet:

- **Trær** og **Nabobygg**: av/på (steg 4 og 5).
- **Koter** og **Høy kvalitet**: av/på.
- **Perspektiv**, **Ovenfra**, **Utsikt fra stua**: kameravisninger.

Siste tilstand av av/på-knappene kan huskes i `localStorage` per bruker. Selve prosjektdataene lagres aldri der.

**Prosjektfil.** Alt som trengs for å åpne et prosjekt igjen ligger i `prosjekt.json` i kundens mappe `1. Tegninger/Terreng/`. Data fra Kartverket og OpenStreetMap mellomlagres i filen med dato, så prosjektet ser likt ut neste gang og kan åpnes uten nett:

```json
{
  "versjon": 2,
  "kunde": "Auran – Fyhn",
  "eiendom": { "kommunenummer": "5001", "gnr": 513, "bnr": 24 },
  "nullpunkt": { "ost": 275203, "nord": 7029358, "koordsys": "EPSG:25833" },
  "modell": {
    "fil": "modell.glb",
    "fotavtrykk": [[-9.13, 9.13, -4.70, 4.70], [-1.89, 5.37, 4.59, 7.40]]
  },
  "plassering": { "ost": -0.5, "nord": -1.5, "retning": 270, "gulv_moh": 192.60 },
  "terrenginngrep": { "modus": "planert", "arbeidsrom": 1.5, "sokkel": 0.35, "helning": "normal", "maks_skraaning": 8 },
  "pdf": { "fil": "plan.pdf", "side": 1, "punkter": [] },
  "visning": { "traer": true, "nabobygg": true, "vis_eksisterende": false },
  "data": { "hentet": "2026-10-15", "teig": [], "dtm4": "…", "traer": "…", "osm": {} }
}
```

Valider filen med en enkel typesjekk når den åpnes, og øk `versjon` når formatet endres, slik at gamle filer kan oppgraderes.

**Åpne og lagre.**

- Åpne: dra hele `Terreng`-mappen (eller filene) inn i vinduet. Verktøyet finner `prosjekt.json`, `.glb` og `.pdf` selv.
- Lagre i Chrome og Edge: `showSaveFilePicker()` skriver rett tilbake til samme fil i Dropbox.
- Lagre i Safari: filen lastes ned, og brukeren legger den i kundemappen. Vis en tydelig beskjed om hvor den skal.
- Skjermbilde: `renderer.domElement.toBlob()` lagrer et PNG av 3D-vinduet til kundemappen.

**Eksport til SketchUp** (senere): terrenget rundt tomta som GLB med `GLTFExporter`, med samme nullpunkt, og høydekurver som DXF.

## Steg 12: Testing og publisering

**Faste testprosjekter.** Bruk de to prosjektene fra forhåndsvisningen som fasit, og sjekk at tallene ikke endrer seg uventet når koden endres:

| Prosjekt | Tomteareal | Skjæring / fylling ved massebalanse | Avstand til grense |
| --- | --- | --- | --- |
| Auran – Fyhn, 513/24 (Sikt hus, 270°) | 1 078 m² | 113 / 109 m³ | 7,0 m |
| Sandsæter, 54/465 (Kjerne liten, 180°) | 3 670 m² | 51 / 57 m³ | 19,8 m |

Tallene gjelder standardinnstillingene: arbeidsrom 1,5 m, sokkel 0,35 m, helning normal og maks skråning 8 m.

**Automatiske tester.**

- Enhetstester (Vitest) for de rene funksjonene: koordinatomregning, punkt-i-polygon, avstander, solposisjon og skjæring/fylling på et lite kunstig terreng med kjent svar (for eksempel et skråplan der volumet kan regnes for hånd).
- Skjermbildetest (Playwright) som åpner begge testprosjektene fra lagrede data, venter til scenen er tegnet, og tar bilde. Bruk programvaregrafikk (`--use-angle=swiftshader`), så testen også kan kjøre i GitHub Actions.
- Kjør testene i samme GitHub Actions-jobb som publiseringen, før bygget. Feiler en test, publiseres ikke siden.

**Manuell sjekk før hver versjon:** åpne siden i Chrome på PC og i Safari på Mac, last inn en ekte GLB fra en kundemappe, flytt huset, slå av og på trær og nabobygg, og lagre prosjektet tilbake til Dropbox.

**Ytelse å holde seg innenfor:** første bilde innen 5 sekunder på en vanlig bærbar, flytting av huset uten synlig hakking, og under 1 GB minne i nettleseren.

**Publisering.** Slå sammen til `main`, så publiserer GitHub Actions til GitHub Pages på 1–2 minutter (se oppsettsdokumentet). Proxyen i steg 6 der trengs ikke: Kartverket og Overpass svarer direkte fra nettleseren.

## Rekkefølge og omfang

Bygg i denne rekkefølgen. Etter steg 6 har dere et verktøy som allerede er nyttig i salg (tomt, terreng og hus fra SketchUp), og resten kan komme i senere versjoner.

| Steg | Innhold | Anslag | Brukbart etter steget |
| --- | --- | --- | --- |
| 1 | Prosjektstruktur og biblioteker | 0,5 dag | – |
| 2 | Finn tomta | 1 dag | Tomtegrense vises |
| 3 | Terreng | 2–3 dager | Tomta i 3D |
| 4 | Vegetasjon | 1 dag | Skogen rundt tomta |
| 5 | Veier og nabobygg | 2 dager | Omgivelsene |
| 6 | Husmodellen | 2 dager | **Første versjon i bruk** |
| 7 | Plassering og kontroller | 2 dager | Avstandskrav sjekkes |
| 8 | Skjæring, fylling, støttemur | 3 dager | Masseoverslag til kalkyle |
| 9 | Plantegning på bakken | 1–2 dager | Situasjon for kunden |
| 10 | Sol, lys, bildekvalitet | 2 dager | Solstudie og gode bilder |
| 11 | Grensesnitt og prosjektfil | 3 dager | Lagring i Dropbox |
| 12 | Testing og publisering | 2 dager | Stabil drift |

Totalt ca. 21–24 arbeidsdager for én utvikler. Mye av koden finnes allerede i forhåndsvisningen og kan flyttes over, så med Claude som hjelp blir det trolig vesentlig raskere. Anslagene er grove og bør justeres etter steg 3.

## Lisenser og kilder

| Data | Kilde | Lisens og krav |
| --- | --- | --- |
| Tomtegrense, adresser | Kartverket, eiendoms- og adresse-API | CC BY 4.0, vis «© Kartverket» |
| Terreng og overflate (DTM1, DOM1) | Kartverket, Høydedata | CC BY 4.0, vis «© Kartverket» |
| Veier og bygningsomriss | OpenStreetMap via Overpass | ODbL, vis «© OpenStreetMap-bidragsytere» |
| Ortofoto (Norge i bilder) | Geovekst | Ikke brukt. Krever avklaring før det bygges inn |
| Tegninger og modeller | Neo hytter | Interne, skal aldri ligge i repoet |

Alle tre biblioteker (three.js, pdf.js, proj4) har åpne lisenser (MIT / Apache 2.0) som tillater intern bruk.

Kilder:

- [Kartverket – vilkår for bruk](https://www.kartverket.no/en/api-and-data/terms-of-use)
- [Kartverket – åpent tilgjengelige eiendomsdata og tjenester](https://kartverket.no/en/api-and-data/eiendomsdata/apent-tilgjengelige-data-og-tjenester)
- [Eiendoms-API, geokoding med gnr/bnr (eksempel i Arbeidstilsynets kode)](https://github.com/Arbeidstilsynet/dotnet-common/issues/243)
- [Geonorge – varsel om ny WCS for terrengmodell](https://register.geonorge.no/varsler/h%C3%B8yde-dtm-1-utm32-wcs/1b4a4737-8f3f-4e57-9bb5-a44a3c8695dc)
- [SketchUp – arbeid med glTF-filer](https://help.sketchup.com/en/sketchup/working-gltf-files)
- Tester av API-ene fra nettleseren, 15.10.2026: eiendom, adresse og høydedata svarte fra andre domener, og Overpass svarte etter nye forsøk.
