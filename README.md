# Stealthicons

**A name and a mark, from the address itself.**

An address is a wall of characters nobody reads. Stealthicons derives two handles from it instead:

- a **name** — five letters alternating vowel and consonant, so it can be said aloud
- a **mark** — a symmetric identicon, recognisable at a glance

Both are pure functions of the input text. Nothing is stored, so nothing can drift out of sync.

The security argument is the cost of forging one. An attacker who wants an address that *looks*
like yours has to match the name and the mark together, and every guess costs a full memory-hard
hash. Four things compound: **Argon2d** makes each attempt expensive in time and memory, the
**name** narrows the target, the **mark** narrows it further, and the **address** itself still has
to be checked. Cheap to verify once; punishing to grind.

---

## Try it

- **[`site/index.html`](site/index.html)** — the consumer page: paste an address, get a name and a
  mark, plus a compact shareable card combining both with the address (and an optional free-text
  memo) into one image, copyable or saveable in a click. Opens directly in a browser, no server
  needed.
- **[`demo/stealthicons.html`](demo/stealthicons.html)** — the parameter workbench. Every dial in
  the generator is exposed here so you can see what it does and settle on a look. **The defaults
  it opens with are the official parameters** — leave them alone and the output matches every
  other conforming implementation.

## Using the library

`lib/stealthicons.js` is the reference implementation, extracted from the workbench as a
dependency-free file — no build step, no npm install. It bundles its own copy of hash-wasm's
Argon2 build, so it works standalone.

**Node:**

```js
const Stealthicons = require("./lib/stealthicons.js");

const icon = await Stealthicons.stealthicon("your address or any text");
console.log(icon.name);              // "hazub"
const mark = icon.mark(8);           // the official size
console.log(mark.svg);               // a standalone <svg>...</svg> string
```

**Browser:**

```html
<script src="lib/stealthicons.js"></script>
<script>
  Stealthicons.stealthicon(text).then(function(icon){
    document.body.innerHTML = icon.name + icon.mark(8).svg;
  });
</script>
```

`stealthicon(text, opts)` hashes once and returns `{ hex, name, letters, syllables, nameBits, mark }`,
where `mark(size, markOpts)` returns `{ svg, bits, palette, figure }`. Every low-level piece
(`digest`, `derive`, `hexFigure`, `buildGrid`, `colorsFor`, `renderFigure`, …) is also exported
directly, for callers that want more control than the convenience wrapper gives — that's what the
workbench itself uses. `Stealthicons.defaults` mirrors [`settings/stealthicons-defaults.json`](settings/stealthicons-defaults.json),
the official parameters as data.

---

## How it works

### Pipeline

```
text ──▶ digest ──┬─▶ bits 0..18   ──▶ name (5 letters)
                  ├─▶ bits 64..191 ──▶ xorshift128 seed ──▶ mark + palette
                  └─▶ bits 19..63, 192..255  unread
```

**Digest.** Argon2d by default: 16 MiB, 8 passes, parallelism 1, 32-byte output, fixed salt
`stealthicons-v1`. SHA-1/256/384/512 are also available via Web Crypto for comparison, though
Argon2d is what a conforming implementation uses. The salt is fixed deliberately: a stealthicon
must be a pure function of its address, so there is nothing per-user to vary, and the cost
parameters carry the defence. hash-wasm's Argon2d rejects a literal empty password outright, so
the empty string is canonically mapped to a single space before hashing — arbitrary, but fixed,
so "no text yet" still resolves to a real stealthicon rather than an error.

**Name.** Repeated `divmod` off the low end of the first 64-bit window: one bit picks vowel-first
or consonant-first, then five indices into a consonant set and a vowel set. Two alphabets — `b58`
(`bcdfghjkmnpqrstvwxz` / `aeiouy`, no `l`) and `full` (adds `l`), default `full`. Costs ~19.1 bits.

**Mark.** `makeRng(hex, salt)` builds four 32-bit xorshift words from digest bytes 8–23. Every grid
size and the palette draw from that same 128-bit slice with different salt values.

### The two lattices

**Hexagonal** puts dots on a triangular lattice addressed in HECS, long rows of *n* alternating
with short rows of *n−1* set half a cell right, so all six neighbours are one unit away and every
waist is identical. Odd grids run short-long-short and even grids drop the first and last dot of
their top row, keeping every corner clear of the crop. Chains take root on the midline and walk
outward, extending with probability `D / (1 + steps/W)` so they thin as they lengthen; each dot may
bifurcate once; adjacent leaves of one colour then close with probability *Closure*. Everything is
mirrored, and a dot adjacent to its own mate is always joined.

