# Patroon Studio — Projectbeschrijving voor Claude Code

## Wat is dit?

Patroon Studio is een mobiel-first webapp voor het genereren van naaipatronen op maat. Het is gebouwd als een cadeau voor mijn vrouw, die geïnteresseerd is in naaien en modeontwerp. De app draait volledig in de browser (geen backend, geen framework) als één enkel HTML-bestand, gehost op Netlify.

**Live URL:** https://patroonstudio.netlify.app  
**Bronbestand:** `index.html` (één enkel bestand, ~80KB, ~966 regels)  
**Repository:** GitHub (private repo, GitHub Pages / Netlify deployment)

---

## Tech stack

- **Pure HTML/CSS/JS** — geen frameworks, geen bundler, geen npm
- **Canvas API** — voor het tekenen van patronen en stappenplan-illustraties
- **OpenRouter API** — gratis AI-tier voor NL-taalondersteuning (Llama 3.3 70B, DeepSeek V3, etc.)
- **jsPDF** (CDN) — voor PDF-export
- **Netlify** — hosting via drag-and-drop upload

De API-key voor OpenRouter wordt opgeslagen in `localStorage` van de gebruiker. Geen server-side code.

---

## Applicatiestructuur

### Tabs (navigatie)
1. **🏠 Start** — AI-functies, patroonreferentie, foto-analyse, aanpassingsadvies, kledingstuk kiezen
2. **📐 Maten** — invulformulier met maten, stijlkeuzes en sluiting per kledingstuk
3. **🗂 Patroon** — canvas-gebaseerd patroon op schaal 1:1 + printfuncties
4. **📋 Stappen** — AI-gegenereerd stappenplan met canvas-illustraties per stap
5. **🧵 Stof** — stofberekening en stofadvies op basis van de maten

### Kledingstukken (8 stuks, allemaal actief)
| ID | Naam | Matengroep |
|---|---|---|
| `broek` | Broek (straight/slim/wide) | `mg-broek` |
| `rok` | Rok (koker/A-lijn/maxi) | `mg-rok` |
| `tshirt` | T-shirt | `mg-shirt` |
| `polo` | Polo | `mg-shirt` |
| `blouse` | Blouse / Shirt | `mg-shirt` |
| `trui` | Trui / Sweater | `mg-shirt` |
| `vest` | Vest / Jas | `mg-shirt` |
| `jurk` | Jurk | `mg-jurk` |

### Matengroepen (HTML-structuur)
Elk kledingstuk heeft een `<div class="maten-group" id="mg-...">` die alleen zichtbaar is als `.active`. De groepen zitten allemaal binnen `#sec-maten`. Sluiting-pills zijn **binnen** de matengroep genest (na eerdere bugs waarbij ze erbuiten stonden en altijd zichtbaar waren).

---

## Bestandsopbouw (index.html)

```
<head>
  CSS (~200 regels inline)
  jsPDF CDN script tag

<body>
  .hdr (sticky header)
  .tabs (tab-navigatie)

  #sec-start
    .key-box (OpenRouter API key invoer)
    Patroonreferentie (patroonnummer opzoeken via AI)
    Foto upload + AI-analyse
    Aanpassing beschrijven (AI-advies)
    Kledingstuk kiezen (grid, 8 cards)

  #sec-maten
    #mg-broek (maten + stijl + sluiting)
    #mg-rok   (maten + stijl + sluiting)
    #mg-shirt (maten + pasvorm + sluiting/kraag — gedeeld door tshirt/polo/blouse/trui/vest)
    #mg-jurk  (maten + sluiting)
    Naadtoeslag (globaal)
    Genereer-knop

  #sec-patroon
    Stats bar (3 maten)
    Canvas #pat
    Export buttons (A4/A3/PNG/PDF)

  #sec-stappen
    #stepsList (canvas-kaartjes per stap)

  #sec-stof
    Stofberekening grid + advies

  <script> — State, tabs, API key
  <script> — Pattern drawing (Canvas)
  <script> — Stof, Steps, Print/Export, Icons
```

---

## JavaScript functies (volledig overzicht)

### Navigatie & UI
- `goTab(name)` — wisselt actieve tab, scrollt naar boven
- `updateKeyUI()` — toont/verbergt API key invoer
- `toast(msg)` — toont een tijdelijke mededeling

