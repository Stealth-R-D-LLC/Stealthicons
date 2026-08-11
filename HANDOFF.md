# Stealthicons — handoff

**Deliverable:** `stealthicons.html`, one self-contained file, ~148 KB, zero external requests.
Open it in a browser; there is no build step.

`pseudoword.html` is the prototype it was forked from. It is superseded — keep it only for
reference. Anything you change should go into `stealthicons.html`.

---

## 1. What it is

An address is a wall of characters nobody reads. A stealthicon derives two handles from it:

- a **name** — five letters alternating vowel and consonant, so it can be said aloud
- a **mark** — a symmetric identicon, recognisable at a glance

Both are pure functions of the input text. Nothing is stored, so nothing can drift out of sync.

The security argument is the cost of forging one. Four things compound: **Argon2d** makes each
guess expensive in time *and memory*; the **name** narrows the target; the **mark** narrows it
much further; the **address** still has to be checked. Verifying is one hash. Grinding a
convincing lookalike is not.

The HTML file is a *workbench* — every parameter is exposed so a user can settle on a look.
**The defaults it opens with are the official parameters.** Leave them and the output matches
every other conforming implementation.

---

## 2. Pipeline

```
text ──▶ digest ──┬─▶ bits 0..18   ──▶ name (5 letters)
                  ├─▶ bits 64..191 ──▶ xorshift128 seed ──▶ mark + palette
                  └─▶ bits 19..63, 192..255  unread
```

**Digest.** Argon2d by default: 16 MiB, 8 passes, parallelism 1, 32-byte output, fixed salt
`stealthicons-v1`. Bundled from hash-wasm (MIT), wasm inlined as base64 — no CDN. SHA-1/256/384/512
also available via Web Crypto. Verified against the Argon2 reference vector
(`password`/`somesalt`, t=2, m=64 MiB, p=1). The salt is fixed deliberately: a stealthicon must be
a pure function of its address, so there is nothing per-user to vary and the cost parameters carry
the defence.

**Name.** Repeated `divmod` off the low end of the first 64-bit window: one bit picks vowel-first
or consonant-first, then five indices into a consonant set and a vowel set. Two alphabets —
`b58` (`bcdfghjkmnpqrstvwxz` / `aeiouy`, no `l`) and `full` (adds `l`), default `full`. Costs
~19.1 bits.

**Mark.** `makeRng(hex, salt)` builds four 32-bit xorshift words from digest **bytes 8–23**. Every
grid size and the palette draw from that same 128-bit slice with different salt values.

---

## 3. The two lattices

### Hexagonal (default)

Triangular lattice addressed in HECS. Long rows of *n* alternate with short rows of *n−1* offset
half a cell, so **all six neighbours are exactly one unit away** — the reason every waist is
identical, and the reason the whole thing was moved off the square grid.

- Odd grids run short-long-short; even grids drop the first and last dot of their top row. Both
  keep the corners clear of the crop, and make even grids pleasantly bottom-heavy.
- Chains root on the midline, sweeping `q` outward, rows centre-out with the lower row of each
  pair first. Extension probability `D / (1 + steps/W)`. Each dot may bifurcate once; the re-walk
  runs to a fixed point.
- Adjacent same-coloured **leaves** close with probability *Closure*, single pass.
- A dot adjacent to its own mirror mate is always joined — this stitches the midline.
- If every non-eye chain came out one colour, one is recoloured (from the foreground/spot pair,
  never the accent).

**Face-like.** One dot at cell coordinates `(2n/3, n/3)`, mirrored into a pair of eyes. They read
as eyes because they carry **no connections** — that falls out for free, since chains only link
cells they place themselves and eyes are excluded from the mate-join and closure passes.
Neighbouring cells fill normally. The accent colour is reserved for eyes, so body chains use only
the other two, and any *other* unconnected dot is joined to a neighbour (taking its colour) or
given a neighbour of its own. A dot exactly on the midline is exempt — no symmetry mate.

### Rectilinear

