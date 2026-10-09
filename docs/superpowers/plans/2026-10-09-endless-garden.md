# Endless Garden Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship Endless Garden as one `index.html`: 45-gene plant generation, a reveal screen, a persistent gallery, and hybridization with lineage.

**Architecture:** Vanilla JS in one file, in labelled sections (Genetics, Naming, Layout, Renderer, Store, UI, Loop). Genetics and layout are pure functions with no DOM, so browser checks can call them through the `EG` namespace. The renderer draws with Canvas 2D from pre-rendered leaf-cluster sprites plus per-frame transforms.

**Tech Stack:** HTML5, CSS3, vanilla JavaScript, Canvas 2D, localStorage, Web Share API (optional). Verification uses Playwright with the preinstalled Chromium, from the session scratchpad. The harness is not committed.

**Spec:** `docs/superpowers/specs/2026-10-09-endless-garden-design.md` (gene tables in Appendix A)

**Execution:** Native (inline) in this session. The session is non-interactive, so the plan-review round trip is skipped and this method was chosen by the author.

## Global Constraints

- One deliverable file: `index.html`, with inline CSS and JS. No network requests, no libraries, no build step.
- 45 genes in groups Trunk 9, Foliage 9, Flowers 7, Fruit and seed 5, Form 4, Special 6, Unique 5. Values come from spec Appendix A.
- Score = 100 × (share of the 10,000-genome calibration set with bits ≤ the plant's bits). Tiers by score: Common < 60; Uncommon 60–85; Rare 85–95; Epic 95–99; Legendary 99–99.7; Mythic ≥ 99.7.
- Notable trait: allele probability `p < 0.12`. The reveal shows at most 8.
- Breeding: per-gene pA uniform in [0.40, 0.60]. Mutation chance mu uniform in [0.05, 0.12] per offspring. A mutation touches 1 gene (80%) or 2 genes (20%).
- Naming: pure plants get an epithet with probability 0.45. Legendary and Mythic always get a legend epithet. Hybrid name is "A × B" with probability 0.5, otherwise the blended genus. Names are trimmed, 1–40 characters, and rendered as text.
- Storage: key `endless-garden:v1`, shape `{version: 1, plants: [], settings: {haptics, particles}}`.
- Layout: node budget 240; 1–3 leaf clusters per tip.
- Timing: reveal growth 1.8 s; detail-open growth 1.2 s; generate lab 0.9 s; hybridize lab 1.4 s; a tap skips a lab after 0.3 s.
- Limits: particle pool 60; thumbnail LRU 150; runtime cache 8 plants × 3 resolutions; gallery page 48.
- Share poster: 1080 × 1350 PNG.
- Every interactive control is at least 44 px tall. Use `100dvh` with a `100vh` fallback. Landscape layout applies at `(orientation: landscape) and (max-height: 620px)`.
- Test namespace: `window.EndlessGarden` (alias `EG`). Each task that produces a function listed in its Interfaces also assigns it to `EG`. The namespace is read-only and must not change state.
- Commit messages end with the trailer `Claude-Session: https://claude.ai/code/session_01NHFKAZCPzB2NkVYYCDfffV`. Never write a model name in commits, PR text, or code.

## Review Focus

1. Corrupt or hand-edited saved data (invalid JSON, 44 genes, allele index out of range). The game still loads; bad records are dropped and good ones are kept. Tested in Task 9.
2. Storage unavailable or full. Play continues in memory with a toast, and no uncaught error is thrown. Tested in Task 9.
3. Rename input: empty, whitespace-only, 200 characters, `<img src=x onerror=…>`. The name is trimmed, capped at 40, and shown as text. Tested in Task 11.
4. Double activation (rapid double taps on Generate, Save, Hybridize, Delete). Exactly one plant is created, saved, bred, or removed. Tested in Tasks 10, 12, and 13.
5. Orientation change and extreme viewports (320×568, 1024×1366) while a reveal or detail is open. The canvas resizes, no uncaught error occurs, and the page does not scroll horizontally. Tested in Task 15.

## Tasks

### Task 1: Shell, tokens, and harness

**Files:**
- Create: `index.html`
- Commit: `docs/superpowers/specs/2026-10-09-endless-garden-design.md`, `docs/superpowers/plans/2026-10-09-endless-garden.md`
- Harness (not committed): `$SCRATCH/harness/check.mjs` and `$SCRATCH/harness/checks/*.mjs`, where `$SCRATCH` is the session scratchpad directory

**Interfaces:**
- Produces: DOM ids `screen-home`, `screen-gallery`, `screen-detail`, `overlay-lab`, `overlay-reveal`, `sheet`, `dialog`, `settings`, `toast`. CSS tokens on `:root`: `--bg0`, `--ink`, `--gold`, `--tier-common`, `--tier-uncommon`, `--tier-rare`, `--tier-epic`, `--tier-legendary`, `--tier-mythic`. The `window.EndlessGarden` object (alias `EG`), filled by later tasks.
- Harness: `node $SCRATCH/harness/check.mjs <name>` loads `file:///home/user/Endless-Garden/index.html` in headless Chromium, runs `checks/<name>.mjs` (default export `async (page, EG) => {...}`, which throws on failure), and fails on any console error or page error. It prints `PASS <name>` or `FAIL <name>: <reason>`. Launch with the default Chromium; if that fails, use `executablePath: '/opt/pw-browsers/chromium'`. Import Playwright from `/opt/node22/lib/node_modules/playwright`.

- [ ] **Step 1: Write the failing check.** `checks/shell.mjs` asserts that every id above exists, that the viewport meta contains `viewport-fit=cover`, and that `getComputedStyle(document.body).backgroundColor` is not transparent.
- [ ] **Step 2: Run it and expect FAIL.** `node $SCRATCH/harness/check.mjs shell` prints `FAIL shell` because the file does not exist yet.
- [ ] **Step 3: Implement the shell.** Doctype, viewport meta, theme-color, and title "Endless Garden". Dark botanical tokens on `:root` (deep greens, gold accents, deep shadows). The containers from Interfaces. One `<script>` with section banners (`Genetics`, `Naming`, `Layout`, `Renderer`, `Store`, `UI`, `Loop`) and `window.EndlessGarden = {}`.
- [ ] **Step 4: Run it and expect PASS.** `node $SCRATCH/harness/check.mjs shell` prints `PASS shell`.
- [ ] **Step 5: Commit.** Stage `index.html` and the two docs, then commit with the message `feat: add game shell and design tokens` and the trailer.

### Task 2: Gene catalogue

**Files:**
- Modify: `index.html` (section Genetics)
- Test: `checks/genes.mjs`

**Interfaces:**
- Produces: `GENES`, an array of 45 `{i, key, label, group, alleles}`. Each allele is `{i, name, w, p, bits}`, where `w` is the weight from Appendix A, `p = w / sum(w)` within its gene, and `bits = -log2(p)`. `GENE_GROUPS`, an ordered list of `{id, label, count}`. `GENE_COUNT = 45`. Visual values are not stored here; the renderer looks them up by gene key and allele name (Task 7).

- [ ] **Step 1: Write the failing check.** `EG.GENES.length === 45`. Group counts are `{trunk:9, foliage:9, flower:7, fruit:5, form:4, special:6, unique:5}`. Every gene has 2–8 alleles and `|sum(p) - 1| < 1e-9`. The allele `Massive Trunk` has `w === 3`, `Golden Leaves` has `w === 1.8`, and `Bioluminescent` has `w === 0.7`.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement the catalogue.** Transcribe Appendix A into compact data of the form `[key, label, group, [[name, w], ...]]`. Derive `i`, `p`, and `bits` at load. Assign `EG.GENES`, `EG.GENE_GROUPS`, and `EG.GENE_COUNT`.
- [ ] **Step 4: Run it and expect PASS.**
- [ ] **Step 5: Commit** with the message `feat: add 45-gene catalogue` and the trailer.

### Task 3: Sampling, rarity, calibration, and tiers

**Files:**
- Modify: `index.html` (section Genetics)
- Test: `checks/rarity.mjs`

**Interfaces:**
- Consumes: `GENES`.
- Produces: `rngFrom(seed) -> () => number in [0, 1)` (mulberry32). `fnv1a(str) -> uint32`. `sampleGenome(rng) -> number[45]`, one allele index per gene, drawn by weight. `analyze(genes) -> {alleles, bits, score, tier, seed, notable}`, where `alleles` is the 45 allele objects, `tier` is `{id, label}`, and `notable` is `[{gene, allele}]` sorted by ascending `w`. `TIERS`, an array of `{id, label, min}` with mins 0, 60, 85, 95, 99, 99.7. `CAL`, a sorted Float64Array of 10,000 bits values drawn from `rngFrom(0x5EED)` and built once at load.
- Score: `100 * (number of CAL values ≤ bits) / CAL.length`, found by binary search. `seed = fnv1a(genes.join(','))`.

- [ ] **Step 1: Write the failing check.**
  ```js
  const rng = EG.rngFrom(12345), N = 20000, counts = {};
  for (let k = 0; k < N; k++) {
    const id = EG.analyze(EG.sampleGenome(rng)).tier.id;
    counts[id] = (counts[id] || 0) + 1;
  }
  const pct = (id) => (100 * (counts[id] || 0)) / N;
  // expect |pct(common) - 60| <= 2, uncommon 25 ±2, rare 10 ±1, epic 4 ±0.6,
  //        legendary 0.7 ±0.2, mythic 0.3 ±0.15
  // expect analyze(g) deep-equals analyze(g) on a second call
  // expect notable.every(n => n.allele.p < 0.12) and notable sorted by ascending w
  ```
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the functions above. Sampling is a linear scan over cumulative weights. Tier is the highest `min ≤ score`. Build `CAL` once at load.
- [ ] **Step 4: Run it and expect PASS.**
- [ ] **Step 5: Commit** with the message `feat: add rarity scoring and tier calibration` and the trailer.

### Task 4: Procedural names

**Files:**
- Modify: `index.html` (section Naming)
- Test: `checks/naming.mjs`

**Interfaces:**
- Consumes: `rngFrom`, `fnv1a`, `analyze` output (`seed`, `tier`).
- Produces: `PREFIXES`, `SUFFIXES`, `EPITHETS`, `LEGEND_EPITHETS` (fixed lists, at least 20 entries each). `pureName(analysis) -> {name, genus: {pre, suf}}`. `hybridName(recordA, recordB, analysis, rng) -> {name, genus: {pre, suf}}`, where each record has `name` and `genus`.
- Rules: the name RNG is `rngFrom(fnv1a('name:' + seed))`. An epithet is chosen with probability 0.45 for Common through Epic, and always for Legendary and Mythic (from `LEGEND_EPITHETS`). Hybrid genus is always `pre(A) + suf(B)`. Name style is `A.genus + ' × ' + B.genus` with probability 0.5, otherwise the blended genus, which takes an epithet by the same rules.

- [ ] **Step 1: Write the failing check.** The same genome gives the same name on two calls. Every name is 1–40 characters. Across 5,000 pure names for a Common analysis, the epithet share is in [0.41, 0.49]. For 500 names with `tier.id = 'legendary'`, every name contains a `LEGEND_EPITHETS` entry. `hybridName` with A's prefix "Lumi" and B's suffix "veil" yields genus "Lumiveil".
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the lists and functions above.
- [ ] **Step 4: Run it and expect PASS.**
- [ ] **Step 5: Commit** with the message `feat: add procedural plant and hybrid names` and the trailer.

### Task 5: Breeding, mutation, and forecast

**Files:**
- Modify: `index.html` (section Genetics)
- Test: `checks/breeding.mjs`

**Interfaces:**
- Consumes: `GENES`, `rngFrom`, `analyze`, `hybridName` (Task 4).
- Produces: `breed(genesA, genesB, rng) -> {genes, origin, mutations, mutationChance}`. `origin[i]` is 0 (from A), 1 (from B), 2 (shared), or 3 (mutated). `mutations` is `[{gene, allele}]`, possibly empty. `forecast(genesA, genesB) -> [{gene, allele, chance, src}]`, where `chance` is 1 for shared alleles and 0.5 otherwise, and `src` is `'shared'`, `'A'`, or `'B'`. Only alleles with `p < 0.12` are listed, sorted rarest first. `makeHybrid(recordA, recordB, rng) -> record`, with `gen`, `parents` (snapshots), `origin`, `mutations`, `mutationChance`, plus the naming and `genes` from `hybridName` and `breed`.
- Algorithm (the signature does not fix it):
  1. For each gene i: if the parents match, take the shared allele (origin 2). Otherwise draw `pA = 0.4 + 0.2 * rng()`. If `rng() < pA`, take A's allele (origin 0); otherwise take B's (origin 1).
  2. `mu = 0.05 + 0.07 * rng()`. If `rng() < mu`, then `hits = rng() < 0.2 ? 2 : 1`. For each hit, pick `g = floor(rng() * 45)`. Candidates are the alleles of gene g not equal to A's or B's allele. If there are none, use every allele other than the current one. Pick a candidate with probability proportional to `1 / sqrt(w)`. Set origin 3 and record `{gene: g, allele}`.
  3. Set `mutationChance = mu` on every result.
  4. `makeHybrid` sets `gen = max(A.gen, B.gen) + 1` and stores parent snapshots `{id, name, genus, gen, genes}`.

- [ ] **Step 1: Write the failing check.** Over 5,000 breedings of random pairs, the fraction of origin-0 genes among genes where the parents differ is in [0.49, 0.51]. Over 20,000 breedings, the fraction with at least one mutation is 0.085 ± 0.01. No mutation allele equals A's or B's allele for a gene with at least 3 alleles. `forecast` on identical parents returns only entries with `chance === 1`.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** `breed`, `forecast`, and `makeHybrid` using the algorithm above.
- [ ] **Step 4: Run it and expect PASS.**
- [ ] **Step 5: Commit** with the message `feat: add breeding with inheritance, mutation, and forecast` and the trailer.

### Task 6: Layout generator

**Files:**
- Modify: `index.html` (section Layout)
- Test: `checks/layout.mjs`

**Interfaces:**
- Consumes: `GENES`, `analyze(genes).alleles`, `rngFrom`.
- Produces: `buildLayout(genes, seed) -> Layout`, where `Layout = {nodes, clusters, trunkChain, marks, aerial, buttress, mushrooms, vine, bounds}`.
  - `nodes`: `[{parent, depth, x, y, angle, length, w0, w1, phase, sway, birth, color}]`. `parent` is -1 for trunk roots. A child's index is always greater than its parent's.
  - `clusters`: `[{node, along, offX, offY, angle, radius, leaves, flowers, fruit, seed, birth, crystal}]`.
  - `trunkChain`: node indices of the trunk segments, bottom to top.
  - `bounds`: `{minX, maxX, top}` at rest. The base is at y = 0, and `top` is the most negative y.
- Algorithm (the signature does not fix it):
  - The trunk is a chain of 4 segments with a small twist from `trunkTwist`. `zigzag` lean alternates sign. `trunkCount` gives 1, 2, 3, or 5 trunks, spaced along x.
  - Whorls at chain nodes 1 to 3 follow `canopyTiers`. Each whorl is a short lateral branch.
  - Recursion: a branch spawns `k` children, with k drawn from the `branching` gene's range. The spread is the `branching` spread times the silhouette spread. Silhouette `up` lifts children and `dr` pulls them down (add `dr * 0.9` to the vertical component, then normalize). Child length is parent length × the `branching` length factor × (0.85–1.15). Stop at the depth from `branching` or when the 240-node budget runs out.
  - Tips get 1–3 clusters, scaled by `foliageDensity`. Cluster radius is `leafLength * (1.5 + 0.35 * sqrt(leaves))`. Leaves are 4–12 per cluster, scaled by density.
  - Flowers and fruit counts follow `flowerPresence` and `fruitCount`. Use the probabilities you choose and keep them fixed.
  - Bark marks, aerial roots, buttress roots, mushrooms, vines, and thorns come from their genes (Appendix A) and are placed on chain or branch nodes.
  - Keep the RNG call order fixed so output is deterministic.

- [ ] **Step 1: Write the failing check.** For 300 random genomes: `nodes.length <= 240`; every node's `parent < index`; `bounds.top < 0`; `buildLayout(g, s)` deep-equals a second call. For a worst-case genome (maximum density, `branching` = Candelabra, silhouette = Cloud-Dome, canopy tiers = Floating Tiers), `nodes.length <= 240`.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** `buildLayout` using the algorithm above. Assign `EG.buildLayout`.
- [ ] **Step 4: Run it and expect PASS.** Also time 300 builds. The average must be under 5 ms in headless Chromium, and the check prints it.
- [ ] **Step 5: Commit** with the message `feat: add deterministic plant layout generator` and the trailer.

### Task 7: Sprites, renderer core, and loop

**Files:**
- Modify: `index.html` (sections Renderer and Loop)
- Test: `checks/renderer.mjs`

**Interfaces:**
- Consumes: `buildLayout`, `GENES`, `analyze`.
- Produces: `VISUALS`, a table keyed by gene key and then allele name, giving colors, shape ids, counts, and tags. `class PlantView` with `constructor(canvas, opts)`, `setPlant(record, opts)`, `resize()`, `start()`, `stop()`, `drawFrame(t, g)`, and `burst(color)`. `record` is a plant record (spec section 4). `opts` is `{detail, thumb, hybridColors}`. `Loop` with `add(viewer)`, `remove(viewer)`, which runs one `requestAnimationFrame` loop only while viewers are active and pauses on `visibilitychange`.
- Sprites: `buildSprites(layout, res)` returns one offscreen canvas per cluster. `res` is device pixels per world unit, and sprites are cached per plant and resolution bucket (rounded to 16 px). The runtime cache holds at most 8 plants and 3 resolutions each, with LRU eviction.
- Fit: `U = min(W * 0.46 / halfWidth, H * 0.82 / height)` in CSS px, with the base at `y = H * 0.92`.
- Frame order: background glow → branches (tapered quads with highlight strokes) → bark marks → cluster sprites (one `drawImage` each, transform = rotation, growth scale, and rustle) → overlays.

- [ ] **Step 1: Write the failing check.** Create a 390×844 canvas and a `PlantView`. For 50 random records: `setPlant` and `drawFrame(1.0, 1)` both complete without exception, and `ctx.getTransform()` is the identity afterward. Averaged over 120 frames of a detail-size canvas, `drawFrame` takes under 6 ms (headless proxy; the check prints the number).
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** `VISUALS` (one visual entry per allele, using the appendix names), leaf outlines computed once per shape and edge type, per-leaf gradients, flowers and fruit composed into each sprite, `PlantView`, `Loop`, and the runtime cache. Assign `EG.PlantView`.
- [ ] **Step 4: Run it and expect PASS.**
- [ ] **Step 5: Commit** with the message `feat: add plant renderer with cached cluster sprites` and the trailer.

### Task 8: Effects, reduced motion, and thumbnails

**Files:**
- Modify: `index.html` (section Renderer)
- Test: `checks/effects.mjs`

**Interfaces:**
- Consumes: `PlantView` (Task 7).
- Produces: `PlantView.motion`, which is 1 normally and 0.4 under `prefers-reduced-motion`. `PlantView.particles`, a pool of at most 60. `burst(color)` spawns 12 star motes. `thumbCache`, an LRU Map of data URLs with at most 150 entries. `renderThumb(record) -> dataURL`, which draws at 220 px with no sway, no particles, and half the clusters. Assign `EG.renderThumb`.
- Effects, each behind its gene's non-default allele: glow aura from `glowAura`; hybrid aura (two radial gradients from `hybridColors`, plus orbiting motes) when `opts.hybridColors` is set; particles from `particleEffect` (pollen, petals, spores, fireflies, starlight, light rays); ethereal alpha from `ethereal`; hover offset and ground shadow from `floating`; crystal sparkles from `crystalline`; vines, mushrooms, thorns, and moss from their genes.

- [ ] **Step 1: Write the failing checks.** The particle pool stays at or below 60 after 600 frames. `motion === 0.4` under emulated reduced motion. After 200 `renderThumb` calls, `thumbCache.size <= 150`. A hybrid record with `hybridColors` draws without error.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the effects as listed. Each effect is off for the default allele.
- [ ] **Step 4: Run it and expect PASS.** Then screenshot six seeded plants with different alleles into `$SCRATCH/shots/` and view them. Fix any broken shapes before committing.
- [ ] **Step 5: Commit** with the message `feat: add plant effects, reduced motion, and thumbnails` and the trailer.

### Task 9: Store, validation, and fallback

**Files:**
- Modify: `index.html` (section Store)
- Test: `checks/store.mjs`

**Interfaces:**
- Consumes: the plant record shape (spec section 4), `GENES` for allele ranges.
- Produces: `Store` with `all() -> records` (newest first), `add(record) -> record`, `update(id, patch) -> record | null`, `remove(id) -> bool`, `get(id) -> record | null`, `load()`, `save()`, and `memoryOnly` (boolean). `showToast(message)` uses the `#toast` element from Task 1. Assign `EG.Store` and `EG.showToast`.
- Load: parse inside try/catch. Keep a record only if `id` is a string, `name` is a string, and `genes` is an array of 45 integers, each within its gene's allele range. Drop the rest silently, and show one toast if any were dropped. Save runs after every change. On a save failure, set `memoryOnly = true` and show the toast "Saving is unavailable. Your garden lasts for this session."

- [ ] **Step 1: Write the failing checks.** Using `addInitScript`, seed `endless-garden:v1` with `'{not json'`. Load the page. `Store.all().length === 0`, there are no page errors, and the toast is visible. Seed one valid record, one with 44 genes, and one with gene index 99. Exactly one record is kept. Stub `Storage.prototype.setItem` to throw, then call `Store.add(record)`. `memoryOnly === true`, `all()` includes the record, and the toast is visible.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** `Store`, `showToast`, and the validation rules above.
- [ ] **Step 4: Run it and expect PASS.**
- [ ] **Step 5: Commit** with the message `feat: add persistent store with validation and fallback` and the trailer.

### Task 10: Home, generate flow, and reveal

**Files:**
- Modify: `index.html` (UI: home, lab, reveal)
- Test: `checks/generate.mjs`

**Interfaces:**
- Consumes: `sampleGenome`, `analyze`, `pureName`, `PlantView`, `Store`, `showToast`.
- Produces: `newPlant(seed?) -> record`, a record not yet stored, with `id`, `name`, `genus`, `genes`, `createdAt`, `fav: false`, and `gen: 0`. `openLab(kind, opts) -> Promise` for `kind` `'generate'` or `'hybridize'`; it resolves after the duration in Global Constraints, or on a tap after 0.3 s. `openReveal(record, opts)` takes `opts.hybrid` (boolean). `closeReveal()`. `buzz(pattern)` calls `navigator.vibrate` only when `settings.haptics` is on.
- Behavior: Generate sets a busy guard, calls `newPlant()`, runs `openLab('generate')`, then `openReveal(record)`. The reveal shows the growing `PlantView` (1.8 s), the name, tier and score, a rarity meter animated to the score, and up to 8 notable traits. Save to Gallery calls `Store.add`, shows "Saved", and disables itself (idempotent). Generate Another discards an unsaved specimen and starts a new generate. The × button discards an unsaved specimen. The home screen shows the latest specimen (tap opens its detail), counts, and shortcuts to Gallery and Hybridize (Hybridize opens the gallery in breed mode, built in Task 13).

- [ ] **Step 1: Write the failing check.** Click Generate. The reveal's name is visible within 1.5 s. Click Save twice quickly: `Store.all().length === 1`. Click Generate twice quickly: exactly one reveal opens (Review Focus 4). After Save and a reload, the plant is still listed.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the home screen, the lab overlay, the reveal overlay, and the busy and saved guards.
- [ ] **Step 4: Run it and expect PASS.** Screenshot the reveal at 390×844 and inspect it.
- [ ] **Step 5: Commit** with the message `feat: add home screen, generate flow, and reveal` and the trailer.

### Task 11: Gallery, search, filters, sort, card actions, and rename

**Files:**
- Modify: `index.html` (UI: gallery, sheet, dialog)
- Test: `checks/gallery.mjs`

**Interfaces:**
- Consumes: `Store`, `renderThumb`, `analyze`.
- Produces: `openGallery()`. `galleryState = {query, filter, sort, pageSize: 48, shown}`. `visibleList() -> records`. `openSheet(record, actions)`. `promptText({title, value, max}) -> Promise<string | null>`, which is the rename dialog. `confirmDialog({title, message, confirmLabel}) -> Promise<boolean>`.
- Filters: `all`, `fav`, `hybrid`, `pure`, and each tier id. Sort: `newest`, `oldest`, `rarest` (score descending), `common` (score ascending), `name` (A–Z), `generation` (descending). Search matches the name or any notable trait name, case-insensitively. Pages of 48 load through an IntersectionObserver sentinel. Cards show a lazy thumbnail (queued, two per frame), a tier chip, a hybrid badge, the generation, a favorite star, a "more" button, and a 500 ms long-press. Both open the sheet with View, Rename, Favorite, and Delete.
- Rename: `promptText` trims the input, keeps the old name when the result is empty, caps at 40 characters, and writes the name with `textContent`.

- [ ] **Step 1: Write the failing check.** Seed 120 plants with `Store.add`. The gallery shows 48 cards; after scrolling to the bottom, 96. Search "zzzqqq" gives 0 cards; All gives 120. Rename (Review Focus 3): input `"   "` leaves the name unchanged; 200 characters store a 40-character name; `<img src=x onerror=…>` creates no `img` element and appears as text. Double-tap Delete in the confirm dialog removes exactly one record (Review Focus 4).
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the gallery, the sheet, the rename dialog, and the confirm dialog.
- [ ] **Step 4: Run it and expect PASS.** Screenshot the gallery in both orientations with 30 seeded plants.
- [ ] **Step 5: Commit** with the message `feat: add gallery with search, filters, sorting, and rename` and the trailer.

### Task 12: Detail view and routing

**Files:**
- Modify: `index.html` (UI: detail, routing)
- Test: `checks/detail.mjs`

**Interfaces:**
- Consumes: `PlantView`, `Store`, `openSheet`, `promptText`, `confirmDialog`, `analyze`.
- Produces: `openDetail(id)`. Route `#/plant/<id>` is handled on `hashchange` and `popstate`. An unknown id shows the gallery and the toast "That specimen is gone."
- Content: a hero `PlantView` that grows over 1.2 s on open. The name with rename. Badges for tier, hybrid, and generation. A rarity card with the score, the tier, and "Rarer than X% of plants". Notable traits with a "Show all" toggle. The full 45-gene list grouped by the seven groups, with allele name and percentage. Lineage: parent names and thumbnails from the snapshots, with origin tags. Actions: Favorite (toggles and saves), Share (wired in Task 14), Use as parent (sets the breed selection and opens the gallery in breed mode), and Delete (confirm, then back to the gallery).

- [ ] **Step 1: Write the failing check.** Open a seeded plant: the hero canvas paints. Toggle Favorite, reload, and the favorite persists. `#/plant/nope` shows the gallery and the toast. Double-tap Delete in the confirm dialog removes exactly one record (Review Focus 4).
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the detail view and routing.
- [ ] **Step 4: Run it and expect PASS.** Screenshot the detail view in portrait and landscape.
- [ ] **Step 5: Commit** with the message `feat: add animated detail view with favorites and delete` and the trailer.

### Task 13: Breeding mode and hybrid reveal

**Files:**
- Modify: `index.html` (UI: gallery breed mode, lab, reveal)
- Test: `checks/hybrid.mjs`

**Interfaces:**
- Consumes: `forecast`, `makeHybrid`, `openLab`, `openReveal`, `Store`.
- Produces: `breedState = {parents: []}`, holding at most 2 ids. `toggleParent(id)`. `hybridize() -> Promise`, which requires two parents, runs `openLab('hybridize')`, then `openReveal(record, {hybrid: true})`. Breed Again re-runs breeding on the same parents with a new RNG.
- Selection rules: tapping a selected card deselects it. A third selection replaces the older one. The same plant cannot occupy both slots. The dock shows both parents (thumbnail and name), the top 6 forecast chips, the note "5–12% chance of mutation", and the Hybridize button, which is enabled at two.
- Hybrid reveal: a Hybrid badge; both parent names with thumbnails; an origin tag on each notable trait (from A, from B, shared, or mutated); a mutation callout when there is at least one mutation. Buttons: Save to Gallery and Breed Again. The aura uses the two parents' palettes through `hybridColors`.

- [ ] **Step 1: Write the failing check.** Select two seeded plants: Hybridize is enabled. Click Hybridize twice quickly: exactly one hybrid (Review Focus 4). The reveal shows both parent names. Save: the record has `parents.length === 2`, `origin.length === 45`, and `gen === max(parent gens) + 1`. After deleting parent A, the hybrid's lineage still shows A's name. Breed Again creates a second, different record.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** breed mode, the dock, the hybridize flow, and the hybrid reveal.
- [ ] **Step 4: Run it and expect PASS.** Screenshot the breed dock and the hybrid reveal.
- [ ] **Step 5: Commit** with the message `feat: add hybridization with parent selection and lineage` and the trailer.

### Task 14: Share posters, haptics, and settings

**Files:**
- Modify: `index.html` (UI: share, settings)
- Test: `checks/share.mjs`

**Interfaces:**
- Consumes: `PlantView`, `renderThumb`, `Store`, `confirmDialog`, `buzz`.
- Produces: `sharePoster(record) -> Promise`. The poster is a 1080 × 1350 canvas: gradient background, plant at high resolution, name, tier, score, top three traits, lineage line, and a "Endless Garden" footer. It exports PNG with `toBlob`, then uses `navigator.share({files})` when `navigator.canShare` accepts files, and otherwise downloads `<slug>.png`. `openSettings()` provides a haptics toggle, a particles toggle, and a Reset garden button that confirms, clears the plants, and keeps the settings.
- Haptics: a light buzz on Save, and an error buzz on Delete.

- [ ] **Step 1: Write the failing check.** The poster blob is a PNG of 1080 × 1350. With `navigator.share` deleted, a download event fires. A `navigator.vibrate` spy records a call on Save when haptics are on, and no call when they are off. Reset leaves `Store.all().length === 0` and keeps the settings.
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the poster, the settings sheet, and the haptics wiring.
- [ ] **Step 4: Run it and expect PASS.** Open the exported poster PNG and inspect it.
- [ ] **Step 5: Commit** with the message `feat: add share posters, haptics, and settings` and the trailer.

### Task 15: Responsive layouts, orientation, and motion

**Files:**
- Modify: `index.html` (CSS and resize handling)
- Test: `checks/responsive.mjs`

**Interfaces:**
- Consumes: all screens.
- Produces: CSS for the landscape breakpoint `(orientation: landscape) and (max-height: 620px)`, the `100dvh` fallback, and safe-area padding. A `ResizeObserver` calls `PlantView.resize()`, which rebuilds sprites when the resolution bucket changes.
- Layouts: home has the stage and the actions in two columns in landscape. The reveal has the canvas on the left and a scrolling info column on the right in landscape. The detail hero sticks to the left in landscape. The gallery grid uses `repeat(auto-fill, minmax(140px, 1fr))`. The breed dock is narrower in landscape.

- [ ] **Step 1: Write the failing check.** At 390×844, 844×390, 320×568, and 1024×1366, on home, gallery, detail, and reveal: `document.documentElement.scrollWidth <= window.innerWidth`, and every visible button, chip, and input is at least 44 px tall. Open the reveal at 390×844, then call `setViewportSize(844×390)`. The canvas CSS size changes and there are no page errors (Review Focus 5).
- [ ] **Step 2: Run it and expect FAIL.**
- [ ] **Step 3: Implement** the CSS and the resize handling.
- [ ] **Step 4: Run it and expect PASS.** Screenshot home, reveal, gallery, and detail in both orientations.
- [ ] **Step 5: Commit** with the message `feat: responsive layouts for portrait and landscape` and the trailer.

### Task 16: Verification, review, README, and PR

**Files:**
- Modify: `index.html` if the review finds problems.
- Modify: `README.md` (how to play, how to open the game, what each file is).

- [ ] **Step 1: Run every check.** `for c in shell genes rarity naming breeding layout renderer effects store generate gallery detail hybrid share responsive; do node $SCRATCH/harness/check.mjs $c; done`. Expected: all PASS, with zero console errors.
- [ ] **Step 2: Print genetics stats.** `node $SCRATCH/harness/stats.mjs` prints the tier shares, the inheritance fraction, the mutation rate, and the average generation time. They must match the targets in Tasks 3 and 5.
- [ ] **Step 3: Review the diff.** Check spec sections 1–14 and Appendix A against the code. Look for placeholder text, model names, external URLs, stray `console.log` calls, and unused code. Fix anything found, then rerun Step 1.
- [ ] **Step 4: Update the README and commit** with the message `docs: describe how to play and verify Endless Garden` and the trailer.
- [ ] **Step 5: Push.** `git push -u origin ccr-d12af1dd-mp4csb`. Retry network failures with the session's backoff (2 s, 4 s, 8 s, 16 s).
- [ ] **Step 6: Open the PR** against `main` with the GitHub tool, loaded through ToolSearch. The title is "Add Endless Garden: procedural plant game with hybridization". The body has a summary, a link to the spec, the verification results, and the footer `🤖 Generated with [Claude Code](https://claude.com/claude-code)`. The body must not include the user's email. The repo has no PR template.
- [ ] **Step 7: Report** to the user: what was built, the verification results, the design choices made without review, and the PR link.
