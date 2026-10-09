# Endless Garden: Design

Date: 2026-10-09
Deliverable: `index.html` (one file: HTML, CSS, JavaScript; no network, no libraries)

## 1. Intent and success criteria

A relaxing, collectible botanical game for phones. The player generates unique plants from a 45-gene genome, reveals them, keeps the best in a gallery, and breeds them into hybrids.

Success means:
- Every feature in the request works in Chromium at 390x844 (portrait) and 844x390 (landscape) with zero console errors.
- Generation and breeding feel instant (under about 100 ms of compute; short staged animations cover the rest).
- The gallery survives a reload; hybrids keep their lineage.
- Rarity percentages come from the actual allele weights, not hand-written text.

Assumptions (the request did not state them):
- One dark botanical theme is enough.
- No sound. Haptics are on by default and can be turned off.
- Plant names are procedural and editable.
- Rarity tiers are calibrated against random generation, so tiers mean "rarer than X% of plants".

## 2. Approaches considered

| Approach | Verdict |
|---|---|
| A. Vanilla JS, Canvas 2D, pre-rendered leaf-cluster sprites | Chosen. Fast on phones, no dependencies, meets the one-file rule. |
| B. SVG with CSS animation | Rejected. A plant is hundreds of animated nodes, and a gallery of them would strain mobile. |
| C. WebGL | Rejected. Largest code and most failure modes on old devices, with no gain at this scale. |

## 3. Components

All code lives in `index.html`, in labelled sections. Each unit has one job and a small interface.

- **Genetics.** Gene catalogue, weighted sampling, surprisal (bits), calibration, breeding, forecast, mutation. Interface: `analyze(genes)` returns `{alleles, bits, score, tier, seed, notable}`; `breed(a, b, rng)` returns `{genes, origin, mutations, mutationChance}`.
- **Naming.** Genus and epithet generation. Pure function of the genome seed.
- **Layout.** Deterministic plant structure built from the genome seed. Interface: `buildLayout(alleles, seed)`. No rendering.
- **Renderer.** Sprite building, per-frame drawing, growth, particles, auras. Interface: `PlantView` with `setPlant(record, opts)`, `start()`, `stop()`, `burst()`.
- **Store.** localStorage load, sanitize, save. Interface: `all()`, `add(record)`, `update(id, patch)`, `remove(id)`.
- **UI.** Home, Gallery, Detail screens; Lab, Reveal, sheet, and dialog overlays.
- **Loop.** One `requestAnimationFrame` loop that ticks the registered viewers. It runs only while a viewer is active and pauses when the document is hidden.

## 4. Data model

Persisted under key `endless-garden:v1` as `{version: 1, plants: [...], settings: {haptics, particles}}`.

Plant record:
- `id` (string), `name` (string, max 40 chars), `genus` (`{pre, suf}`), `genes` (45 integers, one allele index per gene)
- `createdAt` (ms), `fav` (bool), `gen` (0 for pure plants, parent max + 1 for hybrids)
- `parents`: null, or two snapshots `{id, name, genus, gen, genes}`. Snapshots keep lineage after a parent is deleted.
- `origin`: null, or 45 integers for hybrids: 0 = from A, 1 = from B, 2 = shared, 3 = mutated
- `mutations`: null, or a list of `{gene, allele}`
- `mutationChance`: the mu drawn for this hybrid, for display

Derived at runtime, never persisted: alleles, bits, score, tier, seed, layout, sprite caches, thumbnail data URLs.

## 5. Genetics

**Catalogue.** 45 genes in 7 groups: Trunk 9, Foliage 9, Flowers 7, Fruit and seed 5, Form 4, Special 6, Unique 5. Each gene has 2 to 12 alleles, as listed in Appendix A. Allele weights are percentages that sum to 100 per gene (normalized at load). The request's examples are included with their stated weights: Massive Trunk 3%, Golden Leaves 1.8%, Bioluminescent 0.7%.

**Sampling.** Each gene is sampled independently by weight.

