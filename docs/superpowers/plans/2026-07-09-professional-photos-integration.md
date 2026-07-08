# Professional Photos Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate the professional photo set into `index.html` with a full visual refresh — new hero, upgraded About images, 5 clickable room cards with a shared lightbox, a new Conference/Events section, a self-service Laundry facility tile, and a new filterable Gallery — while keeping SEO and page speed intact.

**Architecture:** The site stays a single static `index.html` (inline `<style>` + `<script>`, no build, no framework). New photos are processed through the existing responsive pipeline (AVIF q60 + WebP q80 at fixed widths) into `images/`. A single vanilla-JS lightbox component (data-driven `albums` object) serves both the room cards and the gallery; photos in the lightbox load only on open. All new copy is written in Romanian in the markup **and** mirrored into both the `ro` and `en` translation objects.

**Tech Stack:** HTML5 `<picture>`/`srcset`, inline CSS custom properties + grid, vanilla ES5/ES6 JS, `sips`+`cwebp`+`avifenc` for image encoding. No dependencies, no test runner — verification is done by serving locally (`python3 -m http.server`) and inspecting in a browser, plus shell assertions on generated files.

## Global Constraints

Every task's requirements implicitly include this section. Values copied verbatim from the spec / CLAUDE.md:

- **No build system, no framework, no dependencies.** All CSS inline in the `<style>` block, all JS inline in the `<script>` block of `index.html`.
- **Bilingual RO/EN.** `currentLang='ro'`; `applyLang()` runs ONLY on `toggleLang()`, NOT on load. Therefore every new/changed visible string MUST be written in Romanian directly in the markup **and** added as a key in **both** the `ro` and `en` objects. Keys identical across `ro`/`en`. Values are injected via `innerHTML` (may contain `<strong>`/`<br>`).
- **`alt` attributes stay Romanian-only** (consistent with the current site — `applyLang` does not translate attributes). Out of scope to change.
- **Image pipeline:** each source → AVIF (`avifenc -q 60`) + WebP (`cwebp -q 80`) at widths **480/800/1280** for content, **960/1440/1920** for the hero. **No upscaling** — skip any width above the source's native width. Files named `images/<base>-<width>.{avif,webp}`.
- **Every `<img>` carries explicit `width`/`height`** (CLS) and `loading="lazy"` + `decoding="async"`, **except the hero** (`fetchpriority="high"`, no lazy — it is the LCP element).
- **The `.jpg` files in `images/` are uncompressed masters — NEVER delete them.** Only ever delete `.avif`/`.webp` derivatives.
- **SEO 3-way sync:** business facts live in (1) `<head>` meta/OG/Twitter, (2) the JSON-LD `LodgingBusiness` single line, (3) visible content. Keep them consistent. Phone `+40774326061`, email `contact@ramedal.ro`, rating 9.0/24/10, address Strada Gheorghe Doja 9, Calafat, Dolj, 205200.
- **Do not delete `CNAME`** (`ramedal.clft.ro`) — it maps the custom domain.
- **Work on branch `photos-integration`** (already checked out). Commit after each task. Do NOT push / do NOT touch `master` (that is production) unless the user asks.
- **Commit message trailer:** end each commit body with `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
- **Absolute repo path:** `/Users/geani/Documents/ramedal-website`. Photo sources: `/Users/geani/Documents/ramedal-photos/<folder>/`.

---

## File Structure

- **Modify:** `/Users/geani/Documents/ramedal-website/index.html` — the only shipped file. All HTML/CSS/JS edits land here.
- **Create (in scratchpad, NOT committed):** `<scratchpad>/gen-images.sh` — reusable encoder helper. Scratchpad = `/private/tmp/claude-502/-Users-geani-Documents-ramedal-website/7dd9b011-ab8f-485f-b8b4-f913d41f9455/scratchpad`.
- **Create (in scratchpad, NOT committed):** `<scratchpad>/manifest.md` — the locked source→base mapping produced in Task 2 and consumed by Tasks 3, 7, 10.
- **Create (committed):** derivative image files under `/Users/geani/Documents/ramedal-website/images/` (Task 3).
- **Modify (masters):** `images/hero.jpg` is overwritten by the chosen exterior source (Task 3); the old hero master is not served elsewhere.

> **Naming convention (fixed base names — independent of which source maps to each):**
> Rooms: `room-double-1..6`, `room-twin-1..3`, `room-triple-1..6`, `apt2-1..5`, `apt3-1..6`.
> Spaces/gallery: `exterior-1..8`, `conference-1..3`, `bathroom-1..3`, `laundry-1`, `reception-1..2`.
> Hero: `hero`. Lead of every room album is `-1`.

---

## Task 1: Reusable image-generation helper

**Files:**
- Create: `<scratchpad>/gen-images.sh`

**Interfaces:**
- Produces: a script `gen-images.sh <source> <base> <w1> [w2] [w3]` that writes `images/<base>-<w>.{avif,webp}` for each width ≤ the source's native width, and prints `BASE <base> MAXW <w> DIMS <w>x<h>` (the largest width generated and its pixel dimensions). Tasks 3 relies on this contract.

- [ ] **Step 1: Confirm encoders are present**

Run:
```bash
which cwebp avifenc sips
```
Expected: three paths printed (e.g. `/opt/homebrew/bin/cwebp`, `/opt/homebrew/bin/avifenc`, `/usr/bin/sips`). If any is missing: `brew install webp libavif`.

- [ ] **Step 2: Write the helper script**

Write `<scratchpad>/gen-images.sh`:
```bash
#!/usr/bin/env bash
# Regenerate responsive AVIF+WebP derivatives for one source image.
# Usage: gen-images.sh <source-file> <base-name> <width1> [width2] [width3] ...
# Writes images/<base>-<w>.{avif,webp} for each width <= source native width (no upscaling).
# Prints: BASE <base> MAXW <maxw> DIMS <w>x<h>
set -euo pipefail
SRC="$1"; BASE="$2"; shift 2
OUT="/Users/geani/Documents/ramedal-website/images"
TMP="$(mktemp -d)"; trap 'rm -rf "$TMP"' EXIT
NATW="$(sips -g pixelWidth "$SRC" | awk '/pixelWidth/{print $2}')"
MAXW=0; MW=0; MH=0
for W in "$@"; do
  if [ "$W" -gt "$NATW" ]; then continue; fi
  PNG="$TMP/$BASE-$W.png"
  sips --resampleWidth "$W" "$SRC" --out "$PNG" >/dev/null
  cwebp -quiet -q 80 "$PNG" -o "$OUT/$BASE-$W.webp" >/dev/null
  avifenc -q 60 "$PNG" "$OUT/$BASE-$W.avif" >/dev/null 2>&1
  MW="$W"; MH="$(sips -g pixelHeight "$PNG" | awk '/pixelHeight/{print $2}')"
  MAXW="$W"
done
echo "BASE $BASE MAXW $MAXW DIMS ${MW}x${MH}"
```

- [ ] **Step 3: Make executable and smoke-test on one source**

Run:
```bash
chmod +x <scratchpad>/gen-images.sh
<scratchpad>/gen-images.sh /Users/geani/Documents/ramedal-photos/exterior/CCBD24B3-624C-4E3F-83E7-BDFC67D8E152_1_102_o.jpeg _smoketest 480 800 1280
ls -la /Users/geani/Documents/ramedal-website/images/_smoketest-*
```
Expected: `BASE _smoketest MAXW 1280 DIMS 1280x853` (or similar), and six files `_smoketest-{480,800,1280}.{avif,webp}` listed.

- [ ] **Step 4: Clean up the smoke-test artifacts**

Run:
```bash
rm -f /Users/geani/Documents/ramedal-website/images/_smoketest-*
```
Expected: no output; files gone. (No commit — the script lives in scratchpad, and no repo files changed.)

---

## Task 2: Lock the photo selection into a manifest

Photo selection needs human/visual judgment (brightest, most horizontal frames as leads). This task produces the locked mapping that every image + HTML task consumes. No `index.html` edits here.

**Files:**
- Create: `<scratchpad>/manifest.md`
- Create (temporary): `<scratchpad>/thumbs/` contact thumbnails

**Interfaces:**
- Produces: `manifest.md` — for every base name, the absolute source path, the width set to generate, and `MAXW` (800 for ~1024px `_105_c` crops, 1280 for `_102_o` originals). Tasks 3/7/10 consume it.

- [ ] **Step 1: Generate small contact thumbnails for triage**

Run (one 320px webp per source, into scratchpad, so choices can be eyeballed):
```bash
SRC=/Users/geani/Documents/ramedal-photos; T=<scratchpad>/thumbs; mkdir -p "$T"
find "$SRC" -type f \( -iname '*.jpeg' -o -iname '*.jpg' \) | while read -r f; do
  d=$(basename "$(dirname "$f")"); b=$(basename "$f" | cut -c1-8)
  sips --resampleWidth 320 "$f" --out "$T/${d}__${b}.png" >/dev/null 2>&1
