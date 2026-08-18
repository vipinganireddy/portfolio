<div align="center">

<img src="assets/banner.svg" width="100%" alt="Ganireddy Sudhanshu Vipin — Web Development & Marketing">

<br>

**[↗ Open the live site](https://vipinganireddy-portfolio.vercel.app/)** &nbsp;·&nbsp;
[LinkedIn](https://linkedin.com/in/vipinganireddy) &nbsp;·&nbsp;
[GitHub](https://github.com/vipinganireddy) &nbsp;·&nbsp;
[Email](mailto:vipinganireddy@gmail.com)

<br>

<img src="assets/stack.svg" width="100%" alt="Zero dependencies, zero build step, one HTML file, six sections">

</div>

---

## The whole thing is one file

No framework. No bundler. No `node_modules`. No build step. You can open `index.html` from your desktop with the WiFi off and every pixel still works — the fog, the parallax, the 3D logos, the shooter, even the CV.

That constraint wasn't for bragging rights. It was the point of the exercise: if I let a bundler handle it, I'd learn the bundler. Doing it by hand meant I had to actually understand the scroll engine, the canvas loop and the pointer maths.

```
index.html          344 KB   ← the entire website
├─ code              92 KB   hand-written HTML + CSS + JS (1,888 lines)
├─ webp images      181 KB   fog photo, three 3D logos, campus card
└─ CV (pdf)          71 KB   embedded as base64, decoded to a Blob on demand

og-cover.jpg         34 KB   social preview card
```

Deploy is: drag both files into any static host. That's it.

---

## Hold the line

There's a real, playable space shooter at the bottom of the site. The stage is wide and short, so it runs sideways: the ship holds the left edge and the field drifts in from the right. Arrows or `W`/`S` to move, `Space` to fire — or on a phone, drag to steer and the gun runs itself. Your best score survives a reload.

<div align="center">
<img src="assets/shooter.svg" width="100%" alt="A ship holds the left of the screen and fires at drones and hulks drifting in from the right">
</div>

Nothing here is a sprite. The ship, the drones, the seven-sided hulks, the exhaust plume, the starfield and the debris are all stroked to a `<canvas>` at runtime, in the same ink the rest of the site is drawn in.

It runs on a fixed-timestep loop with an accumulator, so the physics step is `1/120s` no matter what the display is doing. Chase the frame rate instead and the ship's speed changes between a 60Hz laptop and a 120Hz phone, which is exactly the bug that makes browser games feel wrong.

```js
const STEP = 1/120;                       // the simulation tick, always

let dt = (now - last)/1000; last = now;
if (dt > 0.25) dt = 0.25;                 // tab was backgrounded — don't teleport
acc += dt;
while (acc >= STEP){ physics(STEP); acc -= STEP; if (state !== 'run') return; }
draw();
```

The awkward part isn't the loop, it's that the stage is **354px wide on a phone and 1300px on a desktop**. A fixed pixel-per-second world is frantic on one and empty on the other — enemies took six seconds to cross my laptop. Every horizontal speed is scaled against the stage width instead, so the pace feels the same on both:

```js
SC = clamp(W/700, 0.82, 1.95);            // one number, applied to every x velocity
```

> **Try this:** open the site, get past the Education section and stop scrolling. A campus pass on a lanyard swings down from the top of the page. Grab it with your mouse and throw it — it's a real pendulum with gravity, air damping and its own spin axis, and scrolling tugs the cord. *(Desktop only — it's hidden below 1180px, where there's no cursor to grab with.)*

---

## Pick a thread

<details>
<summary><b>🌫️ &nbsp;How the fog actually works</b></summary>

<br>

It's not a video and it's not a GIF. It's fractal value noise, generated per-frame on a canvas that is roughly **130 × 90 pixels**, then blurred and stretched across the whole viewport.

Three octaves of noise are summed, each drifting at its own speed, so the cloud *churns* rather than sliding as one sheet:

```js
function fbmT(x, y, t){
  let v = 0, amp = 0.5, f = 1;
  for (let o = 0; o < 3; o++){
    v += amp * vnoise(x*f + t*DR[o][0], y*f + t*DR[o][1], 11 + o*29);
    f *= 2.07; amp *= 0.5;
  }
  return v;
}
```

The tiny canvas is the whole trick. Painting ~12,000 pixels a frame is basically free; painting 2 million is not. The browser's own bilinear upscale does the softening for nothing, and a 1.25px blur hides the grid.

The noise field never resets, either — `travel` just keeps counting up, so the fog can cross the screen forever without a seam.

</details>

<details>
<summary><b>📜 &nbsp;The scroll engine, and the layout-thrash bug I had to fix</b></summary>

<br>

Everything that moves on scroll — parallax bands, the pinned experience cards, the progress bar, the hanging pass, the 3D logo tilt — runs from **one** `requestAnimationFrame` loop. Separate scroll listeners would fight each other and each one would force its own layout pass.

The version that worked but ran badly interleaved reads and writes:

```js
// ✗ every getBoundingClientRect after a style write forces a synchronous relayout
for (const card of cards) {
  const r = card.getBoundingClientRect();   // READ  → browser must recompute
  card.style.transform = …;                 // WRITE → invalidates layout again
}
```

Batching all the reads to the top of the frame fixed it, and cost nothing:

```js
// ✓ read everything first, then write everything
const cardR = cards.map(c => c.getBoundingClientRect());
const erBottom = eduEl.getBoundingClientRect().bottom;
// …all writes below this line
```

Smoothing is frame-rate independent too. A flat `lerp(a, b, 0.085)` per frame moves twice as fast on a 120Hz screen as a 60Hz one, so the factor is corrected against real elapsed time:

```js
const k = (f) => 1 - Math.pow(1 - f, dt * 60);
sy = lerp(sy, scrollY, k(.085));
```

</details>

<details>
<summary><b>📄 &nbsp;Why the CV button was broken for weeks</b></summary>

<br>

The CV is embedded in the page as base64. The obvious way to link it is the obvious way, and it does not work:

```html
<!-- ✗ two separate bugs stacked on top of each other -->
<a href="data:application/pdf;base64,JVBER…" target="_blank" download="CV.pdf">
```

1. `download` forces a save. `target="_blank"` never gets a look in.
2. Even without it, **Chrome and Firefox block top-level navigation to `data:` URLs** — it was an open-redirect and phishing vector. The tab would just fail.

The fix is to decode the base64 into a real `Blob` and hand the browser an object URL, which is a genuinely navigable resource that its built-in PDF viewer will open:

```js
const bin = atob(src.textContent.trim());
const buf = new Uint8Array(bin.length);
for (let i = 0; i < bin.length; i++) buf[i] = bin.charCodeAt(i);
btn.href = URL.createObjectURL(new Blob([buf], { type: 'application/pdf' }));
```

Decoding happens in `requestIdleCallback` so 71 KB of base64 isn't blocking first paint, with a `setTimeout` fallback for Safari. If the popup blocker eats the new tab, it falls back to the current one.

**The bug this caused:** the button starts as `href="#"`, so the smooth-scroll handler bound to `a[href^="#"]` grabbed it at load. Once the href became `blob:…`, every click threw `'blob:null/…' is not a valid selector`. The handler now re-reads the href at click time.

</details>

<details>
<summary><b>📱 &nbsp;Three mobile bugs that only show up on a real measurement</b></summary>

<br>

**The menu ate itself.** Nav list absolutely centred, footer absolutely pinned to the bottom. On a 390×844 screen "CONTACT" sat directly on top of "DUBAI"; at 640px tall a whole chunk was unreachable with no way to scroll. Below 900px it's now a scrolling flex column.

**Contact was indented 50px too far.** `.ct` is `display:flex` and `.wrap` used `margin: 0 auto` — so instead of filling the section it shrink-wrapped its content and centred itself. Every other section started at 18px; contact started at 70px. Invisible by eye, obvious the moment you measure it:

```
.ct .wrap → x: 50.7, width: 288    ✗
.ct .wrap → x: 0,    width: 390    ✓   (.ct > .wrap { width: 100% })
```

**`:hover` latches on touch.** Tap the CV button on a phone and it stays inverted until you tap elsewhere, because a touch device has no "mouse leave". Every hover effect is now behind `@media (hover: hover)`.

</details>

<details>
<summary><b>♿ &nbsp;The accessibility bits that are easy to skip</b></summary>

<br>

| | |
|---|---|
| `prefers-reduced-motion` | Kills the loader, the fog, the water, the parallax and every reveal. The site becomes a plain scrolling document — not a broken one. |
| Skip link | First tab stop jumps past the loader and the fog straight to the content. |
| Keyboard | Accordions are real `<button>`s with `aria-expanded`. Escape closes the menu. `:focus-visible` rings everywhere. |
| `<noscript>` | Everything that animates in is forced visible, so the page still reads with JS off. The CV button hides itself — a dead link is worse than no link. |
| Scroll-spy | The menu marks the section you're actually in, including the last one, which can never reach the trigger line. |
| Motion | The shooter's `aria-label` describes what's happening on the canvas. The fog, grain and cursor are `aria-hidden`. |

</details>

<details>
<summary><b>🗺️ &nbsp;Map of the file</b></summary>

<br>

| Lines | What |
|------:|------|
| `1–60` | Meta, Open Graph, JSON-LD `Person` schema, `<noscript>` fallbacks |
| `60–540` | All CSS. Ordered atmosphere → loader → chrome → menu → sections → responsive |
| `540–600` | Loader, fixed chrome, the menu, the hanging campus pass (inline SVG hardware) |
| `600–870` | Content: hero, statement, profile, skills, experience, projects, education, contact, arcade |
| `880–1010` | Fractal noise fog, calligraphy reveal, loader, word-split, IntersectionObserver reveals |
| `1010–1150` | Menu, scroll-spy, smooth anchors, accordions, custom cursor, dual clocks |
| `1150–1470` | The shooter — fixed timestep, procedural ship, drones, hulks, particles |
| `1470–1810` | Water caustics, the pendulum pass |
| `1813–1935` | The scroll engine — one rAF loop, all reads batched |
| `1935–1972` | The CV Blob decoder |

</details>

---

## Quiz: three from the codebase

Guess before you open them. No points, only pride.

<details>
<summary><b>1.</b> The fog canvas is upscaled to fill a 4K display. How many pixels does it actually draw per frame?</summary>

<br>

**About 12,000** — the canvas is clamped to roughly 130 × 90 regardless of viewport size.

```js
CW = clamp(Math.round(vw / 12), 52, 150);
CH = clamp(Math.round(vh / 12), 34, 104);
```

A full-resolution 4K fog would be ~8.3 million pixels of per-frame noise. This is ~700× less work, and after a blur and a bilinear upscale you cannot tell.

</details>

<details>
<summary><b>2.</b> The shooter's enemy AI uses negamax with alpha-beta pruning. True or false?</summary>

<br>

**False** — and it's a trick question. The shooter has no AI at all. Drones fly a straight line with a sine wobble; hulks tumble and drift. Nothing in it makes a decision. The negamax search is in **IKYK Games**, a separate project: 50 browser games in one offline HTML file, where the board games actually need something to play against.

</details>

<details>
<summary><b>3.</b> On a phone, why does the shooter fire by itself instead of on tap?</summary>

<br>

**Because a thumb can't hold a trigger and steer at the same time.**

On desktop you hold `Space` and steer with the other hand. On touch there's one finger, and it's already dragging the ship. So on coarse pointers the gun just runs while you're alive:

```js
if (firing || !FINE) fire();
```

The related problem is that dragging to steer and scrolling the page are the same gesture. The stage sets `touch-action: none` when a run begins and clears it on death, so the page only stops scrolling for exactly as long as you're playing.

</details>

---

## What's actually on the site

| Section | |
|---|---|
| **Profile** | CS undergraduate at BITS Pilani, Dubai Campus. Web development, plus field marketing. |
| **Skills** | Web development · AI development tools · Programming (C, Grade A) · Marketing & sales |
| **Experience** | **Marketing Intern**, CURD Network (Spring Technologies FZCO), Dubai — Jun–Jul 2026<br>**Web Developer Intern**, Reliance Industries Limited, Kakinada — Jun–Jul 2025 |
| **Projects** | **Deep Ocean** — an interactive clownfish reef in vanilla JS and Canvas, with steering-behaviour movement · [live](https://deep-ocean-iota.vercel.app/) · [code](https://github.com/vipinganireddy/deep-ocean)<br>**IKYK Games** — 50 browser games in a single offline HTML file, Shadow DOM isolation per game · [live](https://ikyk-games.vercel.app/) · [code](https://github.com/vipinganireddy/ikyk-games) |
| **Education** | B.E. Computer Science, BITS Pilani Dubai (2024–2028) · Intermediate MPC, Aditya, Kakinada · C-Language certificate, Grade A |

---

## Running it

```bash
git clone https://github.com/vipinganireddy/portfolio.git
cd portfolio
open index.html          # macOS   —  no server needed, no install, nothing to wait for
```

Or serve it, if you want the Blob CV and the Open Graph tags behaving exactly as they do in production:

```bash
python3 -m http.server 8000
```

**Deploying:** put `index.html` and `og-cover.jpg` in the same directory at the root of any static host — Vercel, Netlify, GitHub Pages, an S3 bucket, a Raspberry Pi. `og-cover.jpg` has to sit next to `index.html` or every link preview on WhatsApp and LinkedIn renders blank.

Two things to change if you're forking this: the URLs in the `<link rel="canonical">` and `og:url` tags near the top, and the base64 `#cvData` block at the bottom, which is my CV.

---

## Known trade-offs

I'd rather write these down than pretend they aren't there.

- **The embedded CV costs everyone 71 KB**, including the ~95% of visitors who never click it. A real `CV.pdf` next to `index.html` would be lighter and behaves better on older iOS Safari, which is occasionally awkward with Blob PDFs. Keeping it inline is a deliberate choice to preserve the one-file property, not an oversight.
- **The loader holds you for 2.4 seconds.** It's a real progress counter over a real font-load, not a fake spinner, but it is still 2.4 seconds you didn't ask for. It was 3.6.
- **181 KB of images are base64, so they can't be cached separately** or served as WebP/AVIF by a CDN. Same trade: one file, or good caching. Not both.
- **The pass and the drag-to-spin logos are desktop-only.** Most traffic is mobile, so most people never see the fiddliest thing on the site. I know.

---

<div align="center">
<br>

**Ganireddy Sudhanshu Vipin**

Dubai, United Arab Emirates &nbsp;·&nbsp; [vipinganireddy@gmail.com](mailto:vipinganireddy@gmail.com)

<sub>Written by hand. Fog included.</sub>

</div>
