# Oppsett: Neo Tomt3D på GitHub Pages

Oct 15, 2026 · @Kim

## Oversikt

Resultatet er et internt nettverktøy på `https://neohytter.github.io/neo-tomt3d/`, som alle fire åpner i nettleseren på PC eller Mac. Koden ligger på GitHub og publiseres automatisk hver gang noen lagrer en endring. Kundefiler (SketchUp, PDF) blir liggende i Dropbox og lastes aldri opp.

| Hvem | Oppgave | Tid |
| --- | --- | --- |
| Kim (administrator) | Steg 1–7: GitHub, prosjekt, publisering, test | 2–3 timer |
| Den som skriver kode | Steg 2 (utviklingsmiljø) på egen maskin | 30 min |
| Erlend, Roar, Stian | GitHub-konto + bokmerke til verktøyet | 15 min |

Forutsetninger: en nyere nettleser (Chrome, Edge eller Safari), og at Kartverket-tjenestene svarer fra nettleseren (testes i steg 5). Navnene `neohytter` og `neo-tomt3d` er forslag og kan byttes.

## Steg 1: GitHub-organisasjon og repo

Målet er én felles organisasjon som eier koden, slik at verktøyet ikke henger på én persons konto.

1. Alle fire lager en GitHub-konto på [github.com/signup](https://github.com/signup) med @neohytter.no-adressen og slår på tofaktorinnlogging (Settings → Password and authentication).
2. Kim oppretter organisasjonen: profilbildet → Your organizations → New organization → Free. Navn: `neohytter`.
3. I organisasjonen: New repository → navn `neo-tomt3d`, ingen README (den kommer fra prosjektet i steg 3).
   - **Public** på gratisplanen. Det er greit fordi repoet bare inneholder kode, aldri kundedata.
   - **Private** krever GitHub Team (ca. 4 USD per bruker per måned). Nettsiden blir likevel åpen; det er bare koden som skjules.
4. Inviter de andre: organisasjonen → People → Invite member. Gi dem rollen Member og skrivetilgang (Write) til `neo-tomt3d`.
5. Valgfritt: Settings → Branches → beskytt `main`, så endringer må gå via pull request.

## Steg 2: Utviklingsmiljø (PC og Mac)

Bare den som skal skrive eller endre kode trenger dette. Vanlige brukere trenger kun nettleseren.

**Windows** (i PowerShell):

```powershell
winget install Git.Git OpenJS.NodeJS.LTS Microsoft.VisualStudioCode
```

**Mac** (i Terminal, etter at [Homebrew](https://brew.sh) er installert):

```bash
brew install git node
brew install --cask visual-studio-code
```

Sjekk at alt er på plass, og fortell Git hvem du er:

```bash
node -v    # skal vise v22 eller nyere
git --version
git config --global user.name "Kim Holten"
git config --global user.email "kim@neohytter.no"
```

**Viktig:** Legg koden i en egen mappe utenfor Dropbox, for eksempel `~/Kode/neo-tomt3d`. Mappen `node_modules` inneholder titusenvis av småfiler som får Dropbox-synkroniseringen til å gå i stå. GitHub er synkroniseringen for koden.

## Steg 3: Opprett prosjektet

Prosjektet bygges med Vite (byggeverktøy) og TypeScript, med Giro3D som 3D-motor.

```bash
cd ~/Kode
npm create vite@latest neo-tomt3d -- --template vanilla-ts
cd neo-tomt3d
npm install
npm install @giro3d/giro3d three proj4 geotiff pdfjs-dist suncalc
npm install -D @types/three @types/suncalc
```

Sett adressen nettsiden skal ligge på i `vite.config.ts` (lag filen hvis den mangler):

```ts
import { defineConfig } from 'vite';

export default defineConfig({
  base: '/neo-tomt3d/', // må være lik repo-navnet
});
```

Sperr søkemotorer. Lag `public/robots.txt`:

```text
User-agent: *
Disallow: /
```

og legg denne linjen inn i `<head>` i `index.html`:

```html
<meta name="robots" content="noindex, nofollow">
```

Legg til en sikring mot kundefiler nederst i `.gitignore`:

```text
# Kundefiler skal aldri inn i repoet
*.skp
*.pdf
*.glb
*.dwg
prosjekt.json
```

Lag `src/config.ts`, som styrer om kall går direkte til Kartverket eller via proxy (steg 6):

```ts
// Tom streng = direkte kall. Fylles inn i steg 6 hvis det trengs.
export const KARTVERKET_PROXY = '';

export const kartUrl = (url: string) =>
  KARTVERKET_PROXY ? KARTVERKET_PROXY + encodeURIComponent(url) : url;
```

Opprett mappene fra forslaget: `src/oppslag`, `src/data`, `src/scene`, `src/eksport`, `src/prosjekt`, `src/ui`. Test lokalt med `npm run dev` og åpne `http://localhost:5173/neo-tomt3d/`.

Send prosjektet til GitHub:

```bash
git init
git add .
git commit -m "Første versjon"
git branch -M main
git remote add origin https://github.com/neohytter/neo-tomt3d.git
git push -u origin main
```

## Steg 4: Publiser med GitHub Pages

Etter dette steget publiseres nettsiden automatisk 1–2 minutter etter hver endring i `main`.

1. I repoet: Settings → Pages → Build and deployment → Source: **GitHub Actions**.
2. Lag filen `.github/workflows/deploy.yml`:

```yaml
name: Publiser til GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

3. Send filen til GitHub:

```bash
git add .github
git commit -m "Automatisk publisering"
git push
```

4. Følg kjøringen under fanen **Actions** i repoet. Når den er grønn, ligger siden på `https://neohytter.github.io/neo-tomt3d/`.
5. Send lenken til Erlend, Roar og Stian og be dem lagre den som bokmerke.

Feiler bygget, klikk på den røde kjøringen for feilmeldingen. Vanligste årsak er at `base` i `vite.config.ts` ikke matcher repo-navnet.

## Steg 5: Test om Kartverket svarer fra nettleseren

Denne testen avgjør om steg 6 trengs. Den tar fem minutter.

1. Åpne `https://neohytter.github.io/neo-tomt3d/` i Chrome eller Edge.
2. Åpne konsollen: F12 (PC) eller Cmd+Option+J (Mac). I Safari må Utvikle-menyen slås på under Innstillinger → Avansert.
3. Lim inn og trykk Enter:

```js
const tester = {
  adresse: 'https://ws.geonorge.no/adresser/v1/sok?sok=Brendeberget&treffPerSide=1',
  eiendom: 'https://api.kartverket.no/eiendom/v1/geokoding?kommunenummer=3326&gardsnummer=62&bruksnummer=175&utkoordsys=25832',
  terreng: 'LIM_INN_WCS_ADRESSE?service=WCS&request=GetCapabilities',
};
for (const [navn, url] of Object.entries(tester)) {
  fetch(url)
    .then(r => console.log(navn, r.ok ? 'OK' : 'svarte med ' + r.status))
    .catch(e => console.log(navn, 'BLOKKERT (CORS)', e.message));
}
```

For terreng: finn den eksakte tjenesteadressen på [Nasjonal høydemodell DTM 25832 WCS i Kartkatalogen](https://kartkatalog.geonorge.no/metadata/nasjonal-hoeydemodell-digital-terrengmodell-25832-wcs/05821c51-2f5b-411f-9098-924d13dbea9a) og lim den inn. Eksempelet bruker Ramton-tomta (62/175 i Hemsedal).

| Resultat | Betyr | Neste steg |
| --- | --- | --- |
| OK | Nettleseren får lov til å hente data | Hopp over steg 6 |
| svarte med 400/404 | Tjenesten svarer, men parameterne er feil (f.eks. kommunenummer) | Hopp over steg 6, rett parameterne i koden |
| BLOKKERT (CORS) | Tjenesten avviser kall fra nettsider | Gjør steg 6 |

## Steg 6 (bare ved behov): Proxy i Cloudflare

En liten Cloudflare Worker sender kallene videre til Kartverket og legger til tillatelsen nettleseren krever. Gratisplanen gir 100 000 kall per dag, langt mer enn fire brukere trenger. Nettsiden blir fortsatt liggende på GitHub Pages.

1. Lag gratis konto på [dash.cloudflare.com](https://dash.cloudflare.com) med kim@neohytter.no, og slå på tofaktorinnlogging.
2. Workers & Pages → Create → Create Worker. Navn: `neo-kartproxy` → Deploy.
3. Edit code, erstatt alt med koden under, og trykk Deploy:

```js
// Bare disse Kartverket-adressene kan hentes via proxyen
const TILLATTE_VERTER = [
  'ws.geonorge.no', 'api.kartverket.no',
  'wcs.geonorge.no', 'wms.geonorge.no', 'cache.kartverket.no',
];
// Bare vår egen nettside (og lokal utvikling) får bruke proxyen
const TILLATTE_SIDER = ['https://neohytter.github.io', 'http://localhost:5173'];

export default {
  async fetch(request) {
    const origin = request.headers.get('Origin') || '';
    const cors = {
      'Access-Control-Allow-Origin': TILLATTE_SIDER.includes(origin) ? origin : TILLATTE_SIDER[0],
      'Access-Control-Allow-Methods': 'GET, OPTIONS',
      'Vary': 'Origin',
    };
    if (request.method === 'OPTIONS') return new Response(null, { headers: cors });

    const target = new URL(request.url).searchParams.get('url');
    if (!target) return new Response('Mangler url', { status: 400, headers: cors });

    const targetUrl = new URL(target);
    if (targetUrl.protocol !== 'https:' || !TILLATTE_VERTER.includes(targetUrl.hostname)) {
      return new Response('Ikke tillatt', { status: 403, headers: cors });
    }

    // Mellomlagre svar i et døgn, så samme tomt lastes raskt neste gang
    const svar = await fetch(targetUrl, { cf: { cacheTtl: 86400, cacheEverything: true } });
    const ut = new Response(svar.body, svar);
    for (const [k, v] of Object.entries(cors)) ut.headers.set(k, v);
    return ut;
  },
};
```

4. Kopier Worker-adressen (`https://neo-kartproxy.<konto>.workers.dev`) og fyll den inn i `src/config.ts`:

```ts
export const KARTVERKET_PROXY = 'https://neo-kartproxy.<konto>.workers.dev/?url=';
```

5. All kode som henter Kartverket-data bruker `kartUrl(...)` rundt adressen. Da kan proxyen slås av og på ved å endre én linje.
6. Kjør testen i steg 5 på nytt, men sett Worker-adressen + `?url=` foran hver adresse (adressen kodet med `encodeURIComponent`). Alle skal nå vise OK.

## Steg 7: Dropbox-mappe og SketchUp-eksport

Hvert kundeprosjekt får en mappe `Terreng` under `1. Tegninger`. Legg den også inn i malen `-- Kundemappe --`, så nye kunder får den automatisk.

| Fil | Innhold | Lages av |
| --- | --- | --- |
| `prosjekt.json` | Kommune, gnr/bnr, nullpunkt, plassering, gulvhøyde, PDF-punkter | Verktøyet (Lagre prosjekt) |
| `modell.glb` | 3D-modellen | SketchUp |
| `situasjonsplan.pdf` | Situasjonsplanen som skal legges på terrenget | Kopi fra Byggesøknad |
| `eksport/` | Terreng til SketchUp (.glb), høydekoter (.dxf), bilder | Verktøyet |

Eksempel på `prosjekt.json` (verdiene fylles inn av verktøyet):

```json
{
  "versjon": 1,
  "kunde": "10119 Ramton",
  "kommunenummer": "3326",
  "gnr": 62,
  "bnr": 175,
  "koordsys": "EPSG:25832",
  "nullpunkt": { "ost": 0, "nord": 0 },
  "gulvhoyde_moh": 0,
  "modell": { "fil": "modell.glb", "rotasjon_grader": 0, "forskyvning_m": [0, 0] },
  "pdf": { "fil": "situasjonsplan.pdf", "side": 1, "punkter": [] }
}
```

**Nullpunktet** er tomtens UTM-koordinat avrundet til nærmeste 100 m. Alt verktøyet eksporterer, måles fra dette punktet. Når SketchUp-modellen har origo i samme punkt, havner den riktig uten manuell flytting.

**Eksport fra SketchUp:**

1. Skjul eller slett hjelpegeometri og gammelt terreng.
2. Rydd bort ubrukte komponenter og materialer: Vindu → Modellinfo → Statistikk → Fjern ubrukte.
3. Fil → Eksporter → 3D-modell → **GLTF Binary File (\*.glb)**.
4. Lagre som `modell.glb` i `Terreng`-mappen.

Hold filen helst under 50 MB. Store teksturer gjør verktøyet tregt, særlig på eldre maskiner.

## Sikkerhet, vedlikehold og daglig bruk

Siden er offentlig tilgjengelig for den som kjenner adressen. Derfor gjelder disse reglene:

- Ingen kundefiler, eksempelmodeller fra ekte prosjekter eller skjermbilder av kundetomter i repoet. `.gitignore` fra steg 3 er en sikring, ikke en garanti.
- Ingen passord eller API-nøkler i koden. Kartverket krever ingen.
- Vis «© Kartverket» nederst i verktøyet. Det er et krav i lisensen (CC BY 4.0).
- Ikke bygg inn ortofoto (Norge i bilder) før det er avklart med Kartverket eller Geovekst.
- Tofaktorinnlogging på GitHub og Cloudflare for alle.

**Daglig bruk:** Åpne bokmerket, dra inn filer fra kundens `Terreng`-mappe, og lagre prosjektet tilbake dit.

**Endringer i koden:** Lag en egen gren, åpne en pull request, og slå den sammen med `main`. Nettsiden oppdateres automatisk etter 1–2 minutter. Blir noe feil, angre endringen med Revert på pull requesten.

**Vedlikehold:** Slå på Dependabot (Settings → Code security → Dependabot version updates). Den foreslår oppdateringer av bibliotekene automatisk; se over dem én gang i måneden.

## Sjekkliste

- [ ] GitHub-kontoer med tofaktor for Kim, Erlend, Roar og Stian
- [ ] Organisasjon `neohytter` og repo `neo-tomt3d` opprettet, alle invitert
- [ ] Git, Node og VS Code installert hos den som utvikler
- [ ] Prosjektet opprettet med `vite.config.ts`, `robots.txt`, noindex og `.gitignore`-sikring
- [ ] GitHub Actions publiserer, og siden åpner på `neohytter.github.io/neo-tomt3d`
- [ ] CORS-test kjørt mot adresse, eiendom og terreng
- [ ] Cloudflare-proxy satt opp (bare hvis testen viste BLOKKERT)
- [ ] `Terreng`-mappe lagt inn i `-- Kundemappe --`
- [ ] Første testeksport fra SketchUp til `modell.glb`
- [ ] Ortofoto-lisens avklart med Kartverket eller Geovekst (eller valgt bort)
- [ ] Dependabot slått på

Kilder: [Kartverket – vilkår for bruk](https://www.kartverket.no/en/api-and-data/terms-of-use) · [Kartverket – åpne eiendomsdata](https://kartverket.no/en/api-and-data/eiendomsdata/apent-tilgjengelige-data-og-tjenester) · [GitHub – synlighet for Pages](https://docs.github.com/en/enterprise-cloud@latest/pages/getting-started-with-github-pages/changing-the-visibility-of-your-github-pages-site) · [SketchUp – glTF-eksport](https://help.sketchup.com/en/sketchup/working-gltf-files)
