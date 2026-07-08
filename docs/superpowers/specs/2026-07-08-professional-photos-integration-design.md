# Integrarea pozelor profesionale — Design

**Data:** 2026-07-08
**Site:** Pensiunea Ramedal (`index.html`, single-file, fără build)
**Sursă foto:** `~/Documents/ramedal-photos/` (105 poze pro, 12 foldere)

## Scop

Integrarea unui set de poze profesionale în site, cu un refresh vizual complet:
hero nou, upgrade la pozele existente, lightbox per cameră, o galerie filtrabilă
nouă și două spații care nu apar deloc azi (conferințe/evenimente, spălătorie).
Totul cu respectarea a două principii non-negociabile ale proprietarului:
**SEO** (site-ul e făcut pentru căutări) și **viteză** (pipeline responsive,
lazy loading, zero CLS).

## Decizii luate (blocate)

1. **Amploare:** refresh complet + galerie.
2. **Navigare foto:** carduri de cameră clicabile → lightbox per cameră; plus o
   secțiune nouă „Galerie" filtrabilă (Toate / Camere / Exterior / Facilități) cu
   lightbox partajat.
3. **Hero:** înlocuit cu `exterior-c` (colțul de cărămidă branduit, cer albastru;
   fișier sursă `exterior/CCBD24B3-624C-4E3F-83E7-BDFC67D8E152_1_102_o.jpeg`,
   2172×1448).
4. **Spații promovate activ (text + structură):** sala de conferințe/evenimente,
   spălătorie self-service, exterior & grădină.
5. **NU se promovează ca text/structură:** recepția și yala inteligentă (rămân
   doar poze în galerie).
6. **Camere:** un singur card per *tip* de cameră (nu carduri duplicate pentru
   același gen). Se trece de la 6 carduri la **5**.
7. **Denumiri:** fără „Parter", fără „Mansardă" în numele camerelor/apartamentelor.

## Lineup final de camere (5 carduri, unul per tip)

| # | Card | Paturi | Folder sursă | Poze lightbox |
|---|------|--------|--------------|---------------|
| 1 | Cameră Dublă | 1 pat matrimonial | `double-room` (22) | 5–6 (variante teal/olive/roz/bleumarin) |
| 2 | Cameră Twin | 2 paturi de o persoană | `twin-room` (3) | 3 (lead = cadrul cu 2 paturi) |
| 3 | Cameră Triplă | 1 pat dublu + 1 single | `triple-room` (22) | 5–6 |
| 4 | Apartament 2 Camere | dormitor + living | `2-rooms-apartament` (5) | 4–5 |
| 5 | Apartament 3 Camere | 3 camere | `3-rooms-apartamant` (6) | 5–6 |

- Lead-ul fiecărui card = cel mai luminos/reprezentativ cadru orizontal.
- Camera Twin: lead-ul actual e la 1024px (crop). Când proprietarul aduce pozele
  profesionale de twin, se înlocuiește doar lead-ul; structura rămâne.
- Statistica „12+ Camere & Apartamente" rămâne validă (număr fizic neschimbat).

## Modificări pe secțiuni

### Hero (`#top`)
- Regenerez `images/hero-{960,1440,1920}.{avif,webp}` din sursa `exterior-c`.
- Master: copiez sursa `exterior-c` peste `images/hero.jpg` (masterul vechi al
  hero-ului nu e servit și nu e folosit altundeva).
- Rămâne `fetchpriority="high"`, fără `loading="lazy"` (e elementul LCP).
- Alt text nou (RO): „Fațada Pensiunii Ramedal Calafat — cazare în centrul orașului".

### Despre (`#despre`)
- Upgrade la cele 3 imagini existente (azi: corridor + double-green + bathroom):
  - `about-img-main` → un exterior elegant (aleea cu coloane `exterior-b` sau vila
    `exterior-tall`).
  - `about-img-sub` ×2 → o cameră dublă profesională + baia cu duș (`bathroom`).
- Se păstrează layoutul și `about-badge` (9.0★).

### Camere (`#camere`)
- 6 carduri → 5 (vezi lineup). Se elimină cardul dublă-duplicat și se
  redenumesc conform deciziilor.
- Fiecare `.room-card` devine clicabil (buton/`role`), deschide lightbox-ul cu
  albumul tipului respectiv. Lead-ul rămâne în card ca `<picture>` responsive.
