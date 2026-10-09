# Endless Garden

A collecting game for phones. Generate plants from a 45-gene genome, reveal them, keep the best in your gallery, and breed them into hybrids.

The whole game is one file, `index.html`: markup, styles, and JavaScript, with no network requests and no libraries.

## How to play

1. **Generate.** Tap *Generate Plant*. A short germination animation plays (tap to skip after 0.3 s), then the plant grows.
2. **Read the reveal.** It shows the plant's name, its tier (Common, Uncommon, Rare, Epic, Legendary, or Mythic), its rarity score, and its notable traits. A trait's percentage is how common it is among random plants.
3. **Keep it or let it go.** *Save to Gallery* keeps the plant. Closing an unsaved plant discards it. *Generate Another* starts over.
4. **Browse the gallery.** Search by name or trait, filter by favorites, hybrids, pure plants, or tier, and sort by newest, rarest, name, or generation. Tap a card to open its detail. Long-press a card, or tap its *⋯* button, for View, Rename, Favorite, and Delete.
5. **Breed.** Tap *Breed* in the gallery header, or *Hybridize* on the home screen. Tap two plants to select them: tapping a selected plant clears it, and a third plant replaces the older one. The dock shows the traits the pair may pass on. Each gene comes from one parent or the other, with 40 to 60% from parent A depending on the gene, and 5 to 12% of children carry a mutation. Hybrids keep their lineage and can be parents themselves.
6. **Share.** From a detail view, *Share* makes a 1080 × 1350 poster with the plant, its name, tier, top traits, and lineage. Where the device can share files, the share sheet opens. Otherwise the PNG downloads.
7. **Settings.** The gear on the home screen turns haptics and particles on or off, and resets the garden. Reset keeps your settings.

Rarity is measured against 10,000 random plants, so "rarer than 97% of plants" means what it says.

## How to open

Open `index.html` in a current browser (Chrome, Safari, Firefox, or Edge). No server or build step is needed.

Your garden is saved in the browser's local storage on that device, so it is there when you return. It does not sync between devices. Portrait suits phones held upright, and landscape lays the screens out side by side for phones turned sideways.

## Files

- `index.html`: the whole game, in labelled sections: genetics, naming, layout, renderer, loop, store, and the screens.
- `docs/superpowers/specs/2026-10-09-endless-garden-design.md`: the design, with the full gene catalogue in Appendix A.
- `docs/superpowers/plans/2026-10-09-endless-garden.md`: the implementation plan, one task per feature.
- `README.md`: this file.

## Saved data

Gardens are stored under the key `endless-garden:v1`. On load, a record with a bad id or gene list is dropped, and the rest are kept. If storage is full or unavailable, play continues in memory for the session, and a notice says so.

## Checking it

Open the browser's developer console and play through the steps above at 390 × 844 (portrait) and 844 × 390 (landscape). The console should stay empty. If saving fails, or the browser cannot draw canvas, the game shows a message instead of an error.