done
ls "$T" | head
```
Expected: PNG thumbnails named `<folder>__<8char>.png`.

- [ ] **Step 2: Review thumbnails and choose frames per base**

Open the thumbnails (Read tool renders images) and pick, favouring bright, horizontal, representative frames for each `-1` lead:
- `room-double-1..6` ← 6 best from `double-room/` (prefer `_102_o` originals so MAXW=1280).
- `room-twin-1..3` ← all 3 from `twin-room/`; **lead `-1` = the frame that clearly shows two single beds.**
- `room-triple-1..6` ← 6 best from `triple-room/` (prefer `_102_o`).
- `apt2-1..5` ← all 5 from `2-rooms-apartament/` (lead = the brightest living/bedroom).
- `apt3-1..6` ← all 6 from `3-rooms-apartamant/` (all `_105_c` → MAXW=800).
- `exterior-1..8` ← 8 best from `exterior/`. **`exterior-1` should be a strong façade shot** (used in JSON-LD).
- `conference-1..3` ← 3 best from `confference-room/` (all `_105_c` → MAXW=800).
- `bathroom-1..3` ← 3 best from `bathroom/` (prefer `_102_o`).
- `laundry-1` ← the single `laundry/` photo (`_102_o` → MAXW=1280).
- `reception-1..2` ← 2 best from `reception/` (`_102_o` → MAXW=1280).
- Hero source = `exterior/CCBD24B3-624C-4E3F-83E7-BDFC67D8E152_1_102_o.jpeg` (2172×1448, decided in spec).

- [ ] **Step 3: Write the manifest**

Write `<scratchpad>/manifest.md` as a table. Determine `MAXW` per base with the rule: filename contains `_1_102_o` → native ≥1280 → `MAXW=1280`; filename contains `_1_105_c` → native ≈1024 → `MAXW=800`. Example row format:
```
| base            | source (absolute path)                                                        | widths        | MAXW |
|-----------------|-------------------------------------------------------------------------------|---------------|------|
| hero            | .../exterior/CCBD24B3-...-BDFC67D8E152_1_102_o.jpeg                            | 960 1440 1920 | 1920 |
| room-double-1   | .../double-room/8DD49492-...-35EFC3FFCCA4_1_102_o.jpeg                         | 480 800 1280  | 1280 |
| ...             | ...                                                                           | ...           | ...  |
```
Fill a row for every base in the naming convention. This file is the single source of truth for Tasks 3, 7, 10.

- [ ] **Step 4: Verify every manifest source exists**

Run (extract the source column, check each file is readable):
```bash
awk -F'|' 'NR>2 && NF>3 {gsub(/ /,"",$3); print $3}' <scratchpad>/manifest.md | while read -r p; do [ -f "$p" ] || echo "MISSING: $p"; done; echo "done"
```
Expected: only `done` (no `MISSING:` lines). If the `.../` in the table is shorthand, use full absolute paths in the actual file so this check works.

(No commit — scratchpad only.)

---

## Task 3: Generate all image derivatives

**Files:**
- Create: `images/<base>-<width>.{avif,webp}` for every base in the manifest.
- Modify (master): `images/hero.jpg` (overwrite with the chosen hero source).

**Interfaces:**
- Consumes: `manifest.md` (Task 2), `gen-images.sh` (Task 1).
- Produces: all derivative files referenced by Tasks 4–12. Records each base's `MAXW` (already in manifest) — used to build `srcset`s and the `albums` object.

- [ ] **Step 1: Overwrite the hero master**

Run:
```bash
cp /Users/geani/Documents/ramedal-photos/exterior/CCBD24B3-624C-4E3F-83E7-BDFC67D8E152_1_102_o.jpeg /Users/geani/Documents/ramedal-website/images/hero.jpg
```
Expected: no output.

- [ ] **Step 2: Generate the hero derivatives**

Run:
```bash
<scratchpad>/gen-images.sh /Users/geani/Documents/ramedal-website/images/hero.jpg hero 960 1440 1920
```
Expected: `BASE hero MAXW 1920 DIMS 1920x1280`. (2172×1448 → at 1920 wide, height ≈ 1280.)

- [ ] **Step 3: Generate every content base**

For each non-hero row in the manifest, run the helper with widths `480 800 1280`. Example (repeat for all bases — one line each):
```bash
G=<scratchpad>/gen-images.sh
$G /Users/geani/Documents/ramedal-photos/double-room/<lead>.jpeg           room-double-1 480 800 1280
$G /Users/geani/Documents/ramedal-photos/double-room/<n2>.jpeg             room-double-2 480 800 1280
# ... room-double-3..6, room-twin-1..3, room-triple-1..6, apt2-1..5, apt3-1..6,
# ... exterior-1..8, conference-1..3, bathroom-1..3, laundry-1, reception-1..2
```
Expected per line: `BASE <base> MAXW <800|1280> DIMS ...`. Confirm `_105_c` sources print `MAXW 800` and `_102_o` sources print `MAXW 1280`. Cross-check each printed `MAXW` against the manifest; fix the manifest if any differs.

- [ ] **Step 4: Verify the generated set**

Run:
```bash
cd /Users/geani/Documents/ramedal-website
for b in hero room-double-1 room-twin-1 room-triple-1 apt2-1 apt3-1 exterior-1 conference-1 bathroom-1 laundry-1 reception-1; do
  echo "$b: $(ls images/$b-*.avif images/$b-*.webp 2>/dev/null | wc -l | tr -d ' ') files"
done
```
Expected: `hero` = 6 files; each `_102_o`-based lead = 6 files; each `_105_c`-based lead (`apt3-1`, `conference-1`) = 4 files (480+800 × 2 formats). No base shows 0.

- [ ] **Step 5: Commit the images**

```bash
cd /Users/geani/Documents/ramedal-website
git add images/hero.jpg 'images/*.avif' 'images/*.webp'
git commit -m "$(cat <<'EOF'
Add responsive derivatives for professional photo set

Regenerate hero from the branded exterior corner; add AVIF+WebP
variants for room, apartment, exterior, conference, bathroom,
laundry and reception photos.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
)"
```
Expected: commit succeeds; `git status` shows no untracked images.

---

## Task 4: Swap the hero image

**Files:**
- Modify: `index.html:240-244` (the hero `<picture>`), `index.html:243` (`<img>` alt + dims).

**Interfaces:**
- Consumes: `hero-{960,1440,1920}.{avif,webp}` from Task 3.

- [ ] **Step 1: Update the hero `<img>` alt and intrinsic dimensions**

The hero `<picture>` already references `hero-960/1440/1920`. Only the fallback `<img>` needs its alt + dims updated to match the new 3:2 source. Replace line 243:
```html
      <img src="images/hero-1440.webp" alt="Fațada Pensiunii Ramedal Calafat — cazare modernă în centrul orașului" width="1920" height="1280" fetchpriority="high" decoding="async" />
```
(Keep `fetchpriority="high"`, no `loading="lazy"` — hero is the LCP element.)

- [ ] **Step 2: Verify locally**

Run `python3 -m http.server 8000` in the repo root; open `http://localhost:8000`. Expected: hero shows the branded brick-corner exterior with blue sky; no layout shift; dark gradient overlay still legible over the headline.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Use branded exterior as hero image

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 5: Upgrade the About-section images

**Files:**
- Modify: `index.html:280-294` (the three `about-img` `<picture>` blocks).

**Interfaces:**
- Consumes: `exterior-2`, `room-double-2`, `bathroom-1` derivatives (Task 3). (Reuses room/space bases — no new files.)

- [ ] **Step 1: Replace the About images**

Replace lines 280–294 (the `about-img-main` + two `about-img-sub` blocks) with:
```html
      <div class="about-img-main"><picture>
        <source type="image/avif" srcset="images/exterior-2-480.avif 480w, images/exterior-2-800.avif 800w, images/exterior-2-1280.avif 1280w" sizes="(max-width:900px) 92vw, 560px" />
        <source type="image/webp" srcset="images/exterior-2-480.webp 480w, images/exterior-2-800.webp 800w, images/exterior-2-1280.webp 1280w" sizes="(max-width:900px) 92vw, 560px" />
        <img src="images/exterior-2-800.webp" alt="Exteriorul Pensiunii Ramedal Calafat — cazare în centrul orașului" width="1120" height="500" loading="lazy" decoding="async" />
      </picture></div>
      <div class="about-img-sub"><picture>
        <source type="image/avif" srcset="images/room-double-2-480.avif 480w, images/room-double-2-800.avif 800w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 270px" />
        <source type="image/webp" srcset="images/room-double-2-480.webp 480w, images/room-double-2-800.webp 800w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 270px" />
        <img src="images/room-double-2-480.webp" alt="Cameră dublă cu pat matrimonial — Pensiunea Ramedal Calafat" width="540" height="360" loading="lazy" decoding="async" />
      </picture></div>
      <div class="about-img-sub"><picture>
        <source type="image/avif" srcset="images/bathroom-1-480.avif 480w, images/bathroom-1-800.avif 800w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 270px" />
        <source type="image/webp" srcset="images/bathroom-1-480.webp 480w, images/bathroom-1-800.webp 800w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 270px" />
        <img src="images/bathroom-1-480.webp" alt="Baie privată cu duș — Pensiunea Ramedal Calafat" width="540" height="360" loading="lazy" decoding="async" />
      </picture></div>
```
(The `about-badge` div on line 295 stays unchanged. `width`/`height` here are box ratios — CSS fixes the rendered box, so exact values only need a sane ratio.)

