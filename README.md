# Stealthicons

**Cryptographically Secure Identicons**

Stealthicons derive two orthogonal representations from a cryptocurrency address, which is tough to verify because it is hard to read:

- a **name** — five letters alternating vowel and consonant, forming a pseudoword that can be said aloud
- a **mark** — a symmetric identicon image, recognisable at a glance

Stealthicons make cryptocurrency address verification quick, easy, and secure for humans.

Stealthicons are generated purely from the input text. They are not registered or stored, so never expire or go out of sync. The same address always creates the same name and mark.

The security comes from the cost of forging a Stealthicon. A forger or phisher who wants an address that *looks* like yours must match the name and the mark together, and every guess costs a full memory-hard hash.

Four layers compound the difficulty and expense to phish you: **Argon2d** makes each attempt expensive in time and memory, the **name** narrows the target, the **mark** narrows it further, and the **address** itself still has
to be checked. Each layer doesn't simply add to the difficulty — it multiplies it. Stealthicons are cheap to verify once; absolutely punishing to grind a forgery.

---

## Try it

- **[`site/index.html`](site/index.html)** — the consumer page: paste an address, get a name and a mark, shareable via a card, which combines both with the address and an optional memo into one image, copyable or saveable with one click. This page opens directly in a browser, no server needed.
- **[`demo/stealthicons.html`](demo/stealthicons.html)** — the parameter workbench. Every dial in the Stealthicon generator is exposed for users to see what it does and settle on a look. **Its defaults are the official parameters** — leave them alone and the output matches every other conforming implementation of Stealthicons.

## Using the library

`lib/stealthicons.js` is the reference implementation, extracted as a dependency-free file — no build step, no npm install. It bundles its own copy of hash-wasm's Argon2 build and uses Web Crypto for the SHA family, so it works standalone.

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

`stealthicon(text, opts)` hashes once and returns `{ hex, name, letters, syllables, nameBits, mark }`, where `mark(size, markOpts)` returns `{ svg, bits, palette, figure }`. Every low-level piece (`digest`, `derive`, `hexFigure`, `buildGrid`, `colorsFor`, `renderFigure`, …) is also exported directly, for callers who want more control than the convenience wrapper gives — that's what the workbench itself uses. `Stealthicons.defaults` mirrors [`settings/stealthicons-defaults.json`](settings/stealthicons-defaults.json),
the official parameters as data.

---

## How it works

### Pipeline

```
text ──▶ digest ──┬─▶ bits 0..18   ──▶ name (5 letters)
                  ├─▶ bits 64..191 ──▶ xorshift128 seed ──▶ mark + palette
                  └─▶ bits 19..63, 192..255  unread
```

**Digest.** Argon2d by default: 16 MiB, 8 passes, parallelism 1, 32-byte output, fixed salt of `stealthicons-v1`. SHA-1/256/384/512 are also available via Web Crypto for comparison, though Argon2d is what a conforming implementation uses. The salt is fixed deliberately: a stealthicon must be a pure function of its address, so nothing varies from user to user, and the cost parameters carry the defense against forgery. Hash-wasm's Argon2d rejects an empty password outright, so the empty string is canonically mapped to a single space before hashing. This mapping is arbitrary, but fixed, so "no text yet" still resolves to a real stealthicon rather than an error.

**Name.** Repeated `divmod` off the low end of the first 64-bit window: one bit picks vowel-first or consonant-first "shape". Then five indices select a consonant set and a vowel set. Two alphabets — `b58` (`bcdfghjkmnpqrstvwxz` / `aeiouy`, no `l`) and `full` (adds `l`), default `full`. Costs ~19.1 bits.

**Mark.** `makeRng(hex, salt)` builds four 32-bit xorshift words from digest bytes 8–23. Every grid size and the palette use entropy from that same 128-bit slice, with salt values specific to each use.

### The two lattices

**Hexagonal** Stealthicons are the official style. They are laid out on a hexagonal grid, like a honeycomb, with the honeycomb hexagons laid on their sides, forming rows. Each dot on that grid lies at the center of a hexagon. Hexagonal Stealthicon sizes can be expressed by either odd or even numbers, and the number determines the layout (row lengths). For even numbers, the rows are n−1, n, n−1 ... For example if the number is 4 (n = 4), then the rows are 3,4,3,4. For odd numbers the rows are n−2, n−1, n, n−1, n ... For example, for n = 5, the rows are 3, 4, 5, 4, 5. Dots may be connected by "waists", which are drawn to create smooth curves. The result is "chains" that originate from the midline (vertical line of symmetry) of the image and grow outward. Chains take root on the midline and walk outward, extending with probability D / (1 + steps/W) so they thin as they lengthen; each dot may bifurcate once; adjacent leaves of one color then close with probability *Closure*. Everything is mirrored, and a dot adjacent to its own mate is always joined.

**Face-like** Stealthicons have one dot two thirds of the way across the image and a third of the way down. The mirror symmetry around the midline turns this dot into a pair of eyes. These dots carry no connections, are rendered in the accent color, and any other dot left unconnected is joined to a neighbor, taking that neighbor's color. If these lone dots have no neighbors, one will be created. A dot exactly on the midline is exempt from requiring a connection to a neighbor, having no symmetry mate. These constraints make the two mirror dots look strikingly like eyes, giving face-like icons a distinctive personality.

**Rectilinear** Stealthicons have the square grid (instead of the hexagonal grid). Each color chain grows outward from a single cell on the mirror line, so no color is ever split into islands. *Density* sets how much of the grid to fill. The density is divided between the various foreground colors of the image. The growth rule loosens as density climbs, or the image would stop growing chains at about half-full. A third shape in a third foreground color is seeded on the mirror line too, the one place it can sit without its own reflection becoming a second, disconnected island, so it appears whenever it can, given that it won't create isolated dots.