- Se actualizează `data-i18n` pentru nume/size/desc pentru cele 5 carduri
  (chei `r1..r5`), în ambele limbi.

### Secțiune nouă „Conferințe / Evenimente" (`#conferinte`)
- Plasare: după `#facilitati`, înainte de `#locatie`.
- Structură: `section-label` + `h2` + divider + 2 paragrafe copy + 2–3 poze
  (din `confference-room`).
- SEO: text natural cu „sală de conferințe / evenimente în Calafat", potrivită
  pentru ședințe, întâlniri de business și evenimente.
- Chei i18n noi: `conf_label`, `conf_h2`, `conf_p1`, `conf_p2` (RO+EN).
- **Asumție de confirmat:** spațiul e sală de conferințe/evenimente (posibil și
  mic dejun). Copy-ul se ajustează dacă proprietarul precizează altfel.

### Facilități (`#facilitati`)
- Se adaugă un tile nou „Spălătorie self-service" (🧺), chei `fac13_title` /
  `fac13_desc` (RO+EN). Restul tile-urilor rămân.

### Galerie nouă (`#galerie`)
- Plasare: după `#conferinte`, înainte de `#locatie`.
- `section-header` + rând de butoane filtru + grilă responsivă de thumbnails.
- Filtre: **Toate / Camere / Exterior / Facilități** (chei `gal_all`,
  `gal_rooms`, `gal_exterior`, `gal_fac`).
- Fiecare thumbnail e clicabil → lightbox partajat, deschis pe indexul corect în
  cadrul categoriei active.
- Curare: ~24–32 poze. „Camere" reutilizează pozele camerelor (fără fișiere noi).
  „Exterior" ~6–8, „Facilități" = conferințe + baie + spălătorie + recepție.

### Navigație
- Se adaugă „Galerie" (`#galerie`) în `nav-links` și în `mobile-menu`, cu cheia
  `nav_gallery` (RO+EN). Conferințele NU se adaugă în nav (meniu lejer).

## Componenta Lightbox (partajată)

