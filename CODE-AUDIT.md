# Klaarcijfer Demo — Code Audit

Audit target: `index.html` (1239 lines, single-file landing page, Tailwind CDN + Lenis 1.3.23 + lenis-snap).

Findings are grouped by severity. Each item is paired with file location, root cause, why it matters, and a ready-to-paste fix.

---

## Critical bugs (cause the user's complaints, or block functionality)

### C1. `#diensten` content is taller than 100svh — content gets clipped when snapped to "start"

**Where:** `.diensten-section` (line 295–301) + the section markup (line 659–745).

**What's wrong:** The section is declared `min-height: 100svh`, but the inner content is much taller than one viewport on most desktops. Counting: outer wrapper has `py-24 md:py-36` (96–144px top and bottom = 192–288px vertical padding), then `mb-12 md:mb-20` (48–80px) under the heading, then **four** `<article>` cards each with `py-10 md:py-14` (80–112px vertical = 320–448px total just for service rows). Plus four service titles at `text-[40px]` desktop, paragraphs, etc. Real measured content height on a 1440×900 desktop is roughly 1500–1700 px — well over a 900-px viewport. With `Snap` `align: 'start'`, when this section is the snap target Lenis aligns its top edge to the viewport top; the bottom 600–800 px sit below the fold and you can't scroll inside the section because the next wheel triggers a snap to `#werkwijze`.

**Why it matters:** This is exactly the user's complaint #1: "Page 2 is not fully visible when snapped to."

**Fix (option A — preferred): allow this section to be tall, and register a second snap point at the bottom (same trick used for FAQ).**

```html
<!-- line 659 -->
<section id="diensten"
         class="diensten-section section-tall relative z-10"
         data-section data-theme="dark">
```

```js
// no JS change needed — the existing branch already handles `.section-tall`:
//   if (el.classList.contains('section-tall')) snap.addElement(el, { align: ['start', 'end'] });
```

**Fix (option B): drop `min-height: 100svh` on `.diensten-section` and let it be content-height — but then the section won't snap-fill on tall monitors, so option A is better.**

**Fix (option C): trim padding (`py-12 md:py-20`) and shorten article rows (`py-6 md:py-8`) so the whole section actually fits 100svh on a 1080p screen.** Combine with option A on smaller screens.

---

### C2. `#werkwijze` has no `min-height` — section is shorter than viewport, so the next section bleeds in

**Where:** Line 748, `<section id="werkwijze" class="bg-forest text-cream relative" ...>` — no `.diensten-section`-style class, no `min-height` on the section, no `100svh` rule for `[data-theme="dark"]`. Padding is `py-20 md:py-32` (80–128px) plus 3 short step columns. Actual height ≈ 470–700 px depending on viewport — well under 900 px.

**What's wrong:** Snap aligns the top of `#werkwijze` to the viewport top. Because the section is shorter than the viewport, the orange/cream `#tarieven` section starts rendering ~500 px down the page and visually bleeds into the bottom of the viewport.

**Why it matters:** This is the user's complaint #2: "Page 3 doesn't appear vertically centered — page 4 bleeds in below."

**Fix:** Make every snap target at least 100svh AND vertically center its content (so when the section is taller than its content, the content stays in the optical center of the viewport instead of pinned to the top).

Add a generic helper at the bottom of the `<style>` block (next to `.diensten-section`):

```css
/* Every primary snap target fills the viewport so snap-to-start aligns cleanly */
.snap-pane {
  min-height: 100svh;
  display: flex;
  flex-direction: column;
  justify-content: center;
}
```

Then apply it to the three offending sections:

```html
<!-- line 748 -->
<section id="werkwijze" class="snap-pane bg-forest text-cream relative" data-section data-theme="dark">

<!-- line 794 -->
<section id="tarieven" class="snap-pane relative z-10" data-section data-theme="light">

<!-- line 990 -->
<section id="contact" class="snap-pane bg-ink text-cream relative overflow-hidden" data-section data-theme="dark">
```

(Hero already has its own `min-height: 100svh` + flex-center via `.hero-section`. `#diensten` already has `min-height: 100svh` but its content overflows — see C1. `#faq` is already correctly tagged `.section-tall` and snaps top+bottom.)