Each colour grows outward from a single cell on the mirror line, so no colour is ever split into
islands. *Density* sets how much of the half-grid to fill; the growth rule loosens as density
climbs or the dial saturates near half fill. The third shape is seeded on the mirror line too —
the only place it can sit without its reflection becoming a second island — so it appears whenever
it can be had without floaters. Verified over 9,600 grids: zero islands, zero missing thirds.

---

## 4. Geometry

Dots are circles of radius `r` or hexagons of apothem `r` (hexagons only on the hex lattice).

**Circle waists.** Two circular fillets tangent to both dots. Given a neck half-width `w` and a
join length `s`, the fillet radius is solved so the arc leaves each circumference tangentially:

```
ρ = (r² − s²/4 − w²) / (2(w − r))
```

The *Waist* dial sets `w` as a fraction of the dot **diameter**. At 1 the pinch vanishes and the
connector short-circuits to a straight bar (the formula divides by zero exactly there).

**Ideal.** Forces `ρ = 1 − r`, putting the fillet centre on a vacant lattice site. Six dots
ringing an empty cell then leave a hole that is a **true circle**. Solved per join length, so it
is correct on both lattices. Needs `r > 1 − √3/2` (diameter > 0.268).

**Hexagon dots.** Flat side facing each of the six neighbours; the connector is a bar as wide as
the hexagon is **tall**, vertex to vertex (`2r·2/√3`), so its edges run through the two vertices
flanking the join.

**`DIAG_RATIO ≈ 0.4971`** scales diagonal necks on the *rectilinear* lattice, so at Waist 1 the
orthogonal joins are full width but the diagonals are about half. This is deliberate — the
directional inconsistency was judged charming. It is not scale-invariant (the true ideal ratio
runs −8.9 → 0.57 across dot diameters and is only exact at 0.9), so treat it as an aesthetic
constant, not a derivation. The hex lattice has one join length and so needs no such thing.

---

## 5. Palette

725 combinations: 337 from Nanoidenticons (keerifox, WTFPL) plus 388 generated for this project
across three families — dark grounds with bright dots, light grounds with deep dots, mid grounds
with dots at both extremes — over 24 base hues and 12 hue schemes.

Nanoidenticons' own combinations were tuned for its renderer and many wash out here. Each is
repaired on first use, **moving lightness only** so hues and saturations survive:

| constant | value | meaning |
|---|---|---|
| `BG_AIM` | 5.5 | contrast ratio aimed for |
| `BG_MIN` | 3.5 | hard floor, never crossed |
| `SEP_MIN` | 12 | ΔE between dots — only enough to stop two being identical |
| `SPOT_CAP` | 20 | how far a spot may be dragged to rescue a stuck pairing |

Aiming at the floor rather than past it was the original bug: it parked 201 of 337 combinations at
exactly 3.5, which is what read as washed out. Repair is lazy and memoised, ~3 ms first use.

A **third colour** is derived rather than stored: the hue furthest from both dot hues, with the
stronger saturation and whichever lightness stands further from the ground.

**Transparent** mode drops the background rect; the host named by *On dark* becomes the ground and
combinations whose dots wouldn't clear it are **filtered out**, not altered. Pools: 362 dark-safe,
199 light-safe.

---

## 6. Entropy and the 128-bit ceiling

Both generators tally entropy exactly as they run — `H(p)` per Bernoulli decision, `log₂(k)` per
uniform choice — rather than estimating it. Averaged over 48 runs, cached per settings signature:

| lattice | 4×4 | 5×5 | 6×6 | 8×8 |
|---|---|---|---|---|
| hexagonal | 32 | 49 | 65 | 111 |
| rectilinear | 32 | 48 | 67 | **128** |

**Bold values are capped by the 128-bit seed, not by the generator.** Beyond 8×8 a bigger grid buys
detail, not distinctness — 8×8 is the largest size the page offers.

The bit ledger makes the whole budget visible: of 256 digest bits, ~19 go to the name, 128 seed the
mark, and **108 are never read**.

> **If you want more than 128 bits of mark**, widen the seed — `wordAt` currently reads four words
> from bytes 8–23. Reading eight would use the unread tail and lift the ceiling to 256. It changes
> every existing mark, so it is a spec decision, not a tweak.

---

## 7. Grind resistance