**Rarity.** For a genome, `bits = sum over genes of -log2(p)`, where p is the allele's probability. Calibration: 10,000 random genomes from a fixed seed, sorted once at load. `score = 100 * (share of calibration genomes with bits <= this plant's bits)`, so a score of 97.4 means rarer than 97.4% of random plants.

**Tiers** (score ranges): Common below 60, Uncommon 60 to 85, Rare 85 to 95, Epic 95 to 99, Legendary 99 to 99.7, Mythic 99.7 and up. Expected shares for random plants: about 60%, 25%, 10%, 4%, 0.7%, 0.3%. A test checks these.

**Notable traits.** Alleles with probability below 12%. Shown with their percentage, rarest first. The reveal shows up to 8. The detail view shows all, plus the full 45-gene list.

## 6. Breeding

For each gene:
- If both parents carry the same allele, the child keeps it (origin 2).
- Otherwise the child takes parent A's allele with probability pA, where pA is drawn uniformly from [0.40, 0.60] for each gene. Otherwise it takes B's (origin 0 or 1).

Mutation: mu is drawn uniformly from [0.05, 0.12] per offspring. With probability mu, one gene mutates (80%) or two genes do (20%). The gene is chosen uniformly from 45. The new allele is chosen from alleles absent from both parents, weighted by 1/sqrt(w), so rare alleles are favored. The gene is marked origin 3.

Lineage: `gen = max(parent gen) + 1`.

Forecast (shown in the breeding dock): for each notable allele in either parent, 100% if both parents share it, otherwise about 50% per parent.

Hybrid naming: 50% "A x B" from the parents' genera, 50% a blended genus (A's prefix + B's suffix), with an optional epithet.

## 7. Naming

Genus = prefix + suffix from fixed syllable lists. Pure plants get an epithet 45% of the time. Legendary and Mythic plants always get a legend epithet. The name is a pure function of the genome seed, so the same genome always gets the same default name. Renaming is free text, trimmed, max 40 characters, and never empty.

## 8. Rendering

**Seeds.** `seed = FNV-1a(genes.join(','))`. Layout RNG = mulberry32(seed). Name RNG = mulberry32(FNV-1a('name:' + seed)).

**Structure.** The trunk is a 4-segment chain. Lateral whorls follow the canopy-tier gene. The crown comes from the branching and silhouette genes. Node budget: 240. Each tip gets 1 to 3 leaf clusters. Clusters follow their node, so they move with it.

**Sprites.** Each cluster (leaves, flowers, fruit, crystal facets) is pre-rendered once per plant and resolution bucket. Each frame draws it with one `drawImage` under a transform that adds rustle and growth scale. Runtime caches hold at most 8 plants and 3 resolutions each.

**Per-frame work.** Node positions are recomputed with sway. Branches are tapered quads with highlight strokes. Bark marks and special features (vines, mushrooms, thorns, aerial and buttress roots, moss) are drawn from node positions.

**Effects.** Glow aura behind the plant. Hybrids get a dual-color aura from both parents' palettes, plus orbiting motes. A particle pool capped at 60 holds pollen, petals, spores, fireflies, starlight, and light rays. Ethereal genes lower alpha. Floating genes raise the plant and draw a ground shadow.

**Growth.** A growth value g runs from 0 to 1 over 1.8 s on reveal and 1.2 s on detail open. Every node and cluster has a birth time, so the plant grows from the base up.

**Level of detail.** Gallery thumbnails skip sway and particles and use half the clusters at low resolution. They are cached as data URLs (LRU, max 150).

**Reduced motion.** When `prefers-reduced-motion` is set, sway is reduced and particle counts are halved.

**Budgets** (measured in headless Chromium as a proxy only, not on a device): generation under 30 ms; sprite build for the detail view under 80 ms; frame script time under 6 ms at detail size.

## 9. Screens and flows

**Home.** A large "Generate Plant" button (primary). A preview of the latest specimen; tapping it opens its detail. Shortcuts to Gallery and Hybridize. A settings button.

**Generate.** Lab overlay (germination animation, about 0.9 s, skippable by tapping after 0.3 s) → Reveal. The Reveal shows the growing plant, the name, the tier and score, notable traits with percentages, and the rarity meter. Buttons: Save to Gallery, Generate Another. Closing an unsaved specimen discards it.

**Gallery.** Count header. Search (name, or a trait name). Filter chips: All, Favorites, Hybrids, Pure, then each tier. Sort: Newest, Oldest, Rarest, Most common, Name, Generation. Grid of lazy-rendered thumbnails, 48 per page, loading more while the user scrolls. Each card has a "more" button (and long-press) that opens a sheet: View, Rename, Favorite, Delete.

**Breed mode.** Toggled from the gallery header. Tapping a card selects it (glow and check; A and B labels). At most two parents; a third selection replaces the older one. The dock shows both parents, the forecast chips, and a Hybridize button that is enabled at two. Hybridize → Lab (DNA ladder animation, about 1.4 s) → Reveal with a Hybrid badge, both parent names, origin tags on each trait, and a mutation callout if one happened. Buttons: Save to Gallery, Breed Again (same parents).

**Detail.** Animated hero (grows on open). Name with rename. Badges for tier, hybrid, and generation. Rarity card. Notable traits with "show all". The full 45-gene list. Lineage with parent thumbnails. Actions: Favorite, Share, Use as parent (opens breed mode with this plant as A), Delete (with confirmation).

**Share.** A 1080x1350 PNG poster: the plant, name, tier, top traits, lineage, footer. Uses the Web Share API with the file when supported, otherwise downloads it.

## 10. Persistence and errors

- Load: parse inside try/catch. Drop records with a bad id or genes (must be 45 integers in range). Keep the rest.
- Save: after every change. Quota or private-mode failures show a toast and keep the garden in memory for the session.
- Deleting a parent keeps the lineage snapshot.
- A detail route for a missing id goes back to the gallery with a toast.
- If `getContext` returns null, show a short "this browser cannot draw plants" message instead of crashing.

## 11. Responsiveness and accessibility

- Portrait is the default layout. Landscape applies at `(orientation: landscape) and (max-height: 620px)`. Gallery columns use auto-fill.
- Uses `100dvh` with a `100vh` fallback and safe-area insets.
- Touch targets are at least 44 px. Icon buttons have labels. Dialogs trap focus and close on Escape.
- `prefers-reduced-motion` is honored (see section 8).

## 12. Testing

- Browser tests (Playwright, Chromium, via `file://`): zero console errors; generate → save → reload → still there; hybridize from two parents; rename; favorite; delete; search and filters; both orientations with screenshots.
- Genetics checks run in the page on 20,000 samples: tier shares within tolerance of the targets; inheritance proportion within [0.40, 0.60]; mutation rate within [0.05, 0.12]; the same genome gives the same name and layout.
- Visual review of the screenshots.