**Face-like** seats one dot two thirds of the way across and a third of the way down, and lets the
mirror turn it into a pair of eyes. What makes them read as eyes is that they carry no connections,
so the accent colour is reserved for them and the body chains use only the other two. Any other dot
left unconnected is joined to a neighbour, taking that neighbour's colour if need be, or given a
neighbour of its own. A dot exactly on the midline is exempt, having no symmetry mate.

**Rectilinear** keeps the square grid: each colour grows outward from a single cell on the mirror
line, so no colour is ever split into islands. *Density* sets how much of the half-grid to fill and
the colours divide that between them; the growth rule loosens as density climbs, or the dial would
saturate around half fill. A third shape is seeded on the mirror line too — the one place it can sit
without its own reflection becoming a second, disconnected island — so it appears whenever it can be
had without floaters.

### Geometry

Circular dots are joined by two circular fillets tangent to both. The *Waist* dial sets the neck as
a fraction of the dot's diameter and the fillet radius is solved to match, so the arc always leaves
the circumference tangentially and the joint never develops a corner — until at 1 the pinch vanishes
and the connector is a straight bar as wide as the dots. **Ideal** forces the fillet radius so its
centre lands on a vacant lattice site, so six dots ringing an empty cell leave a hole that's a true
circle. **Hexagon** dots share the lattice's own orientation, a flat side facing each of the six
neighbours, and their connector is always as wide as the hexagon is tall, vertex to vertex.

### Palette

725 colour combinations: 337 from [Nanoidenticons](https://github.com/keerifox) (WTFPL) plus 388
generated for this project across three families — dark grounds with bright dots, light grounds
with deep dots, mid grounds with dots at both extremes — over 24 base hues and 12 hue schemes.

Nanoidenticons' own combinations were tuned for its renderer and many wash out here. Each is
repaired on first use, moving lightness only so hues and saturations survive, aiming for a 5.5
contrast ratio against its ground and never falling below 3.5. A third colour is derived rather than
stored: the hue furthest from both dot hues, with the stronger saturation and whichever lightness
stands further from the ground. **Transparent** mode drops the background rect; the host surface
named by *On dark* then acts as the ground, and combinations whose dots wouldn't clear it are
filtered out rather than altered.

### Entropy and the 128-bit ceiling

The mark is seeded from a single 128-bit slice of the digest, so however much variety a generator
would spend, the count can never exceed 2¹²⁸. Both generators tally entropy exactly as they
run — `H(p)` per Bernoulli decision, `log₂(k)` per uniform choice — rather than estimating it,
averaged over 48 runs:

| lattice | 4×4 | 5×5 | 6×6 | 8×8 |
|---|---|---|---|---|
| hexagonal | 32 | 49 | 65 | 111 |
| rectilinear | 32 | 48 | 67 | **128** |

Bold values are capped by the 128-bit seed, not by the generator — the rectilinear 8×8 already
reaches it, the hexagonal 8×8 comes within range. Beyond 8×8 a bigger grid buys detail, not
distinctness, which is why 8×8 is the official size.

### Grind resistance

Quoted for the weakest mark (the 4×4), since that's what an attacker targets. Cost per attempt is
measured live in the browser (the workbench shows it for your machine); illustratively, at the
defaults on a typical machine:

- per attempt: ~260 ms, 16 MiB each
- match the name alone: 2^19.1 ≈ 42 hours
- name + 4×4 mark: 2^51 ≈ 10⁷ years

That gap is the design argument in one line. The real defence is the memory cost — at 16 MiB per
lane, parallelism is bought with RAM rather than cores, so the figure above (single-core wall time)
understates a funded attacker less than it might look.

### Defaults (the official parameters)

```
hash    Argon2d, 16384 KiB, 8 passes, salt stealthicons-v1
name    letters=full, capitalize=off
mark    lattice=hexagonal, shape=circle, sizes 4/5/6/8
        dot diameter 0.7, waist 1, ideal off, corner radius 1.45
        density 0.75, persistence 2, max chain 6, closure 0.5, spot share 0.5
        third colour on, fill triangles on, face-like off
        transparent off, on-dark off, mesh on
```

The workbench's "Reset defaults" restores all of these, including the hash and alphabet. Its
export/load round-trips them as JSON; the input text is deliberately excluded from that (it may be
a passphrase).

---

## Repository layout

```
lib/stealthicons.js               the library — require() in Node, <script> in a browser
demo/stealthicons.html            the parameter workbench (the spec, made interactive)
site/index.html                   the consumer page
settings/stealthicons-defaults.json   the official parameters, as data
```

## License

MIT — see [LICENSE](LICENSE). Bundles [hash-wasm](https://github.com/Daninet/hash-wasm)'s Argon2
build (c) Dani Biro, MIT, and a palette originating in
[Nanoidenticons](https://github.com/keerifox) (WTFPL).