### Geometry

Circular dots are joined by two circular fillets tangent to both dots. The *Waist* slider sets the neck as a fraction of the dot's diameter and the so-called fillet radius is solved such that the arc always leaves the circumference tangentially. This means the connection never develops a corner. When the *Waist* slider is 1, the connections are a straight bar as wide as the dot diameter. **Ideal** forces the fillet radius so its center lands on a vacant lattice site, so six dots ringing an empty cell leave a hole that's a true circle. **Hexagon** dots share the lattice's own orientation, wherein their flat sides face each of their six neighbors. Hexagon connectors are as wide as the hexagon is tall, which is the distance from one vertex to the vertex diametrically across the hexagon.

### Palette

Color combinations are generated for this project: a base hue at a 15° step, the two dot hues offset by a classic harmony angle, and saturation and lightness drawn from a small curated set. Each one already clears a 5.5 contrast ratio against its own background before it's ever used; the repair step is a safety net, moving lightness alone — hues and saturations untouched — and never letting contrast fall below 3.5. Transparent drops the background entirely. The setting named On dark excludes palettes whose foreground colors don't have enough contrast on a dark background; leaving this option off excludes palettes whose foreground colors don't have enough contrast on a light background. The Transparent setting is a huge decision, and probably not advisable unless you have a very good reason. It commits your project to using either light backgrounds or dark backgrounds for your Stealthicons, which can be extra work to manage.

### Entropy and the 128-bit ceiling

The Stealthicon image is seeded from a single 128-bit slice of the cryptographic digest, which puts an upper limit on the variety of Stealthicon images: 2 to the power of 128, or 340 trillion trillion trillion. The practical consequence is that Stealthicons never need to be bigger than 8×8. Larger images can have more detail, but no more distinctness, and therefore no more security. Bear in mind that 8×8 is exceedingly secure. Both generators (hexagonal and rectilinear) tally entropy exactly as they run — `H(p)` per Bernoulli decision, `log₂(k)` per uniform choice — rather than estimating it, averaged over 48 runs:

| lattice | 4×4 | 5×5 | 6×6 | 8×8 |
|---|---|---|---|---|
| hexagonal | 32 | 49 | 65 | 111 |
| rectilinear | 32 | 48 | 67 | **128** |

Bold values are capped by the 128-bit seed, not by the generator — the rectilinear 8×8 reaches this cap; the hexagonal 8×8 comes within range. Beyond 8×8, a bigger grid buys detail, not security, explaining why 8×8 is the official size.

### Grind resistance

Grind resistance is quoted for both the weakest mark (the 4×4) and the toughest (8×8). Nothing prevents choosing the 4×4, but no one implementing custom Stealthicons for something that actually needs security — a cryptocurrency address, say — would pick it; the realistic target for an attacker is the 8×8, which is 600 billion trillion times harder to grind. Cost per attempt is estimated for a typical computer, with the Stealthicon defaults.

- per attempt: ~260 ms, 16 MiB each
- match the name alone: 2^19.1 ≈ 42 hours
- name + 4×4 mark: 2^51 ≈ 10⁷ years
- name + 8×8 mark (official): 2^130 ≈ 10³¹ years (12 million trillion trillion years)

That absurd time to grind a Stealthicon is the design argument in one line. The real defense is the memory cost: at 16 MiB per lane, parallel grinding is bought with RAM rather than cores. So the figure above (single-core wall time)
understates a funded attacker less than it might appear.

### Defaults (the official parameters)

```
hash    Argon2d, 16384 KiB, 8 passes, salt stealthicons-v1
name    letters=full, capitalize=off
mark    lattice=hexagonal, shape=circle, sizes 4/5/6/8
        dot diameter 0.7, waist 1, ideal off, corner radius 1.45
        density 0.75, persistence 2, max chain 6, closure 0.5, spot share 0.5
        third color on, fill triangles on, face-like off
        transparent off, on-dark off, mesh on
```

The workbench's "Reset defaults" restores all of these, including the hash and alphabet. Its export/load round-trips them as JSON; the input text is deliberately excluded from the exported JSON config file because it may contain sensitive information, like a passphrase.

---

## Repository layout

```
lib/stealthicons.js               the library — require() in Node, <script> in a browser
demo/stealthicons.html            the parameter workbench (the spec, made interactive)
site/index.html                   the consumer page
settings/stealthicons-defaults.json   the official parameters, as data
media/                             static assets referenced by the pages above
```

## Hosting

Stealthicons is fully static: no server-side language, no database, no build step. Any static file
host works — GitHub Pages, Netlify, S3, nginx, Apache, Caddy, a folder on a CDN, whatever you
already run.

Two things are worth knowing before you deploy it, though:

- **HTTPS (or `localhost`) is required for full functionality.** The SHA-1/256/384/512 options run
  on the browser's native Web Crypto API, which browsers restrict to secure contexts. Argon2d — the
  default, and what a conforming implementation actually uses — runs via WebAssembly and has no such
  restriction, but serving over plain HTTP will silently break the SHA comparisons in the workbench.
- **Both pages use inline `<script>` and `<style>`.** If you apply a Content-Security-Policy in
  front of them, allow inline script/style for these pages, or extract them and add your own
  nonces/hashes.

The repo doesn't presume which page lives at your domain's root — `site/index.html` is the consumer
page, `demo/stealthicons.html` is the workbench; point your host's index at whichever you want
people to land on first.

## License

MIT — see [LICENSE](LICENSE). Bundles [hash-wasm](https://github.com/Daninet/hash-wasm)'s Argon2
build (c) Dani Biro, MIT.