### Kledingstuk & maten
- `pickGarment(name, el)` — selecteert kledingstuk, toont juiste matengroep, past sluiting-pills aan
- `getPill(groupId)` — geeft de actieve pill-waarde terug
- `gv(id)` — helper: parseFloat van een input veld
- `SA()` — geeft naadtoeslag terug in cm (van de slider)

### AI (OpenRouter)
- `callAI(messages, hasImage)` — roept OpenRouter aan met fallback-modellen:
  1. `deepseek/deepseek-chat-v3-0324:free`
  2. `meta-llama/llama-3.3-70b-instruct:free`
  3. `mistralai/mistral-small-3.1-24b-instruct:free`
  4. `openrouter/auto`
  - Injects altijd een Dutch system prompt
- `callJSON(prompt, b64, mime)` — wrapper die JSON parsed uit AI-response
- `analyzeImg()` — analyseert geüploade foto (kledingstuktype, stijl, maattips)
- `lookupPnr()` — zoekt info op over patroonnummer (bv. Vogue V1486)
- `getAdj()` — geeft aanpassingsadvies (bv. "rok naar jurk")

### Patroon tekenen (Canvas)
- `generateAll()` — roept drawPattern + buildStof + buildSteps aan
- `drawPattern()` — dispatcher op basis van `currentGarment`
- `drawBroek(cv)` — broekpatroon: kwart-constructie methode
- `drawRok(cv)` — rokpatroon: trapeziumvorm met flare-slider
- `drawShirt(cv)` — shirt/polo/blouse/trui/vest: T-vorm + mouwen
- `drawJurk(cv)` — jurk: lijfje (T-vorm) + rok (trapezium) apart
- `mkCanvas(cv, W, H)` — setup canvas met DPR en ruitjesgrid
- `pan(ctx, pts, fill, stroke)` — teken een polygon
- `idash(ctx, pts, off, col)` — naadtoeslag stippellijn (inset)
- `grn(ctx, x1,y1,x2,y2)` — schietlood lijn met pijlen
- `lbl(ctx, cx, cy, t, s)` — tekst label op canvas
- `drawCalBar(ctx, x, y, S)` — 10cm kalibratiebalk

### Stappenplan
- `buildSteps()` — async; probeert AI-gegenereerde stappen, valt terug op `DEFAULT_STEPS`
  - Per stap: canvas-illustratie + titel + beschrijving (4+ zinnen) + tip
  - AI-prompt bevat: kledingstuk, sluiting, naadtoeslag → specifieke instructies
- `STEP_DRAWS` — object met canvas-tekenfuncties per stap per kledingstuktype
- `DEFAULT_STEPS` — fallback stappenplan (6 stappen per type)

### Stof
- `buildStof()` — berekent stoflengte op basis van maten + geeft stofadvies

### Printen & exporteren
- `printTiled(size)` — genereert tegel-printvenster voor A4/A3 op schaal 1:1
  - Overlap: 10mm, paginanummering, stippellijnen
- `exportPNG()` — downloadt canvas als PNG
- `exportPDF()` — maakt PDF via jsPDF op exacte schaal

### Overige
- `handleFile(input)` / `clearImg()` — foto upload handling
- `createIcons()` — genereert favicon en apple-touch-icon via canvas

---

## Data-structuren

### GARMENT_GROUP
```js
{broek:'broek', rok:'rok', tshirt:'shirt', polo:'shirt',
 blouse:'shirt', trui:'shirt', vest:'shirt', jurk:'jurk'}
```

### SHIRT_LABEL
```js
{tshirt:'T-shirt', polo:'Polo', blouse:'Blouse / Shirt',
 trui:'Trui / Sweater', vest:'Vest / Jas'}
```

### SHIRT_SLUITING
Per shirt-type: header-tekst + array van pills `{val, lbl, def}`.
- tshirt/trui: alleen halsafwerking (geen sluiting)
- polo: alleen polokraag placket
- blouse/vest: volledige sluitingskeuze

### DEFAULT_STEPS
```js
{broek: [...6 stappen], rok: [...6 stappen],
 shirt: [...6 stappen], jurk: [...6 stappen]}
```