- [ ] **Step 2: Verify locally**

Reload `http://localhost:8000`. Expected: About block shows exterior on top, a double room + a bathroom below, badge 9.0★ still overlapping the bottom-right corner; no CLS.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Upgrade About images to professional photos

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 6: Restructure Rooms to 5 clickable-ready cards

Reduce 6 cards → 5, rename per spec (no "Parter"/"Mansardă"), consolidate doubles into one card. This task does the HTML + i18n; wiring the click to the lightbox happens in Task 7.

**Files:**
- Modify: `index.html:329-492` (the six `<article class="room-card">` blocks → five).
- Modify: `index.html` translation objects — `ro` block `r1..r6` region (lines 631–642) and `en` block (lines 704–715); update `f_3bed` (lines 649 / 722).

**Interfaces:**
- Consumes: `room-double-1`, `room-twin-1`, `room-triple-1`, `apt2-1`, `apt3-1` derivatives (Task 3).
- Produces: five `.room-card` articles whose lead `<picture>` uses those bases; i18n keys `r1..r5` (RO+EN) redefined; `r6*` removed.

- [ ] **Step 1: Replace all six room cards with five**

Replace lines 330–491 (the whole run of `<article>`s, keeping the surrounding `<div class="rooms-grid">` on 329 and its close on 492) with these five articles. Note: `data-lb-album` attributes are added now so Task 7 can wire clicks without re-editing this block.

```html
    <article class="room-card" data-lb-album="double" tabindex="0" role="button" aria-label="Cameră Dublă — vezi galeria">
      <div class="room-photo">
        <picture>
          <source type="image/avif" srcset="images/room-double-1-480.avif 480w, images/room-double-1-800.avif 800w, images/room-double-1-1280.avif 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <source type="image/webp" srcset="images/room-double-1-480.webp 480w, images/room-double-1-800.webp 800w, images/room-double-1-1280.webp 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <img src="images/room-double-1-800.webp" alt="Cameră dublă cu pat matrimonial — Pensiunea Ramedal Calafat" width="760" height="440" loading="lazy" decoding="async" />
        </picture>
        <span class="room-tag" data-i18n="r1_tag">Cameră Dublă</span>
      </div>
      <div class="room-body">
        <h3 data-i18n="r1_name">Cameră Dublă</h3>
        <p class="room-size" data-i18n="r1_size">1 pat matrimonial</p>
        <p data-i18n="r1_desc">Cameră dublă modernă cu pat matrimonial, aer condiționat, baie privată și TV cu ecran plat. Lenjerie proaspătă, articole de toaletă gratuite și WiFi de mare viteză.</p>
        <div class="room-features">
          <span class="feat-tag" data-i18n="f_ac">❄️ Aer condiționat</span>
          <span class="feat-tag" data-i18n="f_tv">📺 TV ecran plat</span>
          <span class="feat-tag" data-i18n="f_bath">🚿 Baie privată</span>
          <span class="feat-tag" data-i18n="f_wifi">📶 WiFi gratuit</span>
          <span class="feat-tag" data-i18n="f_toilet">🧴 Toaletă gratuită</span>
          <span class="feat-tag" data-i18n="f_dryer">💨 Uscător păr</span>
        </div>
        <div class="room-footer">
          <span class="room-capacity" data-i18n="cap2">👤 max. 2 persoane</span>
          <a href="#contact" class="room-cta" data-i18n="book_btn">Rezervă</a>
        </div>
      </div>
    </article>

    <article class="room-card" data-lb-album="twin" tabindex="0" role="button" aria-label="Cameră Twin — vezi galeria">
      <div class="room-photo">
        <picture>
          <source type="image/avif" srcset="images/room-twin-1-480.avif 480w, images/room-twin-1-800.avif 800w, images/room-twin-1-1280.avif 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <source type="image/webp" srcset="images/room-twin-1-480.webp 480w, images/room-twin-1-800.webp 800w, images/room-twin-1-1280.webp 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <img src="images/room-twin-1-800.webp" alt="Cameră twin cu două paturi single — Pensiunea Ramedal Calafat" width="760" height="440" loading="lazy" decoding="async" />
        </picture>
        <span class="room-tag" data-i18n="r2_tag">Cameră Twin</span>
      </div>
      <div class="room-body">
        <h3 data-i18n="r2_name">Cameră Twin</h3>
        <p class="room-size" data-i18n="r2_size">2 paturi de o persoană</p>
        <p data-i18n="r2_desc">Cameră twin cu două paturi de o persoană, ideală pentru prieteni sau colegi de lucru. Aer condiționat, baie privată, TV și WiFi gratuit.</p>
        <div class="room-features">
          <span class="feat-tag" data-i18n="f_twin2">🛏️ 2 paturi single</span>
          <span class="feat-tag" data-i18n="f_ac">❄️ Aer condiționat</span>
          <span class="feat-tag" data-i18n="f_bath">🚿 Baie privată</span>
          <span class="feat-tag" data-i18n="f_wifi">📶 WiFi gratuit</span>
          <span class="feat-tag" data-i18n="f_light">🪟 Lumină naturală</span>
        </div>
        <div class="room-footer">
          <span class="room-capacity" data-i18n="cap2">👤 max. 2 persoane</span>
          <a href="#contact" class="room-cta" data-i18n="book_btn">Rezervă</a>
        </div>
      </div>
    </article>

    <article class="room-card" data-lb-album="triple" tabindex="0" role="button" aria-label="Cameră Triplă — vezi galeria">
      <div class="room-photo">
        <picture>
          <source type="image/avif" srcset="images/room-triple-1-480.avif 480w, images/room-triple-1-800.avif 800w, images/room-triple-1-1280.avif 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <source type="image/webp" srcset="images/room-triple-1-480.webp 480w, images/room-triple-1-800.webp 800w, images/room-triple-1-1280.webp 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <img src="images/room-triple-1-800.webp" alt="Cameră triplă pentru familii — Pensiunea Ramedal Calafat" width="760" height="440" loading="lazy" decoding="async" />
        </picture>
        <span class="room-tag" data-i18n="r3_tag">Cameră Triplă</span>
      </div>
      <div class="room-body">
        <h3 data-i18n="r3_name">Cameră Triplă</h3>
        <p class="room-size" data-i18n="r3_size">1 pat dublu + 1 pat individual</p>
        <p data-i18n="r3_desc">Cameră spațioasă și luminoasă, perfectă pentru familii sau grupuri de 3 persoane. Aer condiționat, TV cu ecran plat și baie privată proprie.</p>
        <div class="room-features">
          <span class="feat-tag" data-i18n="f_ac">❄️ Aer condiționat</span>
          <span class="feat-tag" data-i18n="f_tv">📺 TV ecran plat</span>
          <span class="feat-tag" data-i18n="f_bath">🚿 Baie privată</span>
          <span class="feat-tag" data-i18n="f_wifi">📶 WiFi gratuit</span>
          <span class="feat-tag" data-i18n="f_family">👨‍👩‍👧 Cameră familie</span>
        </div>
        <div class="room-footer">
          <span class="room-capacity" data-i18n="cap3">👤 max. 3 persoane</span>
          <a href="#contact" class="room-cta" data-i18n="book_btn">Rezervă</a>
        </div>
      </div>
    </article>

    <article class="room-card" data-lb-album="apt2" tabindex="0" role="button" aria-label="Apartament 2 Camere — vezi galeria">
      <div class="room-photo">
        <picture>
          <source type="image/avif" srcset="images/apt2-1-480.avif 480w, images/apt2-1-800.avif 800w, images/apt2-1-1280.avif 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <source type="image/webp" srcset="images/apt2-1-480.webp 480w, images/apt2-1-800.webp 800w, images/apt2-1-1280.webp 1280w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <img src="images/apt2-1-800.webp" alt="Apartament cu 2 camere — living — Pensiunea Ramedal Calafat" width="760" height="440" loading="lazy" decoding="async" />
        </picture>
        <span class="room-tag" data-i18n="r4_tag">Apartament</span>
      </div>
      <div class="room-body">
        <h3 data-i18n="r4_name">Apartament 2 Camere</h3>
        <p class="room-size" data-i18n="r4_size">Dormitor separat + living</p>
        <p data-i18n="r4_desc">Apartament cu dormitor separat și living propriu, ideal pentru sejururi mai lungi sau cupluri care doresc mai mult spațiu. Aer condiționat, baie privată și WiFi gratuit.</p>
        <div class="room-features">
          <span class="feat-tag" data-i18n="f_living">🛋️ Living separat</span>
          <span class="feat-tag" data-i18n="f_ac">❄️ Aer condiționat</span>
          <span class="feat-tag" data-i18n="f_bath">🚿 Baie privată</span>
          <span class="feat-tag" data-i18n="f_wifi">📶 WiFi gratuit</span>
        </div>
        <div class="room-footer">
          <span class="room-capacity" data-i18n="cap3">👤 max. 3 persoane</span>
          <a href="#contact" class="room-cta" data-i18n="book_btn">Rezervă</a>
        </div>
      </div>
    </article>

    <article class="room-card" data-lb-album="apt3" tabindex="0" role="button" aria-label="Apartament 3 Camere — vezi galeria">
      <div class="room-photo">
        <picture>
          <source type="image/avif" srcset="images/apt3-1-480.avif 480w, images/apt3-1-800.avif 800w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <source type="image/webp" srcset="images/apt3-1-480.webp 480w, images/apt3-1-800.webp 800w" sizes="(max-width:600px) 92vw, (max-width:900px) 45vw, 380px" />
          <img src="images/apt3-1-800.webp" alt="Apartament cu 3 camere — Pensiunea Ramedal Calafat" width="760" height="440" loading="lazy" decoding="async" />
        </picture>
        <span class="room-tag" data-i18n="r5_tag">Apartament</span>
      </div>
      <div class="room-body">
        <h3 data-i18n="r5_name">Apartament 3 Camere</h3>
        <p class="room-size" data-i18n="r5_size">3 camere · pentru grupuri și familii</p>
        <p data-i18n="r5_desc">Cel mai spațios tip de cazare din pensiune. Trei camere separate, ideal pentru familii mari, grupuri sau echipe de business.</p>
        <div class="room-features">
          <span class="feat-tag" data-i18n="f_3bed">🏠 3 camere</span>
          <span class="feat-tag" data-i18n="f_ac">❄️ Aer condiționat</span>
          <span class="feat-tag" data-i18n="f_wifi">📶 WiFi gratuit</span>
          <span class="feat-tag" data-i18n="f_group">👨‍👩‍👧‍👦 Grup/Familie</span>
        </div>
        <div class="room-footer">
          <span class="room-capacity" data-i18n="cap6">👤 max. 6 persoane</span>
          <a href="#contact" class="room-cta" data-i18n="book_btn">Rezervă</a>
        </div>
      </div>
    </article>
```
Note: `apt3-1` uses only 480/800 (source is a `_105_c` crop, MAXW=800), so its `srcset` intentionally omits 1280.