Vanilla JS, fără librării (regula „no framework"). Un singur lightbox folosit și
de cardurile de cameră, și de galerie.

- **Model de date:** un obiect JS `albums` cu chei per album
  (`double`, `twin`, `triple`, `apt2`, `apt3`, plus categoriile galeriei). Fiecare
  intrare = listă de `{ base, w, h, alt }` (base = numele fișierului fără
  `-<width>.<ext>`). Lightbox-ul construiește `<picture>` la deschidere și încarcă
  varianta 1280 → **pozele din lightbox se încarcă doar la deschidere**, nu la load.
- **Deschidere:** click pe card → album-ul tipului; click pe thumbnail galerie →
  albumul categoriei active, la indexul dat.
- **Controale:** butoane ◀ ▶, contor „i / n", buton ✕, click pe fundal închide,
  taste ← / → / Esc.
- **Accesibilitate:** `role="dialog"`, `aria-modal="true"`, focus-trap în lightbox,
  focus restaurat pe elementul declanșator la închidere, `body` scroll-lock cât e
  deschis, `aria-label` pe butoane (localizate).
- **Stiluri:** override la `img{width:100%}` global (imaginea din lightbox e
  `max-width/max-height` cu `object-fit:contain`), overlay `position:fixed;inset:0`
  cu fundal întunecat, z-index peste navbar.

## Pipeline imagini & buget de viteză

- Fiecare sursă nouă → **AVIF (q60) + WebP (q80)** la 3 lățimi.
  - Conținut (camere/galerie/spații): **480 / 800 / 1280** (cap la rezoluția
    nativă — unele crop-uri sunt 1024, nu se face upscale).
  - Hero: **960 / 1440 / 1920**.
- Proces (per CLAUDE.md): `sips --resampleWidth <w>` → PNG temp → `cwebp -q 80` +
  `avifenc -q 60`. Encodere: `brew install webp libavif`.
- **Fallback `<img>`** în fiecare `<picture>` = varianta 800 `.webp` (800 e mai
  reprezentativ decât 480 pt. carduri).
- **Lazy:** toate `loading="lazy"` + `decoding="async"`, mai puțin hero.
- **CLS:** `width`/`height` intrinseci pe fiecare `<img>`.
- **Buget:** ~40–45 surse unice noi × 6 fișiere ≈ 240–270 derivate. Pe load
  inițial se încarcă doar: hero (prioritar) + 5 lead-uri camere (lazy) +
  thumbnails galerie (lazy, sub fold) + 3 poze Despre (lazy) + 2–3 poze
  conferințe (lazy). Lightbox = zero cost până la click.

## Convenție de denumire fișiere

`images/<base>-<width>.{avif,webp}` (identic cu convenția existentă).

- Camere: `room-double-<n>`, `room-twin-<n>`, `room-triple-<n>`, `apt2-<n>`,
  `apt3-<n>` (n de la 1; lead = `-1`).
- Spații/galerie: `exterior-<n>`, `reception-<n>`, `conference-<n>`,
  `bathroom-<n>`, `laundry-1`.
- Hero: rămâne `hero`.
- Despre: reutilizează bazele de mai sus (fără baze noi dedicate).
- Bazele vechi înlocuite (`room-double-teal`, `room-double-green`, `room-triple`,
  `room-attic`, `room-studio`, `room-attic-studio`, `corridor`) → se șterg
  derivatele lor după ce HTML-ul nu le mai referă. Masterele `.jpg` NU se șterg.

## i18n (bilingv RO/EN)

- Tot textul nou: scris în RO direct în markup **și** adăugat ca cheie în **ambele**
  obiecte `ro` și `en` din obiectul `translations`.
- Chei noi: `nav_gallery`, `gal_label`, `gal_h2`, `gal_sub`, `gal_all`,
  `gal_rooms`, `gal_exterior`, `gal_fac`, `conf_label`, `conf_h2`, `conf_p1`,
  `conf_p2`, `fac13_title`, `fac13_desc`, plus chei `r1..r5` actualizate și
  `aria`-uri lightbox (`lb_prev`, `lb_next`, `lb_close`).
- **Alt text:** rămâne RO-only (consecvent cu site-ul actual — `applyLang` nu
  traduce atribute `alt`). Nu extindem scopul aici. Captionul lightbox = același
  text RO.

## SEO

- Alt-uri descriptive, cu cuvinte-cheie naturale (Calafat, tip cameră) pe toate
  pozele noi.
- JSON-LD `LodgingBusiness` (linia din `<head>`) — se extinde, păstrând sincron
  cu conținutul vizibil și meta:
  - Se adaugă `image`: array cu 4–6 URL-uri absolute reprezentative
    (`https://ramedal.clft.ro/images/...`).
  - Se adaugă `amenityFeature`: `LocationFeatureSpecification` pentru WiFi gratuit,
    parcare gratuită, aer condiționat, cameră nefumători, spălătorie
    (self-service), sală de conferințe/evenimente.
- `og:image` / `twitter:image` rămân `og-cover.jpg` (nu se schimbă).

## Fonturi

- „Conferințe", „evenimente", „spălătorie" folosesc doar ă/î/ț/ș, deja incluse.
- După adăugarea copy-ului: verific glifele; dacă apare vreun caracter nou,
  re-subsetez din masterele variabile cu `pyftsubset --text-file=index.html
  --flavor=woff2` (instanțiere pe greutate întâi).

## Testare / verificare

- Servire locală: `python3 -m http.server 8000`; deschidere în browser.
- De verificat:
  - Pozele se încarcă; `srcset` alege lățimea potrivită; hero e LCP-ul.
  - Lightbox: deschide / navighează ◀▶ / închide / taste / focus-trap / restore.
  - Filtrele galeriei ascund/afișează corect; lightbox-ul respectă categoria.
  - Toggle limbă traduce tot textul nou (RO↔EN), inclusiv filtre și conferințe.
  - Zero CLS vizibil; fără erori în consolă.
  - JSON-LD valid (fără rupere de sintaxă) — test cu validator.
  - Cele 3 puncte SEO rămân sincronizate (meta / JSON-LD / conținut vizibil).

## În afara scopului

- Recepția și yala inteligentă ca text/structură (doar galerie).
- Restructurarea ofertei de camere dincolo de consolidarea cardurilor.
- Traducerea atributelor `alt`.
- Fotografie nouă (twin profesional — proprietarul aduce ulterior, se schimbă
  doar lead-ul).

## Asumții de confirmat la revizuire

1. „Conferințe/evenimente" e denumirea corectă a sălii (sau e și sală de mic dejun?).
2. Selecția exactă de poze per cameră/galerie se finalizează la implementare, din
   foldere, alegând cadrele cele mai luminoase și orizontale ca lead-uri.
3. Ordinea cardurilor: Dublă → Twin → Triplă → Apartament 2 → Apartament 3.