Quoted for the **weakest** mark on the page (the 4×4), since that is what an attacker targets.
Cost per attempt is measured live in the browser. At the defaults on a typical machine:

- per attempt: **~260 ms**, 16 MiB each
- match the name alone: `2^19.1` ≈ **42 hours**
- name + 4×4 mark: `2^51` ≈ **10⁷ years**

That gap is the design argument in one line.

**Known weakness of the figure:** it is single-core wall time on the viewer's machine, which
understates a funded attacker. The real defence is the memory cost — at 16 MiB per lane,
parallelism is bought with RAM rather than cores — but the page only states that in prose. A
"lanes per GiB" figure would make it concrete.

---

## 8. Defaults (the official parameters)

```
hash    Argon2d, 16384 KiB, 8 passes, salt stealthicons-v1
name    letters=full, capitalize=off
mark    lattice=hexagonal, shape=circle, sizes 4/5/6/8
        dot diameter 0.7, waist 1, ideal off, corner radius 1.45
        density 0.75, persistence 2, max chain 6, closure 0.5, spot share 0.5
        third colour on, fill triangles on, face-like off
        transparent off, on-dark off, mesh on
```

Reset restores all of these, including the hash and alphabet. Export/Load round-trips them as
JSON; the input text is deliberately **excluded** (it may be a passphrase). Unrecognised keys are
reported — listed inline up to five, counted beyond that.

---

## 9. Page layout

Visual results first, settings below, explanations attached to the sections they explain:

```
intro → At size (64/32/16) → Name → text input + Random text
   ── rule ──
Mark settings → Marks → Grind resistance → Name settings
→ composition → Name derivation → Bit ledger → Hashing
```

Hashing is last because those parameters are the ones a user is least likely to touch.
Font is Helvetica throughout (Stealth's official face), with a system mono for labels.

---

## 10. Working on it

There is no test runner. Everything was verified by driving the page in **jsdom** and rendering
its SVG output with **cairosvg** to look at it. That combination is worth keeping:

```js
const dom = new JSDOM(html, {runScripts:'dangerously', beforeParse(win){
  Object.defineProperty(win,'crypto',{value:webcrypto,configurable:true,writable:true});
  win.TextEncoder = TextEncoder; win.WebAssembly = WebAssembly; win.performance = performance;
  win.console.error = (...a) => console.log('CAUGHT:', a.map(x => x&&x.stack || x).join('\n'));
}});
```

That `console.error` hook matters — `update()` swallows exceptions into a catch, and without it a
crash looks like "nothing rendered".

Generators can also be pulled out of the file directly with `vm.runInContext` on the script blocks,
which is how the statistical checks (island counts, entropy, contrast audits) were run at scale.

**Things worth re-checking after any change:**

- no colour splits into islands (rectilinear), no eye carries an edge (face-like)
- every palette clears `BG_MIN` in all three host modes
- no `NaN` or `Infinity` in path data across a sweep of dot/waist/ideal combinations
- config export → reset → load round-trips exactly, with no warnings
- all four tiles and all twelve preview boxes render in both lattices

**A caution learned the hard way:** this file has been edited by scripted string splices, and twice
a splice silently deleted a function that happened to sit inside the cut range —
`scatterGrid`/`buildGrid` once, the dot-diameter listeners another time. Both times the symptom
appeared far from the cause. After any structural edit, load the page and exercise *every* control,
not just the one you changed.

---

## 11. Open items

1. **Seed width** — 128 bits caps mark entropy (§6). Decide whether to widen.
2. **Grind figure** — express parallelism in RAM, not cores (§7).
3. **Salt** — `stealthicons-v1` is versioned; settle the policy for a v2.
4. **Favicon guidance** — measured floor is ~3 pixels per cell: 4×4 and 5×5 survive 16 px, 8×8 is
   the boundary at 32 px. Not yet stated on the page.
5. **Reference implementation** — the page is the only spec. A second implementation would be the
   real test of whether §2–§4 are written precisely enough.
6. **Rectilinear sunset** — was slated for removal, then kept for its charm. Still carries
   `DIAG_RATIO` and the two-join-length awkwardness.
