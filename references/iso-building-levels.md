# Isometric building upgrade chains (Lv.1 → Lv.3)

How to turn one existing building sprite into a 3-level upgrade chain without the art falling apart.

Distilled from a settlement asset job: two buildings, 7 levels total, ~16 renders.

Read this before generating any building level.

---

## 0. The one-sentence version

A level chain is **one building growing**, not three buildings. Everything below is about keeping that true while the model tries to redesign it.

---

## 1. The reference for colour is the client's original — never your own edit

If Lv.1 is the client's untouched art, aligning colour to Lv.1 is correct.

If you **regenerated or edited Lv.1 yourself**, it is no longer a reference — it is your own output, and matching to it is self-reference. The palette anchor must be the client's original layer (or the original of an adjacent building from the same set).

Symptom of getting this wrong: every level matches statistically, and the whole set still reads as "a different world from the rest of the art".

---

## 2. Alignment means same furniture scale, not same canvas

Same canvas and same bottom edge is the easy half. The half that breaks:

**Person-scale objects must be identical across levels.** Counters, stoves, tables, stools, pots. The building grows; the furniture does not.

The model will silently shrink the whole scene to fit a new roof into a 1024 canvas. After every generation, check a known object:

```
measure one unchanged fixture (stool width, pot diameter, table length)
→ ratio vs the reference level
→ rescale before placing
```

Do not trust overall content width as a proxy — a bigger building legitimately has a bigger footprint.

In this job: model shrank scenes by ~15–35% across generations, discovered only by comparing stool length.

**Target:** same width for Lv.1/Lv.2, +5% allowed for the top level if it visibly grows. Same bottom Y.

---

## 3. Detail must be tiered, and it is measurable

The most common client note on AI-generated buildings: **"细节处理的太平均"** — detail handled too evenly. Not "too little". Too uniform.

### Why it happens

The model optimises for *plausible everywhere*. It renders every face at similar density. A human illustrator **omits** — puts the pen down on the hero area and leaves the rest alone.

### How to measure it

Split the sprite into a 3×3 grid over its bounding box, compute mean Laplacian per cell, then take **max/min** across cells.

| | mean density | max/min |
|---|---|---|
| Client original | 61.6 | **2.07** |
| Our Lv.3 (even) | 69.9 | 1.64 |
| Our Lv.3 (tiered) | 60.6 | 1.77 |

The original is *less* dense overall but has **twice** the spread. That gap is what "has a focal point" looks like numerically.

**Target: max/min ≥ 2.0, mean density at or below the reference.** Raising density is almost always the wrong move.

### How to specify it in the prompt

Name one hero area and give four explicit tiers. Vague "add less detail" does nothing.

```
Tier 1 · hero (the one thing the player looks at):
  full detail — every object readable, one by one
Tier 2 · same face, rest of it:
  half the detail — flat colour + one shadow, no grain, no bolts, no seams
Tier 3 · back / sides / roof:
  structure only — big flat areas + one structural line. No fittings, no patches, no rust.
Tier 4 · ground:
  one flat colour + one prop for scale. Nothing else.
```

The tier list must be **per surface**, not per feature. "Simplify the roof" gets ignored; "roof: vertical corrugation only, no rust, no patches, no nail heads, two colours only" lands.

---

## 4. Colour unification: three steps, then a fourth that is actually the one that matters

Run on every non-reference level, in this order:

1. **Palette constraint** — k-means (k=24) the reference's colours; snap each target pixel's chroma/hue to the nearest swatch, keep its own L. Preserves shading.
2. **Saturation/lightness convergence** — iterate to the reference's global means.
3. **Luminance histogram match at 65%** — restores the highlight band. **100% produces visible banding; 65% is the practical ceiling.**
4. **Per-material chroma separation** — this is the one that decides whether it reads as the same world.

Step 4 exists because steps 1–3 can match global statistics exactly and the image still looks off. Measured case: target a+11.4 vs reference a+9.4 (target *warmer* on paper) and it still read washed out. The difference was not the mean, it was that the reference's chroma was concentrated in saturated wood, while the target spread it thin over large pale grey counters.

Fix: split by material, push warm chroma up and neutral light greys toward blue-grey. Only then do the two sets sit together.

**Also: do not fix "too orange" by rotating hue.** Rotating yellow-orange toward red-brown made it more vivid, not less. Reduce warm chroma (≈0.62) and b\* (≈0.80); keep hue and L.

---

## 5. Keying: colour distance is the wrong test

Never threshold by distance-to-screen-colour. A greenish-grey wall falls inside the radius and gets punched through.

Correct test:

```
screen = (hue in green band) AND (saturation high)
background = screen components connected to the canvas border
           + enclosed screen pockets → transparent (gaps between thin members)
alpha = NOT background
```

- **Do not fill enclosed screen pockets with a dark colour.** A tripod's gaps become a solid dark triangle at game resolution. Make them transparent.
- **The model will return a two-tone backdrop** (green frame + white interior). Handle both: flood from the border over `green OR near-white`.
- Thin members (antenna, guy wires, tripod legs) collapse at 185 px. If the silhouette depends on them, thicken the member rather than accepting it.

---

## 6. Generation-process failure modes, in order of how often they hit

| Symptom | What the model did | Fix |
|---|---|---|
| Building smaller than the previous level | shrank the scene to fit a new roof | rescale by a known fixture (§2) |
| Front opened up "a bit" instead of fully | treated "open this face" as a local edit | generate the open level **from** a level that is already open |
| Roof that does not close (slats / pergola) | read "add a roof" as "add roof beams" | specify the sheet: corrugation, eaves board, drip edge |
| Everything equally detailed | default behaviour | tier list (§3) |
| Previously specified parts silently dropped | prompt listed them under "keep" but the new instruction dominated | re-state every must-keep item **inside** the change instruction |
| Brick / masonry appearing at too low a level | model escalates material for "upgrade" | state the material ceiling explicitly: "this level has not reached masonry" |

---

## 7. Acceptance checklist

Before delivering a level chain:

- [ ] every level bottom-aligned at the same Y, same width (±5% on the top level only)
- [ ] one unchanged fixture measured across levels, within 5%
- [ ] 3×3 density spread ≥ 2.0, mean ≤ reference
- [ ] palette + histogram matched to the **client original**, not to your own edit
- [ ] warm chroma within ~10% of the reference's material chroma
- [ ] keying: zero screen-coloured pixels inside the silhouette
- [ ] the functional hero (counter, stove, door) is visible at final size, in every level
- [ ] the same building is recognisable across levels: same footprint, same openings, same hero