---

### C3. Footer is registered as a snap target — produces a near-empty 8th snap point

**Where:** Line 1035, `<footer ... data-section data-theme="dark" id="footer">` and line 1088 `const sections = Array.from(document.querySelectorAll('[data-section]'));` then line 1182 `sections.forEach(el => { ... snap.addElement(el, ...) })`.

**What's wrong:** The footer has `data-section`, so it's registered as a snap point. The footer is short (≈ 380 px), so when the user scrolls past `#contact` they get yanked into a position where the footer top is at the top of the viewport — leaving most of the viewport showing the bottom of `#contact` above the footer. This produces the "extra" snap point the user observed (8 snap points for 7 sections) and creates an awkward dead-stop at the page bottom.

**Why it matters:** Bad UX at the bottom of the page; also pollutes the dot indicator (the `#contact` dot stops being active too early).

**Fix:** Remove `data-section` from the footer. It can keep `id="footer"` and `data-theme` for any future styling.

```html
<!-- line 1035 -->
<footer class="bg-ink text-cream/70 rule-top border-cream/15" id="footer">
```

Also remove the now-orphan `<li>` for the footer in the dot nav — actually there isn't one (good), so no further change needed.

---

### C4. Snap option `distanceThreshold: '500%'` — string format is invalid; should be a number or `Infinity`

**Where:** Line 1169.

**What's wrong:** The lenis-snap module expects `distanceThreshold` as a number of pixels (or `Infinity` to disable proximity gating). Passing the string `'500%'` is silently coerced to `NaN` inside the module's distance check, and the proximity gate either always passes or always fails depending on the comparison direction. This is harmless when `type: 'lock'` is used (because lock mode locks to the next/prev anyway), but it is dead code that future-readers will trip over, AND if someone changes `type` to `'mandatory'` or `'proximity'` the snap will misbehave.

**Why it matters:** Latent bug; produces console-quiet failure if the type is changed.

**Fix:**

```js
// line 1166
const snap = new Snap(lenis, {
  type: 'lock',
  // distanceThreshold removed — irrelevant in 'lock' mode.
  // If you ever switch to 'proximity' or 'mandatory', set: distanceThreshold: Infinity
  lerp: 0.1,
  duration: 1.1,
  easing: t => t < 0.5
    ? 4 * t * t * t
    : 1 - Math.pow(-2 * t + 2, 3) / 2,
  debounce: 0,
});
```

---

### C5. Hero `padding-top: 130px` is hard-coded but the absolute header is ~96px tall

**Where:** Line 85 `.hero-section { padding-top: 130px; }`. The header (line 526–548) has `py-4` (16px top+bottom = 32px) plus a 36px logo plus an announcement bar at line 513–523 with `py-2.5` (10px top+bottom = 20px) plus 13-px line-height text — total stack ≈ 36 (logo) + 32 (header padding) + ~40 (announcement bar including text) = roughly 108 px on desktop.

**What's wrong:** 130 px is 22 px more than needed → the hero headline sits with extra dead space at the top. Hero also has `pt-12 md:pt-20` (48–80 px) on the inner `<div>` (line 566), so the *real* top padding is 130 + 80 = 210 px. The headline starts ~210 px down the viewport, which in combination with `display: flex; justify-content: center` (line 287–292) actually pushes content far below center on shorter monitors and may cause the bottom of the hero to clip on a 900-px viewport.

**Why it matters:** Misalignment, wasted vertical space at the top of the hero, and (on short monitors) hero content can clip below the fold.

**Fix:** Replace the hard-coded 130 px with a CSS variable measured from the actual header at runtime, or just shrink it and let the inner `pt-12` carry the visual gap:

```css
/* line 85 */
.hero-section { padding-top: 0; }                /* let inner pt-* do the work */
.hero-section > .relative { padding-top: clamp(96px, 14vh, 160px); }
```

Or simpler — use `pt-32 md:pt-40` Tailwind classes on the inner div (line 566) and drop `.hero-section { padding-top: 130px }` entirely. The `display: flex; justify-content: center` will then center the content and the absolute header floats over the top of it cleanly.

---

## Real bugs (subtle issues that should be fixed)

### R1. IntersectionObserver "spy" thresholds will never fire for sections taller than viewport