## 13. Out of scope

Sound, accounts, sync, multiplayer, import and export, a codex, a light theme, a build step.

## 14. Decisions made without a review round trip

This session is non-interactive, so these were settled by the author rather than approved by the requester. They are listed so a reviewer can change them:
- Single-file vanilla approach (section 2).
- Percentile-based tiers and the 10,000-sample calibration (section 5).
- Mutation parameters: one or two genes, weighted toward rare alleles (section 6).
- Hybrid naming split of 50/50 (section 6).
- Share poster format (section 9).

## Appendix A. Gene catalogue

One line per gene: `key`: label (group): allele name weight, in order. Weights are relative; the code normalizes them per gene. Visual effects are an implementation choice inside the renderer. They must be deterministic and must never change a weight.

Trunk
- `trunkThickness`: Trunk thickness (trunk): Slender 25, Sturdy 35, Thick 25, Stout 12, Massive Trunk 3
- `trunkHeight`: Trunk height (trunk): Stunted 10, Low 20, Medium 35, Tall 25, Towering 8, Skyward 2
- `barkTexture`: Bark texture (trunk): Smooth 20, Furrowed 30, Scaled 15, Papery 15, Mossy 12, Gnarled 8
- `trunkColor`: Bark color (trunk): Umber 30, Ash Grey 20, Silver 12, Mahogany 18, Ivory 8, Obsidian 7, Moonstone 3, Sun-Gilded 2
- `branching`: Branching (trunk): Single Stem 8, Forked 30, Open Crown 25, Dense Branching 20, Candelabra 10, Spiral Branching 7
- `aerialRoots`: Aerial roots (trunk): None 60, Few 25, Dangling 10, Cascading 5
- `buttressRoots`: Buttress roots (trunk): None 55, Low Buttress 30, Great Buttress 12, Cathedral Roots 3
- `trunkCount`: Trunk count (trunk): Single 70, Twin 20, Triple 8, Clustered 2
- `trunkTwist`: Trunk shape (trunk): Straight 50, Gentle Curve 28, Twisted 15, Corkscrew 5, Ancient Spiral 2

Foliage
- `leafShape`: Leaf shape (foliage): Oval 22, Heart 18, Lanceolate 16, Palmate 12, Round 14, Feathery 10, Star-Lobed 6, Ginkgo Fan 2
- `leafSize`: Leaf size (foliage): Tiny 15, Small 25, Medium 30, Large 20, Enormous 10
- `foliageDensity`: Foliage density (foliage): Sparse 15, Airy 25, Lush 35, Dense 18, Impenetrable 7
- `leafPalette`: Leaf palette (foliage): Verdant 25, Sage 18, Jade 14, Olive 12, Emerald 10, Teal 8, Plum 4, Crimson 3, Amber 3, Silver-Blue 2, Violet 1
- `seasonalShift`: Seasonal shift (foliage): Evergreen 40, Autumn Blush 25, Dawn Gradient 15, Frostbitten 11.5, Blossom Cycle 6.7, Golden Leaves 1.8
- `leafArrangement`: Leaf arrangement (foliage): Alternate 35, Opposite 25, Whorled 15, Rosette 12, Spiraled 9, Clustered 4
- `foliageType`: Foliage type (foliage): Broadleaf 50, Needle 25, Scale 10, Fern Frond 10, Ribbon 4, Glass Needle 1
- `crownPattern`: Crown pattern (foliage): Clumped 25, Even Spread 25, Tiered Leaves 15, Wispy 15, Tufted 10, Veiled 10
- `leafEdge`: Leaf edge (foliage): Smooth 40, Serrated 25, Lobed 15, Frilled 10, Spiked Edge 7, Lace 3