- [ ] **Step 2: Rewrite the `ro` room keys**

In the `ro` object, replace the `r1..r6` region (lines 631–642) with the five-card set:
```javascript
    r1_tag:'Cameră Dublă', r1_name:'Cameră Dublă', r1_size:'1 pat matrimonial',
    r1_desc:'Cameră dublă modernă cu pat matrimonial, aer condiționat, baie privată și TV cu ecran plat. Lenjerie proaspătă, articole de toaletă gratuite și WiFi de mare viteză.',
    r2_tag:'Cameră Twin', r2_name:'Cameră Twin', r2_size:'2 paturi de o persoană',
    r2_desc:'Cameră twin cu două paturi de o persoană, ideală pentru prieteni sau colegi de lucru. Aer condiționat, baie privată, TV și WiFi gratuit.',
    r3_tag:'Cameră Triplă', r3_name:'Cameră Triplă', r3_size:'1 pat dublu + 1 pat individual',
    r3_desc:'Cameră spațioasă și luminoasă, perfectă pentru familii sau grupuri de 3 persoane. Aer condiționat, TV cu ecran plat și baie privată proprie.',
    r4_tag:'Apartament', r4_name:'Apartament 2 Camere', r4_size:'Dormitor separat + living',
    r4_desc:'Apartament cu dormitor separat și living propriu, ideal pentru sejururi mai lungi sau cupluri care doresc mai mult spațiu. Aer condiționat, baie privată și WiFi gratuit.',
    r5_tag:'Apartament', r5_name:'Apartament 3 Camere', r5_size:'3 camere · pentru grupuri și familii',
    r5_desc:'Cel mai spațios tip de cazare din pensiune. Trei camere separate, ideal pentru familii mari, grupuri sau echipe de business.',
```

- [ ] **Step 3: Rewrite the `en` room keys**

In the `en` object, replace the `r1..r6` region (lines 704–715) with:
```javascript
    r1_tag:'Double Room', r1_name:'Double Room', r1_size:'1 double bed',
    r1_desc:'Modern double room with a double bed, air conditioning, private bathroom and flat-screen TV. Fresh linen, complimentary toiletries and high-speed WiFi.',
    r2_tag:'Twin Room', r2_name:'Twin Room', r2_size:'2 single beds',
    r2_desc:'Twin room with two single beds, ideal for friends or work colleagues. Air conditioning, private bathroom, TV and free WiFi.',
    r3_tag:'Triple Room', r3_name:'Triple Room', r3_size:'1 double bed + 1 single bed',
    r3_desc:'Spacious and bright, perfect for families or groups of 3. Air conditioning, flat-screen TV and a private bathroom.',
    r4_tag:'Apartment', r4_name:'2-Room Apartment', r4_size:'Separate bedroom + living room',
    r4_desc:'Apartment with a separate bedroom and private living area, ideal for longer stays or couples wanting extra space. Air conditioning, private bathroom and free WiFi.',
    r5_tag:'Apartment', r5_name:'3-Room Apartment', r5_size:'3 rooms · for groups and families',
    r5_desc:'Our most spacious accommodation. Three separate rooms, ideal for large families, groups or business teams.',
```

- [ ] **Step 4: Update the `f_3bed` label (both languages)**

In `ro` (line 649) change `f_3bed:'🏠 3 dormitoare'` → `f_3bed:'🏠 3 camere'`.
In `en` (line 722) change `f_3bed:'🏠 3 bedrooms'` → `f_3bed:'🏠 3 rooms'`.

- [ ] **Step 5: Verify locally + language toggle**

Reload `http://localhost:8000`. Expected: exactly 5 room cards, order Dublă → Twin → Triplă → Apartament 2 Camere → Apartament 3 Camere; all lead photos load; no "Parter"/"Mansardă" anywhere. Click the EN toggle → every card name/size/desc/tag switches to English; toggle back → Romanian. No console errors.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Consolidate rooms to 5 typed cards with pro photos

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 7: Shared lightbox component + wire room cards

Add the lightbox CSS, the `albums` data object, the lightbox JS, and wire every `.room-card` click/Enter to open its album.

**Files:**
- Modify: `index.html` `<style>` block — add lightbox CSS before `/* RESPONSIVE */` (line 186).
- Modify: `index.html` — add the lightbox markup before `</body>` (line 794), after the FAB link.
- Modify: `index.html` `<script>` — add `albums`, the lightbox functions, and card wiring; add `lb_prev/lb_next/lb_close` to both translation objects.

**Interfaces:**
- Consumes: derivatives from Task 3; `data-lb-album` attributes on `.room-card` (Task 6); `manifest.md` `MAXW` per base (Task 2).
- Produces: global functions `openLightbox(albumKey, index)`, `closeLightbox()`, `lbNav(delta)`; global `albums` object keyed by `double,twin,triple,apt2,apt3,exterior,facilities`, each an array of `{b, mw, alt}`. Task 10 (gallery) consumes `openLightbox` and the same `albums`.

- [ ] **Step 1: Add lightbox CSS**

Insert immediately before `/* RESPONSIVE */` (line 186) in the `<style>` block:
```css
    /* LIGHTBOX */
    .lb{position:fixed;inset:0;z-index:300;display:none;align-items:center;justify-content:center;background:rgba(10,20,22,.92);backdrop-filter:blur(4px)}
    .lb.open{display:flex}
    .lb-stage{position:relative;max-width:92vw;max-height:86vh;display:flex;align-items:center;justify-content:center}
    .lb img{width:auto;height:auto;max-width:92vw;max-height:86vh;object-fit:contain;border-radius:8px;box-shadow:0 12px 48px rgba(0,0,0,.5)}
    .lb-btn{position:absolute;top:50%;transform:translateY(-50%);width:48px;height:48px;border-radius:50%;border:none;background:rgba(255,255,255,.15);color:#fff;font-size:1.5rem;cursor:pointer;display:flex;align-items:center;justify-content:center;transition:background .2s}
    .lb-btn:hover{background:rgba(255,255,255,.3)}
    .lb-prev{left:-64px}
    .lb-next{right:-64px}
    .lb-close{position:absolute;top:-52px;right:0;width:44px;height:44px;border-radius:50%;border:none;background:rgba(255,255,255,.15);color:#fff;font-size:1.4rem;cursor:pointer;transition:background .2s}
    .lb-close:hover{background:rgba(255,255,255,.3)}
    .lb-counter{position:absolute;bottom:-40px;left:50%;transform:translateX(-50%);color:rgba(255,255,255,.8);font-size:.85rem;letter-spacing:.05em}
    .lb-cap{position:absolute;bottom:-40px;right:0;color:rgba(255,255,255,.55);font-size:.78rem;max-width:60%;text-align:right}
    @media(max-width:700px){
      .lb-prev{left:4px}.lb-next{right:4px}
      .lb-btn{background:rgba(0,0,0,.4)}
      .lb-close{top:8px;right:8px}
      .lb-cap{display:none}
    }
```

- [ ] **Step 2: Add the lightbox markup**