### STEP_DRAWS
```js
{broek: [fn0, fn1, ...fn5], rok: [...], shirt: [...], jurk: [...]}
```
Elk een `(ctx, W, H) => void` canvas-tekenfunctie.

---

## Patroonberekening methode

### Broek (kwart-constructie)
- Voorpand breedte = heup/4 + 1cm (+ gemak)
- Achterpand breedte = heup/4 + 3cm
- Kruisvorm via punten: taille, heuplijn, kniehoogte, enkel
- Pijpwijdte via slider (14–50cm), schaalt lineair naar kniewijd punt

### Rok (trapezium)
- Taillebreedte = taille/4 + 1cm
- Onderbreedte = taille/4 + 1cm + flare-slider waarde
- Rechthoek-trapezium constructie

### Shirt (T-vorm)
- Breedte = (borst + gemak)/4 + 1cm
- Schouderbreedte / 2 = halve schouderbreedte
- Armsgat = halve schouder × 0.7
- Mouw: kap van 5cm, tapering naar polsomtrek

### Jurk
- Lijfje = shirt-methode (T-vorm)
- Rok = rok-methode (trapezium)
- Getekend als aparte panelen onder elkaar

---

## Bekende beperkingen & toekomstige verbeteringen

### Patroonkwaliteit
- Patronen zijn vereenvoudigd (rechte lijnen, geen echte curven)
- Kruislijn op broek is een kwadratische curve, niet de echte naaipatroon-curve
- Geen dart/nep constructie op het voorpand shirt
- Geen mouwhoogte-berekening (cap height is gefixeerd op 5cm)
- Geen grading (maten veranderen niet proportioneel per maat)

### Functies die ontbreken
- 3D visualisatie / avatar draping
- Maten opslaan / laden (localStorage)
- Meerdere patronen tegelijk
- Eigen patronen uploaden en aanpassen (bv. Vogue PDF)
- Metric/imperial omschakeling
- Meer kledingstukken: short, jumpsuit, jas met voering, bh-patroon
- Stofpatronen (ruitjes, strepen) uitgelijnd op patroon
- Notities per patroon

### AI-afhankelijkheden
- OpenRouter free tier heeft rate limits (200 req/dag, 20 req/min)
- Modelbeschikbaarheid wisselt — fallback-array nodig
- Foto-analyse vereist vision-model; `llama-4-maverick:free` en `qwen2.5-vl:free` worden geprobeerd

### UX/technisch
- Geen PWA service worker (offline werkt niet)
- Patronen worden niet opgeslagen tussen sessies
- Canvas-patronen zijn pixelgebaseerd (niet SVG/vector) — dus niet oneindig schaalbaar
- Printfunctie werkt niet vanuit WhatsApp in-app browser (vereist Safari op iOS)

---

## Deployment workflow

```
1. Bewerk index.html lokaal
2. Ga naar app.netlify.com → jouw site → Deploys
3. Sleep index.html in het drag-and-drop veld
4. Site is live binnen ~10 seconden
```

Alternatief via GitHub:
```
1. Push naar main branch
2. Netlify deployt automatisch (als gekoppeld aan repo)
```

---

## OpenRouter instelling

De OpenRouter API key wordt bij de eerste keer gebruik gevraagd en opgeslagen in `localStorage` onder de key `or_key`. De key begint met `sk-or-v1-`.

System prompt (altijd meegestuurd):
```
Je bent een expert naaipatroon-assistent. Antwoord ALTIJD in het Nederlands. Geef praktisch naai-advies.
```

---

## Mogelijke Claude Code taken

- **Patroonkwaliteit verbeteren**: echte curven toevoegen aan kruislijn, armsgat, schouderslope
- **Maten opslaan**: localStorage profiel met naam, maten, voorkeursinstellingen
- **SVG export**: patroon als vectorbestand i.p.v. canvas PNG
- **Rok uitbreiden**: volledige cirkelrok constructie, yoke rok
- **3D preview**: simpele Three.js draping simulatie
- **Meer kledingstukken**: short, jumpsuit, jas met voering, rok met volants
- **Patroon aanpassen**: direct op canvas klikken en punten verschuiven
- **Stiksimulator**: animatie die laat zien hoe je het in elkaar naait
- **Print kalibratie**: automatisch schaal aanpassen aan printer DPI