**Where:** Line 1104–1112.

**What's wrong:** `threshold: [0.4, 0.6]` only fires when 40–60% of the *target* is visible relative to *itself*. For a section that is 1700 px tall on a 900-px viewport, the section can never be more than 900/1700 = 53% visible — so the 60% callback never fires, and the 40% boundary triggers only briefly mid-scroll. Active dot can stick on the previous section. Conversely, very short sections (footer, `#werkwijze` if not made full-height) may never reach 40% if they're squeezed by adjacent sections.

**Why it matters:** Active dot indicator drifts out of sync with the user's actual position.

**Fix:** Use `rootMargin` to shift the observation rectangle to a thin viewport-center band, and lower the threshold:

```js
const spy = new IntersectionObserver((entries) => {
  entries.forEach(e => {
    if (e.isIntersecting) {
      setActive(e.target.id);
      setTheme(e.target.dataset.theme || 'light');
    }
  });
}, {
  // Observe only the middle 1px-tall horizontal band of the viewport.
  // Whichever section's body is crossing the viewport center "wins".
  rootMargin: '-50% 0px -50% 0px',
  threshold: 0,
});
sections.forEach(s => spy.observe(s));
```

This pattern is robust to sections of any height.

---

### R2. Anchor click handler binds to ALL `a[href^="#"]` — including `#hero`, dot links, nav links, and footer links — but `lenis.scrollTo(t)` with a DOM element is fine; however on initial-load the same handler also hits the `#contact` button inside the hero. Not a bug per se. **However:** every anchor goes through the same `duration: 1.2`, including dot taps which feel slow.

**Where:** Line 1202–1215.

**What's wrong:** A dot tap to jump 3 sections takes 1.2s — acceptable. A dot tap to jump 1 section also takes 1.2s — feels sluggish.

**Why it matters:** Polish.

**Fix (optional):** Scale duration with distance:

```js
lenis.scrollTo(t, {
  offset: 0,
  duration: Math.min(1.4, 0.6 + Math.abs(window.scrollY - t.offsetTop) / 4000),
  easing: x => 1 - Math.pow(1 - x, 4),
});
```

---

### R3. `e.preventDefault()` is called even when target is the same section (no-op scroll)

**Where:** Line 1208.

**What's wrong:** Clicking the active dot still fires `preventDefault()` and a 1.2-second scroll-to-self. Harmless but wasteful.

**Fix:**

```js
if (Math.abs(window.scrollY - t.offsetTop) < 4) return;  // already there
```

---

### R4. `history.scrollRestoration = 'manual'` is set inside an `if (location.hash ...)` block — never reset on subsequent navigations

**Where:** Line 1221.

**What's wrong:** Once set to `'manual'`, browser back/forward navigation will not restore scroll on this page. For a single-page site this is mostly fine, but on `bfcache` restore the user lands at the top instead of where they were.

**Fix:** Either set it unconditionally at the top of the IIFE, or leave the browser default (`'auto'`) and rely on the lenis `scrollTo({ immediate: true })` to override:

```js
// line 1220
if (location.hash && document.querySelector(location.hash)) {
  requestAnimationFrame(() => {
    lenis.scrollTo(location.hash, { immediate: true, force: true });
  });
}
```

---

### R5. `snap.resize()` may not be a function — guarded with `&&` but never tested

**Where:** Line 1233 `resizeTimer = setTimeout(() => snap.resize && snap.resize(), 150);`.

**What's wrong:** lenis-snap `1.3.x` does not expose a public `resize()` method — snap points are recomputed automatically when their underlying elements move. The `&&` guard makes this a silent no-op; the `setTimeout` and `clearTimeout` are dead code.

**Fix:** Drop the listener entirely, or call `lenis.resize()` (which IS a public method):

```js
window.addEventListener('resize', () => {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => lenis.resize(), 150);
});
```

---

### R6. `touchMultiplier: 1.4` is aggressive on iOS — may cause overshoot

**Where:** Line 1154.

**What's wrong:** With Lenis `smoothWheel: true` and the snap module locking direction, a single iOS swipe is already amplified. `1.4×` makes a small flick travel two or three sections, which fights the lock-snap.

**Fix:**