Insert after the FAB `</a>` (line 602), before `<script>` (line 604):
```html
<div class="lb" id="lightbox" role="dialog" aria-modal="true" aria-label="Galerie foto" aria-hidden="true">
  <div class="lb-stage">
    <button class="lb-close" id="lbClose" onclick="closeLightbox()" data-i18n-aria="lb_close" aria-label="Închide galeria">✕</button>
    <button class="lb-btn lb-prev" id="lbPrev" onclick="lbNav(-1)" data-i18n-aria="lb_prev" aria-label="Imaginea anterioară">‹</button>
    <picture id="lbPic"><img id="lbImg" src="" alt="" decoding="async" /></picture>
    <button class="lb-btn lb-next" id="lbNext" onclick="lbNav(1)" data-i18n-aria="lb_next" aria-label="Imaginea următoare">›</button>
    <span class="lb-counter" id="lbCounter"></span>
    <span class="lb-cap" id="lbCap"></span>
  </div>
</div>
```

- [ ] **Step 3: Add the `albums` data object**

Insert into the `<script>` right after the `translations` object closes (after line 753, before `let currentLang`). Fill each entry from the manifest: `b`=base, `mw`=MAXW (800 or 1280), `alt`=Romanian alt text. Leads first. Example (complete the arrays for every base generated in Task 3):
```javascript
// ── ALBUME LIGHTBOX ──
const albums = {
  double: [
    {b:'room-double-1',mw:1280,alt:'Cameră dublă cu pat matrimonial — Pensiunea Ramedal Calafat'},
    {b:'room-double-2',mw:1280,alt:'Cameră dublă — Pensiunea Ramedal Calafat'},
    {b:'room-double-3',mw:1280,alt:'Cameră dublă — Pensiunea Ramedal Calafat'},
    {b:'room-double-4',mw:1280,alt:'Cameră dublă — Pensiunea Ramedal Calafat'},
    {b:'room-double-5',mw:1280,alt:'Cameră dublă — Pensiunea Ramedal Calafat'},
    {b:'room-double-6',mw:1280,alt:'Cameră dublă — Pensiunea Ramedal Calafat'}
  ],
  twin: [
    {b:'room-twin-1',mw:1280,alt:'Cameră twin cu două paturi single — Pensiunea Ramedal Calafat'},
    {b:'room-twin-2',mw:800,alt:'Cameră twin — Pensiunea Ramedal Calafat'},
    {b:'room-twin-3',mw:800,alt:'Cameră twin — Pensiunea Ramedal Calafat'}
  ],
  triple: [
    {b:'room-triple-1',mw:1280,alt:'Cameră triplă pentru familii — Pensiunea Ramedal Calafat'},
    {b:'room-triple-2',mw:1280,alt:'Cameră triplă — Pensiunea Ramedal Calafat'},
    {b:'room-triple-3',mw:1280,alt:'Cameră triplă — Pensiunea Ramedal Calafat'},
    {b:'room-triple-4',mw:1280,alt:'Cameră triplă — Pensiunea Ramedal Calafat'},
    {b:'room-triple-5',mw:1280,alt:'Cameră triplă — Pensiunea Ramedal Calafat'},
    {b:'room-triple-6',mw:1280,alt:'Cameră triplă — Pensiunea Ramedal Calafat'}
  ],
  apt2: [
    {b:'apt2-1',mw:1280,alt:'Apartament cu 2 camere — living — Pensiunea Ramedal Calafat'},
    {b:'apt2-2',mw:1280,alt:'Apartament cu 2 camere — Pensiunea Ramedal Calafat'},
    {b:'apt2-3',mw:1280,alt:'Apartament cu 2 camere — Pensiunea Ramedal Calafat'},
    {b:'apt2-4',mw:800,alt:'Apartament cu 2 camere — Pensiunea Ramedal Calafat'},
    {b:'apt2-5',mw:800,alt:'Apartament cu 2 camere — Pensiunea Ramedal Calafat'}
  ],
  apt3: [
    {b:'apt3-1',mw:800,alt:'Apartament cu 3 camere — Pensiunea Ramedal Calafat'},
    {b:'apt3-2',mw:800,alt:'Apartament cu 3 camere — Pensiunea Ramedal Calafat'},
    {b:'apt3-3',mw:800,alt:'Apartament cu 3 camere — Pensiunea Ramedal Calafat'},
    {b:'apt3-4',mw:800,alt:'Apartament cu 3 camere — Pensiunea Ramedal Calafat'},
    {b:'apt3-5',mw:800,alt:'Apartament cu 3 camere — Pensiunea Ramedal Calafat'},
    {b:'apt3-6',mw:800,alt:'Apartament cu 3 camere — Pensiunea Ramedal Calafat'}
  ],
  exterior: [
    {b:'exterior-1',mw:1280,alt:'Fațada Pensiunii Ramedal Calafat'},
    {b:'exterior-2',mw:1280,alt:'Exteriorul Pensiunii Ramedal Calafat'},
    {b:'exterior-3',mw:1280,alt:'Exteriorul Pensiunii Ramedal Calafat'},
    {b:'exterior-4',mw:1280,alt:'Exteriorul Pensiunii Ramedal Calafat'},
    {b:'exterior-5',mw:1280,alt:'Exteriorul Pensiunii Ramedal Calafat'},
    {b:'exterior-6',mw:1280,alt:'Exteriorul Pensiunii Ramedal Calafat'},
    {b:'exterior-7',mw:1280,alt:'Exteriorul Pensiunii Ramedal Calafat'},
    {b:'exterior-8',mw:1280,alt:'Exteriorul Pensiunii Ramedal Calafat'}
  ],
  facilities: [
    {b:'conference-1',mw:800,alt:'Sală de conferințe și evenimente — Pensiunea Ramedal Calafat'},
    {b:'conference-2',mw:800,alt:'Sală de conferințe — Pensiunea Ramedal Calafat'},
    {b:'conference-3',mw:800,alt:'Sală de evenimente — Pensiunea Ramedal Calafat'},
    {b:'bathroom-1',mw:1280,alt:'Baie privată cu duș — Pensiunea Ramedal Calafat'},
    {b:'bathroom-2',mw:1280,alt:'Baie privată — Pensiunea Ramedal Calafat'},
    {b:'bathroom-3',mw:1280,alt:'Baie privată — Pensiunea Ramedal Calafat'},
    {b:'laundry-1',mw:1280,alt:'Spălătorie pentru oaspeți — Pensiunea Ramedal Calafat'},
    {b:'reception-1',mw:1280,alt:'Recepția Pensiunii Ramedal Calafat'},
    {b:'reception-2',mw:1280,alt:'Recepția Pensiunii Ramedal Calafat'}
  ]
};
```
**Set each `mw` to the actual `MAXW` printed by Task 3 for that base** (800 for `_105_c` sources, 1280 for `_102_o`). Trim arrays to the bases you actually generated.

- [ ] **Step 4: Add the lightbox functions**

Insert into the `<script>` after `applyLang` (after line 785). This builds the `<picture>` srcset from `mw`, handles keyboard, focus-trap and scroll-lock:
```javascript
// ── LIGHTBOX ──
let lbAlbum = [], lbIndex = 0, lbTrigger = null;
function lbSrcset(entry, ext){
  const ws = [480,800,1280].filter(w => w <= entry.mw);
  return ws.map(w => `images/${entry.b}-${w}.${ext} ${w}w`).join(', ');
}
function lbRender(){
  const e = lbAlbum[lbIndex];
  const pic = document.getElementById('lbPic');
  pic.innerHTML =
    `<source type="image/avif" srcset="${lbSrcset(e,'avif')}" sizes="92vw" />` +
    `<source type="image/webp" srcset="${lbSrcset(e,'webp')}" sizes="92vw" />` +
    `<img id="lbImg" src="images/${e.b}-${e.mw}.webp" alt="${e.alt}" decoding="async" />`;
  document.getElementById('lbCounter').textContent = (lbIndex+1) + ' / ' + lbAlbum.length;
  document.getElementById('lbCap').textContent = e.alt;
}
function openLightbox(albumKey, index){
  if(!albums[albumKey]) return;
  lbAlbum = albums[albumKey]; lbIndex = index || 0;
  lbTrigger = document.activeElement;
  lbRender();
  const lb = document.getElementById('lightbox');
  lb.classList.add('open'); lb.setAttribute('aria-hidden','false');
  document.body.style.overflow = 'hidden';
  document.getElementById('lbClose').focus();
}
function closeLightbox(){
  const lb = document.getElementById('lightbox');
  lb.classList.remove('open'); lb.setAttribute('aria-hidden','true');
  document.body.style.overflow = '';
  if(lbTrigger && lbTrigger.focus) lbTrigger.focus();
}
function lbNav(delta){
  lbIndex = (lbIndex + delta + lbAlbum.length) % lbAlbum.length;
  lbRender();
}
// close on backdrop click (not when clicking the image/controls)
document.getElementById('lightbox').addEventListener('click', e => {
  if(e.target.id === 'lightbox') closeLightbox();
});
// keyboard
document.addEventListener('keydown', e => {
  if(!document.getElementById('lightbox').classList.contains('open')) return;
  if(e.key === 'Escape') closeLightbox();
  else if(e.key === 'ArrowLeft') lbNav(-1);
  else if(e.key === 'ArrowRight') lbNav(1);
  else if(e.key === 'Tab'){ // focus-trap: keep focus on the three buttons
    const f = [document.getElementById('lbClose'),document.getElementById('lbPrev'),document.getElementById('lbNext')];
    if(!f.includes(document.activeElement)){ e.preventDefault(); f[0].focus(); }
  }
});
// wire room cards
document.querySelectorAll('.room-card[data-lb-album]').forEach(card => {
  const key = card.getAttribute('data-lb-album');
  const open = e => {
    if(e.target.closest('.room-cta')) return; // let the "Rezervă" link work
    openLightbox(key, 0);
  };
  card.addEventListener('click', open);
  card.addEventListener('keydown', e => { if(e.key==='Enter'||e.key===' '){ e.preventDefault(); openLightbox(key,0); } });
});
```