Flowers
- `flowerPresence`: Flowering (flowers): None 30, Sparse 25, Common 25, Profuse 15, Overwhelming 5
- `flowerSize`: Flower size (flowers): Tiny 30, Small 30, Medium 25, Large 12, Giant Blossom 3
- `petalCount`: Petal count (flowers): 3 15, 5 35, 8 20, 12 15, 20 8, 40 5, Fractal 2
- `flowerColor`: Flower color (flowers): White 18, Pink 18, Lilac 12, Yellow 14, Coral 10, Red 8, Blue 8, Indigo 4, Silver 3, Black Velvet 2, Opal 2, Rainbow 1
- `bloomStyle`: Bloom style (flowers): Open Cup 30, Bell 20, Star 20, Tubular 12, Spiral Bloom 9, Cluster Spray 9
- `fragrance`: Fragrance (flowers): Faint 35, Sweet 25, Spiced 15, Heady 12, Resinous 8, Mystic 5
- `bioluminescentFlowers`: Bioluminescent flowers (flowers): None 99.3, Bioluminescent 0.7

Fruit and seed
- `fruitType`: Fruit type (fruit): None 30, Berries 20, Pods 12, Drupes 10, Orbs 10, Nuts 8, Starfruit 6, Lanterns 4
- `fruitColor`: Fruit color (fruit): Scarlet 22, Gold 18, Violet 14, Cream 12, Blue-black 12, Pearl 8, Ruby 6, Ember Glow 4, Jade 4
- `fruitSize`: Fruit size (fruit): Tiny 35, Small 30, Medium 20, Large 12, Huge 3
- `fruitCount`: Fruit count (fruit): Single 25, Handful 30, Bunches 25, Abundant 15, Bountiful 5
- `fruitArrangement`: Fruit arrangement (fruit): Scattered 30, Hanging 25, Crowned 15, Garland 15, Spiral 10, Constellation 5

Form
- `silhouette`: Silhouette (form): Upright 20, Weeping 14, Sprawling 14, Columnar 12, Umbrella 12, Bonsai 10, Cloud-Dome 9, Spiral Spire 5, Hanging Tiers 4
- `lean`: Lean (form): Straight 60, Leaning 25, Heavy Lean 10, Zigzag 5
- `symmetry`: Symmetry (form): Symmetrical 55, Asymmetrical 35, Chaotic 10
- `canopyTiers`: Canopy tiers (form): Single Layer 45, Two Tiers 30, Three Tiers 15, Pagoda 8, Floating Tiers 2

Special
- `glowAura`: Aura (special): None 90, Faint Halo 6, Ember Glow 2.5, Moonlight Aura 1.5
- `crystalline`: Crystalline (special): None 93, Crystal Tips 5, Crystal Veins 1.8, Fully Crystalline 0.2
- `floating`: Floating (special): Grounded 96, Hovering Roots 2.5, Levitating 1.2, Ethereal Float 0.3
- `ethereal`: Ethereal (special): Solid 93, Misty 5, Phantom 1.8, Spectral 0.2
- `ancient`: Age (special): Young 60, Mature 25, Aged 10, Ancient 4, Primordial 1
- `symbiosis`: Symbiosis (special): None 80, Vine-Covered 10, Mushroom Caps 6, Mushroom Symbiont 3, Lichen Crown 1

Unique
- `thorns`: Thorns (unique): None 62, Few Thorns 18, Thorny 12, Spiny 6, Razor Thorns 2
- `particleEffect`: Particles (unique): None 70, Pollen 12, Falling Petals 8, Glowing Spores 5, Light Rays 3, Fireflies 1.5, Starlight Motes 0.5
- `barkPattern`: Bark pattern (unique): Plain 35, Spotted 20, Striped 15, Runic 10, Eye Knots 6, Marbled 9, Veined Glow 5
- `leafGradient`: Leaf gradient (unique): Mono 45, Two-tone 30, Radial 15, Iridescent 8, Aurora 2
- `mossCover`: Moss cover (unique): Bare 45, Patchy Moss 25, Lush Moss 15, Lichen Shelves 8, Velvet Moss 5, Glowing Moss 2