```js
touchMultiplier: 1,
```

(Or test 1.0 vs 1.2 on a real device — 1.4 is the default upper bound.)

---

### R7. Multiple `[data-svc]` cards live inside the dark `.diensten-section`, but the spotlight gradient uses a fixed `--mx`/`--my` set only on `mousemove` — never reset on `mouseleave`

**Where:** Line 1117–1123.

**What's wrong:** When the cursor leaves a card, the spotlight stays at the last position (frozen). When you mouse-back-in across a different edge, the gradient pops because the position hasn't been reset.

**Fix:**

```js
document.querySelectorAll('[data-svc]').forEach(card => {
  card.addEventListener('mousemove', (e) => {
    const r = card.getBoundingClientRect();
    card.style.setProperty('--mx', (e.clientX - r.left) + 'px');
    card.style.setProperty('--my', (e.clientY - r.top) + 'px');
  });
  card.addEventListener('mouseleave', () => {
    card.style.removeProperty('--mx');
    card.style.removeProperty('--my');
  });
});
```

---

### R8. `<img alt="">` with no `alt` text on the logo is fine for decorative use, but the immediately-following text is `"Mentink op maat"` — screen readers will hear the brand name twice if the alt is restored. Currently it's empty (line 529: `alt=""`). OK as-is, but worth a comment.

---

### R9. `details` elements use the native `<details>` toggle — Lenis does not interfere, but the click target is the `<summary>`, not the chevron. Tap area on mobile is fine. No fix needed; mentioned for completeness.

---

## Code smell / dead code

### S1. Lenis baseline CSS `html.lenis, html.lenis body { height: auto; }` — redundant when not running Lenis without document context

**Where:** Line 51–55.

**What's wrong:** These rules are correct but include `.lenis-stopped { overflow: hidden }` — there's no place in this code that calls `lenis.stop()`, so the rule is dead. Keep `.lenis.lenis-smooth [data-lenis-prevent]` and `.lenis.lenis-smooth iframe`; drop the others.

---

### S2. Globals `window.lenis` and `window.snap` are set "for debug"

**Where:** Lines 1156, 1177.

**What's wrong:** Fine for development but should be guarded for production:

```js
if (location.hostname === 'localhost' || location.hostname.startsWith('127.')) {
  window.lenis = lenis;
  window.snap = snap;
}
```

---

### S3. Unused commented-out testimonial section

**Where:** Lines 907–909.

**What's wrong:** Just a comment placeholder. Either remove or replace with a real testimonial. Harmless but adds noise.

---

### S4. Dead `.marquee` / `.marquee-track` CSS — no marquee element exists in the markup

**Where:** Lines 394–396.

**What's wrong:** ~150 bytes of unused CSS. Drop it.

---

### S5. `data-svc` attribute exists but only used by the spotlight handler — no other consumers

**Where:** All four `<article>` tags in `#diensten`.

**What's wrong:** Harmless; could just use `.svc-card` selector. Not worth changing.

---

### S6. Comment claims "the snap module debounces by destination, not source" — no longer true since `debounce: 0` is set explicitly

**Where:** Line 1160–1175.

**What's wrong:** Comment is misleading. With `debounce: 0`, every wheel event re-targets, which is the desired behavior — but the comment implies destination-debouncing is a feature of the module rather than something you opted into.

**Fix:** Tighten the comment or just remove the multi-paragraph explanation; keep `debounce: 0,` with a one-line "// re-target on every wheel".

---

### S7. `animScale = reduceMotion ? 0.55 : 1` is computed but only applied to `lenis.duration` — the *snap* duration is hard-coded at 1.1

**Where:** Line 1144 + line 1171.

**What's wrong:** Reduced-motion users still get a 1.1-second snap animation. Either honor the variable everywhere or drop it.

**Fix:**

```js
// line 1171
duration: 1.1 * animScale,
```

---

### S8. `setTheme` toggles a class that only changes dot styling — but the `data-theme` attribute is also a useful hook for future dark-aware components. OK to keep.

---

## Polish suggestions

### P1. `100svh` is supported in iOS 15.4+ / Chrome 108+ / Firefox 110+; older browsers fall back to invalid value. Add a `100vh` fallback:

```css
.hero-section, .diensten-section, .snap-pane {
  min-height: 100vh;       /* fallback */
  min-height: 100svh;
}
```

### P2. `backdrop-filter` on `.glass-card`, `.price-card`, `.scroll-dots ul` — paired with `-webkit-backdrop-filter`. Good. But Firefox-on-Android does not support either; consider a fallback `background: rgba(251,249,242,0.85)` for `@supports not (backdrop-filter: blur(1px))`.

### P3. Header `z-25` is not a default Tailwind value — you'd need `class="z-[25]"` or extend the config. Currently it's only set in CSS (`.site-header { z-index: 25; }` line 74), which is fine — but the Tailwind class `z-30` on the announcement bar (line 513) could conflict. Currently announcement is z-30, header z-25, dots z-40, hero z-10 — no actual conflict, but a comment explaining the layer cake would help.

### P4. No `aria-current="true"` on the active dot or nav pill — only a class. Add for screen readers:

```js
const setActive = (id) => {
  dotLinks.forEach(l => {
    const isActive = l.dataset.target === id;
    l.classList.toggle('is-active', isActive);
    if (isActive) l.setAttribute('aria-current', 'true'); else l.removeAttribute('aria-current');
  });
  navLinks.forEach(l => {
    const isActive = l.getAttribute('href') === '#' + id;
    l.classList.toggle('is-active', isActive);
    if (isActive) l.setAttribute('aria-current', 'true'); else l.removeAttribute('aria-current');
  });
};
```

### P5. The two floating glass cards on the hero (line 620, 635) use `position: absolute` with `top: 6` / `bottom: -5` — on viewport widths between 1024 and 1280 px (the `lg:` breakpoint), the right card's `-right-3` puts it just inside the viewport, but the photo's `max-w-[420px]` plus floating cards may exceed the column width and overlap text. Consider `lg:-right-8` minimum.

### P6. The cinematic background image `assets/cinematic-bg.jpg` is loaded eagerly and uncompressed via `background-image: url(...)` — no `loading="lazy"` is possible on a CSS background, but you can defer it with a small JS trick:

```js
window.addEventListener('load', () => {
  document.querySelector('.diensten-bg').style.backgroundImage = "url('assets/cinematic-bg.jpg')";
});
```

(Remove the `background-image` from the static rule.)

### P7. Multiple `<h1>` would be a problem — checked: only one `<h1>` exists (line 579). Heading hierarchy is correct: `h1` → `h2` (sections) → `h3` (services / pricing tier names / FAQ summaries) → no skips.

### P8. `<a href="tel:...">` and `<a href="mailto:...">` — fine; have visible text.

### P9. SVG icons all have `aria-hidden="true"` or no `<title>` — fine for decorative use but ensure outer link/button has accessible text. All checked: each SVG is paired with text.

### P10. Tailwind CDN in production — slow first paint and no purge. For production, swap to a built `tailwind.css`. (Out of scope for this audit but flagging.)

---

## Prioritized fix order — top 5 to resolve user complaints

These are the changes I'd ship first, in order:

1. **C1 — Mark `#diensten` as `.section-tall` so it gets two snap points (top + bottom).** Single class change. Resolves "page 2 not fully visible." Line 659.

2. **C2 — Add a `.snap-pane { min-height:100svh; flex-center }` helper and apply it to `#werkwijze`, `#tarieven`, `#contact`.** Resolves "page 3 not centered, page 4 bleeds in." 4-line CSS + 3-class additions.

3. **C3 — Remove `data-section` from `<footer>`.** Eliminates the spurious 8th snap point and the awkward dead-stop. One attribute removal.

4. **R1 — Replace IntersectionObserver thresholds with `rootMargin: '-50% 0px -50% 0px'`.** Active dot will now correctly track which section crosses the viewport center, regardless of section height. ~5-line JS change.

5. **C5 — Drop `.hero-section { padding-top: 130px }` and let the inner `pt-12 md:pt-20` carry the gap (or replace with `clamp()`).** Hero content recentered correctly under the absolute header on all viewport heights.

After these five, ship C4 (remove invalid `distanceThreshold`), R5 (fix the resize handler), and R6 (`touchMultiplier: 1`) as a small follow-up.