- [ ] **Step 5: Localize the lightbox aria-labels**

The buttons use `data-i18n-aria`. Extend `applyLang` to translate them. In `applyLang` (after the `[data-i18n]` loop, before the toggle-button update, ~line 771) add:
```javascript
  document.querySelectorAll('[data-i18n-aria]').forEach(el => {
    const key = el.getAttribute('data-i18n-aria');
    if (t[key] !== undefined) el.setAttribute('aria-label', t[key]);
  });
```
Add keys to `ro` (near the room keys):
```javascript
    lb_prev:'Imaginea anterioară', lb_next:'Imaginea următoare', lb_close:'Închide galeria',
```
Add keys to `en`:
```javascript
    lb_prev:'Previous image', lb_next:'Next image', lb_close:'Close gallery',
```

- [ ] **Step 6: Verify locally**

Reload. Expected:
- Clicking any room card opens the lightbox on that type's first photo; the ✕, ‹, › and "1 / N" counter show.
- ‹ › (and ← →) cycle within the album and wrap around; counter updates.
- ✕, Esc, and backdrop click all close; focus returns to the card.
- Clicking "Rezervă" inside a card does NOT open the lightbox (it jumps to #contact).
- Only on open does the network tab show the 1280/800 image request (nothing on page load).
- Toggle to EN: focus the lightbox buttons — `aria-label`s are English (inspect element).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add shared lightbox and open it from room cards

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 8: Conference / Events section

**Files:**
- Modify: `index.html` — insert a new `<section id="conferinte">` after `#facilitati` (after line 517), before `#locatie` (line 519).
- Modify: translation objects — add `conf_*` keys to `ro` and `en`.

**Interfaces:**
- Consumes: `conference-1..3` derivatives (Task 3).

- [ ] **Step 1: Add the section markup**

Insert between `</section>` (line 517, end of facilities) and `<!-- LOCATIE -->` (line 519):
```html
<!-- CONFERINTE -->
<section class="conference" id="conferinte">
  <div class="conference-inner">
    <div class="conference-text">
      <span class="section-label" data-i18n="conf_label">Evenimente & Business</span>
      <h2 class="section-title" data-i18n="conf_h2">Sală de Conferințe și Evenimente</h2>
      <div class="divider"></div>
      <p data-i18n="conf_p1">Pensiunea Ramedal dispune de o sală spațioasă pentru conferințe, ședințe și evenimente în Calafat — potrivită pentru întâlniri de business, traininguri și ocazii private.</p>
      <p data-i18n="conf_p2">Spațiul poate fi aranjat în funcție de nevoile tale, cu WiFi gratuit și acces facil din centrul orașului. Contactează-ne pentru disponibilitate și detalii.</p>
      <a href="#contact" class="directions-btn" data-i18n="conf_cta">Cere ofertă</a>
    </div>
    <div class="conference-photo">
      <picture>
        <source type="image/avif" srcset="images/conference-1-480.avif 480w, images/conference-1-800.avif 800w" sizes="(max-width:900px) 92vw, 520px" />
        <source type="image/webp" srcset="images/conference-1-480.webp 480w, images/conference-1-800.webp 800w" sizes="(max-width:900px) 92vw, 520px" />
        <img src="images/conference-1-800.webp" alt="Sală de conferințe și evenimente — Pensiunea Ramedal Calafat" width="1040" height="694" loading="lazy" decoding="async" />
      </picture>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Add section CSS**

Insert in the `<style>` block after the `/* FACILITATI */` rules (after line 133, before `/* LOCATIE */`):
```css
    /* CONFERINTE */
    .conference{background:var(--bg);padding:5rem 5%}
    .conference-inner{max-width:1100px;margin:0 auto;display:grid;grid-template-columns:1fr 1fr;gap:3.5rem;align-items:center}
    .conference-text p{color:var(--text-soft);margin-bottom:1rem}
    .conference-text .directions-btn{margin-top:.5rem}
    .conference-photo{border-radius:16px;overflow:hidden;box-shadow:var(--shadow-lg);cursor:pointer}
    .conference-photo img{width:100%;height:340px;object-fit:cover;transition:transform .4s}
    .conference-photo:hover img{transform:scale(1.04)}
    @media(max-width:900px){.conference-inner{grid-template-columns:1fr;gap:2rem}}
```

- [ ] **Step 3: Make the conference photo open the facilities album**

The `facilities` album (Task 7) leads with `conference-1`. Add a click handler. In the card-wiring area of the `<script>` (Task 7 Step 4), append:
```javascript
const confPhoto = document.querySelector('.conference-photo');
if(confPhoto) confPhoto.addEventListener('click', () => openLightbox('facilities', 0));
```

- [ ] **Step 4: Add `conf_*` translation keys**

`ro`:
```javascript
    conf_label:'Evenimente & Business', conf_h2:'Sală de Conferințe și Evenimente',
    conf_p1:'Pensiunea Ramedal dispune de o sală spațioasă pentru conferințe, ședințe și evenimente în Calafat — potrivită pentru întâlniri de business, traininguri și ocazii private.',
    conf_p2:'Spațiul poate fi aranjat în funcție de nevoile tale, cu WiFi gratuit și acces facil din centrul orașului. Contactează-ne pentru disponibilitate și detalii.',
    conf_cta:'Cere ofertă',
```
`en`:
```javascript
    conf_label:'Events & Business', conf_h2:'Conference & Events Hall',
    conf_p1:'Pensiunea Ramedal has a spacious hall for conferences, meetings and events in Calafat — suitable for business meetings, training sessions and private occasions.',
    conf_p2:'The space can be arranged to suit your needs, with free WiFi and easy access from the city centre. Contact us for availability and details.',
    conf_cta:'Request a quote',
```

- [ ] **Step 5: Verify locally**

Reload. Expected: a Conference section appears between Facilities and Location; two-column on desktop, stacked on mobile; clicking the photo opens the lightbox on `conference-1` and ‹ › cycle through conference→bathroom→laundry→reception; EN toggle translates the heading/paragraphs/CTA.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add conference/events section

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 9: Self-service Laundry facility tile

**Files:**
- Modify: `index.html:515` — add a 13th `.facility` tile after `fac12`.
- Modify: translation objects — add `fac13_title` / `fac13_desc` to `ro` and `en`.

**Interfaces:** none new (pure content).

- [ ] **Step 1: Add the tile**

After the `fac12` tile (line 515), before `</div>` (line 516), insert:
```html
    <div class="facility"><div class="facility-icon">🧺</div><h4 data-i18n="fac13_title">Spălătorie</h4><p data-i18n="fac13_desc">Spălătorie self-service pentru oaspeți, ideală pentru sejururi mai lungi și tranzit</p></div>
```

- [ ] **Step 2: Add translation keys**

`ro`: `fac13_title:'Spălătorie', fac13_desc:'Spălătorie self-service pentru oaspeți, ideală pentru sejururi mai lungi și tranzit',`
`en`: `fac13_title:'Laundry', fac13_desc:'Self-service guest laundry, ideal for longer stays and transit',`

- [ ] **Step 3: Verify + commit**

Reload → 13 facility tiles, laundry 🧺 present, EN toggle works. Then:
```bash
git add index.html
git commit -m "Add self-service laundry facility

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 10: Filterable Gallery section

**Files:**
- Modify: `index.html` — insert `<section id="galerie">` after `#conferinte`, before `#locatie`.
- Modify: `<style>` — add gallery CSS.
- Modify: `<script>` — add filter logic; add `gal_*` keys to both languages.

**Interfaces:**
- Consumes: room/exterior/facilities derivatives (Task 3); `openLightbox` + `albums` (Task 7).

- [ ] **Step 1: Add the section markup**

Insert between the conference `</section>` (Task 8) and `<!-- LOCATIE -->`. Each `.gallery-item` carries `data-cat` (for filtering), `data-album` + `data-index` (for the lightbox). Rooms items each point at their type's album index 0; exterior/facilities items index into their shared album:
```html
<!-- GALERIE -->
<section class="gallery" id="galerie">
  <div class="section-header">
    <span class="section-label" data-i18n="gal_label">Galerie foto</span>
    <h2 class="section-title" data-i18n="gal_h2">Galerie Foto Pensiunea Ramedal</h2>
    <div class="divider"></div>
    <p class="section-sub" data-i18n="gal_sub">Explorează camerele, exteriorul și facilitățile pensiunii noastre din Calafat.</p>
  </div>
  <div class="gallery-filters">
    <button class="gal-filter active" data-filter="all" onclick="galFilter('all',this)" data-i18n="gal_all">Toate</button>
    <button class="gal-filter" data-filter="rooms" onclick="galFilter('rooms',this)" data-i18n="gal_rooms">Camere</button>
    <button class="gal-filter" data-filter="exterior" onclick="galFilter('exterior',this)" data-i18n="gal_exterior">Exterior</button>
    <button class="gal-filter" data-filter="fac" onclick="galFilter('fac',this)" data-i18n="gal_fac">Facilități</button>
  </div>
  <div class="gallery-grid" id="galleryGrid">
    <!-- Rooms: one thumbnail per type, opens that type's album -->
    <button class="gallery-item" data-cat="rooms" data-album="double" data-index="0"><picture><source type="image/avif" srcset="images/room-double-1-480.avif 480w, images/room-double-1-800.avif 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><source type="image/webp" srcset="images/room-double-1-480.webp 480w, images/room-double-1-800.webp 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><img src="images/room-double-1-480.webp" alt="Cameră dublă — Pensiunea Ramedal Calafat" width="400" height="300" loading="lazy" decoding="async" /></picture></button>
    <button class="gallery-item" data-cat="rooms" data-album="twin" data-index="0"><picture><source type="image/avif" srcset="images/room-twin-1-480.avif 480w, images/room-twin-1-800.avif 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><source type="image/webp" srcset="images/room-twin-1-480.webp 480w, images/room-twin-1-800.webp 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><img src="images/room-twin-1-480.webp" alt="Cameră twin — Pensiunea Ramedal Calafat" width="400" height="300" loading="lazy" decoding="async" /></picture></button>
    <button class="gallery-item" data-cat="rooms" data-album="triple" data-index="0"><picture><source type="image/avif" srcset="images/room-triple-1-480.avif 480w, images/room-triple-1-800.avif 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><source type="image/webp" srcset="images/room-triple-1-480.webp 480w, images/room-triple-1-800.webp 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><img src="images/room-triple-1-480.webp" alt="Cameră triplă — Pensiunea Ramedal Calafat" width="400" height="300" loading="lazy" decoding="async" /></picture></button>
    <button class="gallery-item" data-cat="rooms" data-album="apt2" data-index="0"><picture><source type="image/avif" srcset="images/apt2-1-480.avif 480w, images/apt2-1-800.avif 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><source type="image/webp" srcset="images/apt2-1-480.webp 480w, images/apt2-1-800.webp 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><img src="images/apt2-1-480.webp" alt="Apartament cu 2 camere — Pensiunea Ramedal Calafat" width="400" height="300" loading="lazy" decoding="async" /></picture></button>
    <button class="gallery-item" data-cat="rooms" data-album="apt3" data-index="0"><picture><source type="image/avif" srcset="images/apt3-1-480.avif 480w, images/apt3-1-800.avif 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><source type="image/webp" srcset="images/apt3-1-480.webp 480w, images/apt3-1-800.webp 800w" sizes="(max-width:600px)45vw,(max-width:900px)30vw,220px" /><img src="images/apt3-1-480.webp" alt="Apartament cu 3 camere — Pensiunea Ramedal Calafat" width="400" height="300" loading="lazy" decoding="async" /></picture></button>
    <!-- Exterior: 8 thumbnails, index into the exterior album -->
    <!-- Facilities: 9 thumbnails, index into the facilities album -->
  </div>
</section>
```
Then add the **exterior** items (8) — one per `exterior-N` with `data-cat="exterior" data-album="exterior" data-index="N-1"` — and the **facilities** items (9) with `data-cat="fac" data-album="facilities" data-index="N-1"`, in the same order as the `albums.facilities` array (conference 0–2, bathroom 3–5, laundry 6, reception 7–8). Use the exact `<button class="gallery-item">…</button>` structure above, swapping the base name, alt, `data-album`, and `data-index`. Build these from the manifest.

- [ ] **Step 2: Add gallery CSS**

Insert in `<style>` after the conference rules:
```css
    /* GALERIE */
    .gallery{background:var(--white);padding:5rem 5%}
    .gallery-filters{display:flex;flex-wrap:wrap;gap:.6rem;justify-content:center;margin-bottom:2rem}
    .gal-filter{background:transparent;border:1.5px solid var(--border);color:var(--text-soft);padding:.45rem 1.15rem;border-radius:50px;font-size:.82rem;font-weight:600;cursor:pointer;font-family:'Inter',sans-serif;transition:all .2s}
    .gal-filter:hover{border-color:var(--teal);color:var(--teal)}
    .gal-filter.active{background:var(--teal);border-color:var(--teal);color:#fff}
    .gallery-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(200px,1fr));gap:.9rem;max-width:1200px;margin:0 auto}
    .gallery-item{padding:0;border:none;background:none;cursor:pointer;border-radius:12px;overflow:hidden;box-shadow:var(--shadow);aspect-ratio:4/3}
    .gallery-item img{width:100%;height:100%;object-fit:cover;transition:transform .4s}
    .gallery-item:hover img{transform:scale(1.06)}
    .gallery-item.hidden{display:none}
    @media(max-width:600px){.gallery-grid{grid-template-columns:repeat(2,1fr)}}
```

- [ ] **Step 3: Add gallery JS (filter + open lightbox)**

Append to the `<script>` (near the card wiring):
```javascript
// ── GALERIE ──
function galFilter(cat, btn){
  document.querySelectorAll('.gal-filter').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
  document.querySelectorAll('.gallery-item').forEach(it => {
    it.classList.toggle('hidden', cat !== 'all' && it.getAttribute('data-cat') !== cat);
  });
}
document.querySelectorAll('.gallery-item').forEach(it => {
  it.addEventListener('click', () => {
    openLightbox(it.getAttribute('data-album'), parseInt(it.getAttribute('data-index'),10) || 0);
  });
});
```

- [ ] **Step 4: Add `gal_*` translation keys**

`ro`:
```javascript
    gal_label:'Galerie foto', gal_h2:'Galerie Foto Pensiunea Ramedal',
    gal_sub:'Explorează camerele, exteriorul și facilitățile pensiunii noastre din Calafat.',
    gal_all:'Toate', gal_rooms:'Camere', gal_exterior:'Exterior', gal_fac:'Facilități',
```
`en`:
```javascript
    gal_label:'Photo Gallery', gal_h2:'Pensiunea Ramedal Photo Gallery',
    gal_sub:'Explore the rooms, exterior and facilities of our guesthouse in Calafat.',
    gal_all:'All', gal_rooms:'Rooms', gal_exterior:'Exterior', gal_fac:'Facilities',
```

- [ ] **Step 5: Verify locally**

Reload. Expected: Gallery section with 4 filter chips; "Toate" active shows all (~22) thumbnails; clicking "Camere"/"Exterior"/"Facilități" shows only that set; a clicked thumbnail opens the lightbox on the correct album+photo, and ‹ › cycle that album; EN toggle translates chips + heading + sub.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add filterable photo gallery

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 11: Add "Galerie" to navigation

**Files:**
- Modify: `index.html:211-217` (`.nav-links`) and `index.html:228-235` (`.mobile-menu`).
- Modify: translation objects — `nav_gallery` (both languages).

- [ ] **Step 1: Add the desktop nav link**

In `.nav-links`, after the Facilities `<li>` (line 214), insert:
```html
    <li><a href="#galerie" data-i18n="nav_gallery">Galerie</a></li>
```

- [ ] **Step 2: Add the mobile menu link**

In `.mobile-menu`, after the Facilities link (line 231), insert:
```html
  <a href="#galerie" onclick="closeMenu()" data-i18n="nav_gallery">Galerie</a>
```

- [ ] **Step 3: Add translation key**

`ro`: `nav_gallery:'Galerie',`  ·  `en`: `nav_gallery:'Gallery',`

- [ ] **Step 4: Verify + commit**

Reload → "Galerie" appears in the desktop nav and (at ≤900px) the mobile menu; clicking scrolls to the gallery; EN toggle shows "Gallery". Then:
```bash
git add index.html
git commit -m "Add Galerie link to navigation

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 12: SEO — extend JSON-LD with images + amenities

**Files:**
- Modify: `index.html:24` (the single-line JSON-LD `LodgingBusiness`).

**Interfaces:**
- Consumes: derivative filenames from Task 3 (absolute URLs under `https://ramedal.clft.ro/images/`).

- [ ] **Step 1: Add `image` and `amenityFeature` to the JSON-LD**

Edit the JSON on line 24, inserting two new properties before `"checkinTime"`. Use `-1280` URLs for `_102_o`-based bases and `-800` for `_105_c`-based ones (match the manifest MAXW). Final JSON (single line — the diff shows it expanded for readability, but keep it one line):
```json
"image":["https://ramedal.clft.ro/images/hero-1440.webp","https://ramedal.clft.ro/images/room-double-1-1280.webp","https://ramedal.clft.ro/images/room-triple-1-1280.webp","https://ramedal.clft.ro/images/apt3-1-800.webp","https://ramedal.clft.ro/images/exterior-1-1280.webp","https://ramedal.clft.ro/images/conference-1-800.webp"],
"amenityFeature":[{"@type":"LocationFeatureSpecification","name":"Free WiFi","value":true},{"@type":"LocationFeatureSpecification","name":"Free parking","value":true},{"@type":"LocationFeatureSpecification","name":"Air conditioning","value":true},{"@type":"LocationFeatureSpecification","name":"Non-smoking rooms","value":true},{"@type":"LocationFeatureSpecification","name":"Self-service laundry","value":true},{"@type":"LocationFeatureSpecification","name":"Conference/events hall","value":true}],
```
So the object becomes `…,"aggregateRating":{…},"image":[…],"amenityFeature":[…],"checkinTime":"14:00","checkoutTime":"12:00"}`.

- [ ] **Step 2: Validate the JSON-LD**

Run:
```bash
cd /Users/geani/Documents/ramedal-website
python3 -c "import re,json,sys; h=open('index.html').read(); m=re.search(r'application/ld\+json\">(.*?)</script>', h, re.S); json.loads(m.group(1)); print('JSON-LD OK')"
```
Expected: `JSON-LD OK` (no exception). Also confirm every image URL's file exists:
```bash
for u in hero-1440 room-double-1-1280 room-triple-1-1280 apt3-1-800 exterior-1-1280 conference-1-800; do ls images/$u.webp >/dev/null && echo "$u ok" || echo "$u MISSING"; done
```
Expected: all `ok`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Extend JSON-LD with image gallery and amenities

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 13: Font glyph-coverage check

The subset fonts were built from the old `index.html` text. New copy uses only standard Romanian letters, digits and punctuation, so no new glyphs are expected. **`pyftsubset`/`fonttools` are NOT installed and the variable masters are not in the repo** — so if a new glyph IS found, re-subsetting is blocked and must be reported, not silently skipped.

**Files:** none modified (unless a new glyph is found).

- [ ] **Step 1: Diff the glyph set of new vs. old text**

Run (compares characters in the current `index.html` against the last committed-before-this-branch version; any char present now but absent then is a candidate new glyph):
```bash
cd /Users/geani/Documents/ramedal-website
python3 - <<'PY'
import re,subprocess
def chars(s):
    # strip tags, keep visible + attribute text, drop emoji (rendered by system font)
    t = re.sub(r'<[^>]+>',' ', s)
    return set(c for c in t if c.isprintable() and ord(c) < 0x2500)
now = chars(open('index.html').read())
old = chars(subprocess.run(['git','show','master:index.html'],capture_output=True,text=True).stdout)
new = sorted(now - old)
print('NEW GLYPHS:', ''.join(new) if new else '(none)')
PY
```
Expected: `NEW GLYPHS: (none)`.

- [ ] **Step 2: Act on the result**

- If `(none)`: no action; fonts already cover all text. Done.
- If any glyph is listed: **stop and report to the user** — re-subsetting needs `pip install fonttools` **and** the variable font masters (not in the repo). Do not ship a broken subset and do not add a Google Fonts request (violates the self-hosting constraint). Surface the exact missing character(s) and ask how to proceed.

(No commit unless fonts change.)

---

## Task 14: Clean up orphaned derivatives + final verification

**Files:**
- Delete: unreferenced `.avif`/`.webp` derivatives of retired bases. **Never touch `.jpg` masters.**

- [ ] **Step 1: Confirm the retired bases are no longer referenced**

Run:
```bash
cd /Users/geani/Documents/ramedal-website
for b in corridor room-double-teal room-double-green room-triple room-attic room-studio room-attic-studio bathroom; do
  n=$(grep -c "images/$b-" index.html || true)
  echo "$b: $n references"
done
```
Expected: every line `: 0 references`. (Note `bathroom` here is the OLD base; the new photos are `bathroom-1..3`, which won't match `bathroom-` followed by a digit? They DO — `bathroom-1` matches `bathroom-`. So instead check the old base precisely below.)

Correction — check the retired bases with a digit-free anchor so `bathroom-1` is not counted:
```bash
for b in corridor room-double-teal room-double-green room-triple room-attic room-studio room-attic-studio; do
  echo "$b: $(grep -c "images/$b-" index.html || true)"
done
echo "old bathroom (480/800 only): $(grep -Ec 'images/bathroom-(480|800)\.' index.html || true)"
```
Expected: all `0`.

- [ ] **Step 2: Delete orphaned derivatives (masters preserved)**

Run:
```bash
cd /Users/geani/Documents/ramedal-website/images
rm -f corridor-*.avif corridor-*.webp \
      room-double-teal-*.avif room-double-teal-*.webp \
      room-double-green-*.avif room-double-green-*.webp \
      room-triple-*.avif room-triple-*.webp \
      room-attic-*.avif room-attic-*.webp \
      room-studio-*.avif room-studio-*.webp \
      room-attic-studio-*.avif room-attic-studio-*.webp \
      bathroom-480.avif bathroom-480.webp bathroom-800.avif bathroom-800.webp
ls *.jpg | wc -l
```
Expected: the `.jpg` master count is unchanged (15). Verify `room-triple-*` deletion did NOT catch new `room-triple-1..6` — those are separate files `room-triple-1-480.webp` etc. **Important:** `room-triple-*` glob WOULD match `room-triple-1-480.webp`. Do NOT use the bare `room-triple-*` glob. Instead delete the old triple derivatives explicitly:
```bash
rm -f room-triple-480.avif room-triple-480.webp room-triple-800.avif room-triple-800.webp room-triple-1280.avif room-triple-1280.webp
```
and remove `room-triple-*.avif room-triple-*.webp` from the batch above. (Same care applies only to `room-triple`, since it's the one retired base that is a prefix of a new base. All other retired bases have no new-base prefix collision.)

- [ ] **Step 3: Confirm nothing referenced is missing (no broken images)**

Run — extract every `images/...` reference from `index.html` and assert each file exists:
```bash
cd /Users/geani/Documents/ramedal-website
grep -oE 'images/[a-zA-Z0-9._-]+\.(avif|webp|jpg|png)' index.html | sort -u | while read -r f; do [ -f "$f" ] || echo "MISSING: $f"; done; echo "check done"
```
Expected: only `check done` (no `MISSING:` lines).

- [ ] **Step 4: Full manual verification pass (serve + browse)**

With `python3 -m http.server 8000` running, in the browser confirm the spec's test checklist:
- Hero is the branded exterior and loads immediately (LCP); everything else lazy-loads on scroll.
- 5 room cards; each opens its lightbox; ‹ › / ← → / Esc / backdrop / ✕ all work; focus returns to trigger.
- Conference section present and its photo opens the facilities album.
- 13 facility tiles incl. laundry.
- Gallery: 4 filters work; thumbnails open the correct album+index.
- "Galerie" in nav + mobile menu scrolls correctly.
- Language toggle RO↔EN translates ALL new text (rooms, conference, gallery filters, nav, facility, lightbox aria).
- No horizontal scroll; no visible CLS; browser console shows no errors.

- [ ] **Step 5: Commit the cleanup**

```bash
cd /Users/geani/Documents/ramedal-website
git add -A images
git commit -m "Remove orphaned derivatives of retired image bases

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

- [ ] **Step 6: Report completion**

Summarize to the user: sections changed, image count added/removed, and that `master`/production is untouched (still on branch `photos-integration`), ready for their review before any merge/deploy.

---

## Self-Review

**Spec coverage:**
- Hero swap → Task 4. About upgrade → Task 5. Rooms 6→5 + rename + clickable → Tasks 6+7. Lightbox (data-driven, load-on-open, ◀▶/counter/✕, ESC/arrows, focus-trap, aria-modal, scroll-lock, CSS override) → Task 7. Conference section → Task 8. Laundry tile → Task 9. Filterable gallery → Task 10. Nav "Galerie" → Task 11. SEO JSON-LD image+amenityFeature → Task 12. Font re-subset check → Task 13. Naming convention + cleanup of retired bases → Tasks 3+14. i18n (all new keys RO+EN) → woven through Tasks 6–11. Image pipeline + speed budget (lazy, CLS, load-on-open) → Tasks 1–3 + per-section tasks. Alt-text RO-only, twin lead-only-swap, out-of-scope items → respected (no tasks contradict them). All spec sections map to a task. ✅
- Spec "assumptions to confirm" (conference naming, card order, per-photo selection) were confirmed by the user ("Mergem maestre") and are locked in Tasks 2/6/8.

**Placeholder scan:** No "TBD/handle appropriately". Judgment-based photo selection is isolated in Task 2 which produces a concrete manifest artifact consumed downstream (a real task interface, not a hidden placeholder). `mw`/source paths are marked "fill from manifest/Task 3 output" — a genuine cross-task interface, with the rule for deriving each value stated explicitly (`_102_o`→1280, `_105_c`→800).

**Type/name consistency:** `openLightbox(albumKey,index)`, `closeLightbox()`, `lbNav(delta)`, `galFilter(cat,btn)`, global `albums` with keys `double/twin/triple/apt2/apt3/exterior/facilities`, entry shape `{b,mw,alt}`, and DOM ids (`lightbox,lbPic,lbImg,lbClose,lbPrev,lbNext,lbCounter,lbCap`) are used identically across Tasks 7, 8, 10. `data-lb-album` (cards, Task 6) matches the keys consumed in Task 7. Gallery `data-album`/`data-index` match `albums` keys and array order (Task 10). i18n keys added in a task match the `data-i18n`/`data-i18n-aria` attributes introduced in the same task. ✅

**Known risk flagged in-plan:** font masters + fonttools unavailable (Task 13 stops-and-reports rather than shipping broken glyphs); `room-triple` is a base-name prefix of the new `room-triple-1..6` (Task 14 Step 2 handles the glob collision explicitly).
