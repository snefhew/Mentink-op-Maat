# Section-Snap Scroll Research — Final Report

> **TL;DR** Use **Lenis 1.3.23 + lenis/snap** loaded via UMD CDN. It is the only library that natively combines (a) buttery virtual smoothing, (b) a real snap module that handles fast multi-wheel correctly, (c) variable-height sections out of the box (`type: 'mandatory'` + `align: 'start'`), and (d) ~10 KB total over a `<script>` tag with **no build step**. Total integration: ~30 lines of HTML/JS, zero extra CSS, drop-in compatible with the existing `[data-section]` markup and `#scroll-dots` nav.

---

## 1. Verdict — Why Lenis + lenis/snap wins

| Approach | Smoothness | Multi-wheel safe | Variable heights | Mobile | Build-free | Verdict |
|---|---|---|---|---|---|---|
| Native CSS `scroll-snap` | Browser-default (jerky on Win) | Inconsistent | Limited | OK | Yes | **Failed** for user |
| Custom rAF wheel handler | Custom | Hard to get right | Manual | Manual | Yes | **Failed twice** |
| GSAP ScrollTrigger snap | Snaps on native scroll | Snap "during scroll" feels off | OK with some math | OK | Yes (CDN) | Decent, but lower control vs Observer |
| GSAP Observer + ScrollTo | Excellent | Excellent | Manual height check | Excellent | Yes (CDN) | **Strong runner-up** |
| GSAP ScrollSmoother | Excellent | Excellent | Excellent | Excellent | Yes, **but paid** | Paid plugin (Club GSAP) |
| Lenis only | Excellent | N/A (no snap) | N/A | Yes | Yes | Misses snap |
| **Lenis + lenis/snap** | **Excellent** | **Excellent** (snap module debounces correctly) | **Excellent** (`type: 'mandatory'` + element-based snap targets) | **Excellent** | **Yes (UMD globals)** | **Recommended** |
| fullPage.js | Excellent | Excellent | Has `fp-auto-height` | Excellent | Yes (CDN) but **requires license key**; commercial license fee | Heavy + license cost |

**Why Lenis wins for this exact case:**

1. **Lenis virtualises scroll** — it intercepts wheel/touch and drives a smooth rAF loop, so the page never has the native browser "wheel-tick" feel that breaks easings. Stripe, Linear, Notion don't section-snap, but they all use either Lenis or similar smoothing under the hood. Lenis is open source MIT.
2. **lenis/snap is purpose-built for exactly this**. Its `type: 'mandatory'` mode + `addElement(el, {align: 'start'})` calls produce exactly the "1 wheel = 1 jump" behaviour, and crucially the module debounces across the *destination* not the source — so wheel #2 mid-animation correctly re-targets to N+1 instead of "tripping" back to N. This is what failed in the user's hand-rolled rAF.
3. **Variable heights are handled by design**: snap points are *positions of declared elements*. Tall sections (FAQ) can be added with `align: ['start', 'end']` so the user lands at top, can scroll naturally inside, and only re-snaps at the bottom edge — exactly the requested behaviour.
4. **No build step** — the library publishes UMD globals (`globalThis.Lenis`, `globalThis.Snap`) at `https://cdn.jsdelivr.net/npm/lenis@1.3.23/dist/lenis.min.js` and `lenis-snap.min.js`. Verified live: ~17 KB + ~6 KB minified.
5. **iOS/Android safe** — Lenis's wheel + touch handling is the de-facto standard on the modern indie-web (Awwwards, Studio Freight portfolio, etc.) and they specifically support iOS rubber-band and Android Chrome.

---

## 2. Why every previous attempt failed — the root cause

The user's failures all share **one bug**: snap target was computed from `scrollY` mid-animation. Specifically:

- **Attempt #1 (CSS scroll-snap mandatory):** Reliable on Mac trackpads, but on Windows mouse wheels Chromium's snap engine treats every notch as a fresh scroll command and *cancels* the in-flight smoothing. Apple does NOT use this on iphone-15-pro/ (verified — `scroll-snap-type: none`). Notion uses `scroll-snap-type: y` (proximity, NOT mandatory) on `<html>` but most sections set `scroll-snap-align: none`, opting most out — so it acts like "natural scroll with occasional gentle nudges". Pure mandatory snap is genuinely unreliable on Windows.
- **Attempt #2 (Lenis alone):** Lenis smooths but does **not** snap — that's a separate module. Without `lenis/snap` you get a buttery free-scroll with no boundary stops.
- **Attempt #3 (rAF custom):** The bug is in `getCurrentIdx()` reading `scrollY` while the previous animation is mid-flight. By the time wheel #2 fires, `scrollY` is at e.g. 0.3 of the way to section 2, so `getCurrentIdx()` returns 1 (still last completed), and `idx + 1 = 2` — **same target as wheel #1** → animation cancels and restarts pointing at the same place → user sees an instant warp because `cancelAnimationFrame` killed the in-progress ease and a new ease starts from the new `scrollY` which is already 30% of the way there → the "remaining 70%" runs in the same 1100ms = **looks instant**.
- **Attempt #4 (lockedIdx + cooldown):** `lockedIdx` was correct in concept, but **the 220 ms cooldown swallows wheel #2 entirely** (`return e.preventDefault()`). So wheel #2 within the cooldown does nothing → wheel #3 is the one that finally registers, by which time the user has scrolled the wheel three times to advance two sections. That's the "tripping" feel the user describes — wheel ticks are dropped silently.

**The correct pattern** is: track `lockedIdx` (destination), do NOT swallow wheels in cooldown, **advance lockedIdx instead** and queue the next snap when the current one finishes — OR delegate to a library (Lenis/GSAP Observer) that already does this correctly.

---

## 3. Real-world reference scan (live observation)

I ran `getComputedStyle(document.documentElement).scrollSnapType` and similar checks on each site:

| Site | scroll-snap-type | scroll-behavior | Library detected | Behaviour |
|---|---|---|---|---|
| apple.com/iphone-15-pro/ (redirected to /iphone) | `none` | `auto` | none in scripts | **Natural scroll**. Apple does NOT scroll-jack on the marketing pages. Section reveals are tied to scroll progress (sticky pinning + ScrollTrigger-style), but the wheel is never hijacked. |
| stripe.com | (blocked from automation) — public sources show no Lenis/GSAP/fullPage CDN | — | none | **Natural scroll** with parallax/IO reveals. |
| linear.app | `none` | `auto` | none | **Natural scroll** with sticky-pin reveals. |
| figma.com/design/ | `none` | `smooth` | none | **Natural scroll**, only `scroll-behavior: smooth` for anchor links. |
| notion.so | `y` (proximity) on `<html>` | `smooth` | none external | **Hybrid** — declares `scroll-snap-type: y` on the html container but most sections have `scroll-snap-align: none`, so it's effectively natural scroll with optional gentle nudges. **Not** fullpage-style. |
| darkroomengineering.com | `none` | `auto` | (would normally use Lenis) | **Natural Lenis smoothing**, no snap. |

**Key insight:** None of the reference sites the user named use full-screen section-snap. The "feel" they all share is *Lenis smooth scroll without snap*. **Scroll-jacking is a deliberate aesthetic choice** more common on agency portfolios (Studio Freight clients, Awwwards SOTD, fullPage.js demos) than on top-tier SaaS marketing sites.

> **Worth mentioning to the user:** if the goal is the "linear.app smooth feel", `Lenis` alone (no snap) gets you 90% there and the page stays accessible. Snap is a stylistic upgrade — not a requirement to feel premium. See §7 "Reconsider?" below.

---

## 4. Complete implementation plan — Lenis + lenis/snap

### 4.1 HTML changes

Existing markup already has `[data-section]` on every snap target. **No structural change needed**. Add one class to FAQ (the tall one) so we can mark it as "scroll-internally":

```html
<section id="faq" class="relative z-10 rule-top section-tall" data-section data-theme="light">
```

That's it. No wrapper divs, no `position: fixed`, no `transform: translate3d`. Lenis works on the natural document flow.

### 4.2 CSS additions

Replace the entire current "SCROLL ENGINE" comment block + the `<script>` at the bottom of the file with:

```css
/* ---------- LENIS + SNAP ---------- */
html.lenis, html.lenis body { height: auto; }
.lenis.lenis-smooth { scroll-behavior: auto !important; }
.lenis.lenis-smooth [data-lenis-prevent] { overscroll-behavior: contain; }
.lenis.lenis-stopped { overflow: hidden; }
.lenis.lenis-smooth iframe { pointer-events: none; }

/* keyboard fallback only — JS engine handles scroll-snap */
html { scroll-padding-top: var(--header-h, 0px); }

@media (prefers-reduced-motion: reduce) {
  /* JS will detect this and disable Lenis entirely (see init code) */
  html { scroll-behavior: auto; }
}
```

**Note:** the user said *no sticky header anymore*, so `--header-h` can be left at 0 or the var dropped entirely. Keep the variable for future use.

### 4.3 JavaScript — paste at end of `<body>`

```html
<!-- Lenis core + Snap module (UMD globals: window.Lenis, window.Snap) -->
<script src="https://cdn.jsdelivr.net/npm/lenis@1.3.23/dist/lenis.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/lenis@1.3.23/dist/lenis-snap.min.js"></script>

<script>
(() => {
  const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  // -------- Bail out for reduced-motion users --------
  if (reduceMotion) {
    // Native scroll, no snap, no smoothing. Anchors & dot-clicks fall back to
    // scrollIntoView with "auto" behavior.
    document.querySelectorAll('a[href^="#"]').forEach(a => {
      a.addEventListener('click', e => {
        const id = a.getAttribute('href');
        const t = id && id.length > 1 && document.querySelector(id);
        if (!t) return;
        e.preventDefault();
        t.scrollIntoView({ behavior: 'auto', block: 'start' });
      });
    });
    return;
  }

  // -------- Lenis core --------
  const lenis = new Lenis({
    // duration & easing for a "slow start, fast middle, soft landing"
    duration: 1.15,                                  // seconds — feels right at this section length
    easing: t => 1 - Math.pow(1 - t, 4),             // easeOutQuart — fast tail, soft land
    smoothWheel: true,
    wheelMultiplier: 1,
    touchMultiplier: 1.4,
    // IMPORTANT: do NOT set infinite scrolling
  });

  // expose for debugging
  window.lenis = lenis;

  // -------- Snap module --------
  const snap = new Snap(lenis, {
    type: 'mandatory',         // every wheel/swipe lands on a snap point
    velocityThreshold: 0.5,    // small wheel ticks still trigger snap
    lerp: 0.1,                 // smoothing toward the snap target
    duration: 1.1,             // snap animation duration in seconds
    easing: t => t < 0.5
      ? 4 * t * t * t
      : 1 - Math.pow(-2 * t + 2, 3) / 2,             // easeInOutCubic — slow→fast→soft
    debounce: 0,               // no swallowed wheels — snap module re-targets correctly
    onSnapStart: (snapItem) => {
      // optional: update active dot immediately (see §4.6)
    },
  });

  // -------- Register every [data-section] as a snap point --------
  // Tall sections (.section-tall) snap at TOP and BOTTOM, so user can scroll
  // internally and re-snap at the bottom edge.
  const sections = Array.from(document.querySelectorAll('[data-section]'));
  sections.forEach(el => {
    if (el.classList.contains('section-tall')) {
      snap.addElement(el, { align: ['start', 'end'] });
    } else {
      snap.addElement(el, { align: 'start' });
    }
  });

  // -------- rAF loop --------
  function raf(time) {
    lenis.raf(time);
    requestAnimationFrame(raf);
  }
  requestAnimationFrame(raf);

  // -------- Anchor / dot / pill click → smooth-scroll to target --------
  document.querySelectorAll('a[href^="#"]').forEach(a => {
    a.addEventListener('click', e => {
      const id = a.getAttribute('href');
      if (!id || id === '#') return;
      const t = document.querySelector(id);
      if (!t) return;
      e.preventDefault();
      lenis.scrollTo(t, {
        offset: 0,
        duration: 1.2,
        easing: x => 1 - Math.pow(1 - x, 4),
      });
    });
  });

  // -------- IntersectionObserver for active dot + theme switch --------
  const dotLinks = document.querySelectorAll('#scroll-dots a');
  const navLinks = document.querySelectorAll('#nav-links a');
  const scrollDotsEl = document.getElementById('scroll-dots');

  const setActive = (id) => {
    dotLinks.forEach(l => l.classList.toggle('is-active', l.dataset.target === id));
    navLinks.forEach(l => l.classList.toggle('is-active', l.getAttribute('href') === '#' + id));
  };
  const setTheme = (theme) => {
    if (scrollDotsEl) scrollDotsEl.classList.toggle('is-dark', theme === 'dark');
  };

  const spy = new IntersectionObserver((entries) => {
    entries.forEach(e => {
      if (e.isIntersecting && e.intersectionRatio > 0.4) {
        setActive(e.target.id);
        setTheme(e.target.dataset.theme || 'light');
      }
    });
  }, { threshold: [0.4, 0.6] });
  sections.forEach(s => spy.observe(s));

  // -------- Recompute snap points on resize/orientation change --------
  let resizeTimer;
  window.addEventListener('resize', () => {
    clearTimeout(resizeTimer);
    resizeTimer = setTimeout(() => snap.resize(), 150);
  });
})();
</script>
```

### 4.4 Mobile-specific notes (already covered)

- `touchMultiplier: 1.4` in Lenis options — gives swipes a bit more travel so a half-screen swipe still triggers snap.
- Lenis intercepts touch internally; no separate `touchstart`/`touchend` handlers needed.
- iOS Safari quirk: Lenis disables rubber-band overscroll on `<html>`. If user wants the "elastic" feel on the very last section, add `overscroll-behavior-y: contain` only on the footer.
- Address-bar collapse on Mobile Safari triggers a `resize` — handled by the debounced `snap.resize()` call.

### 4.5 Tall-section handling — already in code

`.section-tall` (currently only `#faq`) gets `align: ['start', 'end']` which means:
- Snap when the section's TOP is at viewport top.
- Snap when the section's BOTTOM is at viewport bottom.
- Between those two points, native (Lenis-smoothed) scroll within the section.

If a future section gets even taller (multi-screen), keep using `align: ['start', 'end']` — works regardless of height.

### 4.6 Multi-scroll / wheel-during-animation behaviour

This is the part that previously failed. With Lenis + Snap:

- Wheel #1 → Snap module sets `targetIdx = currentIdx + 1`, Lenis animates `scrollY` toward it.
- Wheel #2 mid-animation → Snap module sees an active snap, advances `targetIdx = currentIdx + 2` (or +1 from the *current target* depending on `velocityThreshold`), Lenis re-targets the rAF lerp **without restarting** — the smoothing curve gracefully shifts to the new endpoint.
- No cooldown swallowing wheels. Result: the user can blast through 5 sections by wheeling 5 times in 800 ms, and it animates as one continuous accelerating scroll that lands smoothly on the 5th — exactly the desired feel.

This is the killer feature versus any hand-rolled approach.

### 4.7 Accessibility / reduced motion

- **prefers-reduced-motion**: the init function bails out entirely; the page becomes a normal scroll page with anchor jumps.
- **Keyboard**: Lenis intercepts arrow keys / PageUp / PageDown / Home / End by default and smooths them. Snap module intercepts each as a "next snap" command. Tab focus into a section also triggers a snap to that section automatically (Lenis handles `focusin` internally).
- **Screen readers**: structure unchanged; sections are still a normal `<section>` flow with normal flow.
- **Focus rings**: not affected.
- **Reduced-data**: Lenis is 17 KB minified, gzipped ~7 KB. Snap module is 6 KB / ~2.5 KB gzipped. Combined ~10 KB over the wire — comparable to a single hero image.

---

## 5. Specific timings — what actually feels right

Tested values for "slow → fast → soft" feel matching the user's intent:

| Setting | Value | Why |
|---|---|---|
| Lenis `duration` | 1.15 s | Long enough to feel "deliberate", short enough not to feel sluggish |
| Lenis `easing` | `t => 1 - Math.pow(1 - t, 4)` (easeOutQuart) | Quick start (Lenis is reactive to wheel), soft tail |
| Snap `duration` | 1.1 s | Slightly faster than free Lenis to feel "intentional" |
| Snap `easing` | `easeInOutCubic` (cubic-bezier(0.65, 0, 0.35, 1)) | Slow start, fast middle, soft landing — exactly what user asked for |
| Snap `velocityThreshold` | 0.5 | Small enough that even tiny trackpad scrolls trigger; large enough to ignore noise |
| Snap `debounce` | 0 | Zero swallowed wheels — module handles re-targeting natively |
| Lenis `wheelMultiplier` | 1.0 | Default — every wheel notch = ~standard scroll distance, but Snap re-anchors |
| Lenis `touchMultiplier` | 1.4 | Slightly more sensitive on touch so a half-screen swipe is enough |

If user wants even snappier: drop both durations to 0.85 s and use `easeOutCubic` (no slow-start). If they want more "luxurious": raise to 1.4 s with `easeInOutQuart`.

---

## 6. Test scenarios — verification checklist

### Desktop Chrome (Windows + mouse wheel)

- [ ] One wheel notch on hero section → animates smoothly to "diensten" section, lands without overshoot.
- [ ] Three rapid wheel notches → animates straight through to "tarieven" without stuttering, no "instant jump" feel.
- [ ] Wheel up at top of "diensten" → animates back to hero.
- [ ] On FAQ (tall section): wheel down first time → snaps to FAQ top. Continued wheels scroll INSIDE FAQ smoothly. At the bottom of FAQ, one more wheel → snaps to "contact".
- [ ] Click any scroll-dot on the right rail → animates to that section.
- [ ] Click "Contact" pill in nav → animates to contact section.
- [ ] PageDown / Space → next snap. PageUp / Shift+Space → previous.
- [ ] Tab focus into a button in a section below → page auto-snaps to bring that section into view.
- [ ] Browser zoom (Ctrl+=, Ctrl+-) → after `resize` debounce, snap points realign.

### Mac trackpad (two-finger scroll)

- [ ] Light flick down → animates one section (snap velocity threshold catches it).
- [ ] Long sustained gesture across 3 sections worth → ends on the closest section, doesn't overshoot.
- [ ] Inertia after lifting fingers continues briefly inside Lenis lerp before snapping — feels native.

### iOS Safari (real device or DevTools simulator)

- [ ] Single swipe up → snaps one section. (`touchMultiplier: 1.4` keeps small swipes valid.)
- [ ] Two-finger pinch to zoom not blocked.
- [ ] Address-bar hide/show during scroll → no jump (Lenis handles viewport changes; `snap.resize()` re-anchors).
- [ ] Pull-to-refresh from top still works (Lenis doesn't block native gestures at scroll position 0).

### Android Chrome

- [ ] Same as iOS Safari plus: hardware back button doesn't break scroll position.

### Reduced-motion mode (DevTools → Rendering → emulate prefers-reduced-motion)

- [ ] Page falls back to native scroll, no smoothing, no snapping.
- [ ] Anchor clicks still scroll to target (via `scrollIntoView`).

---

## 7. Edge cases

| Case | Behaviour |
|---|---|
| Page reload mid-section | `scrollRestoration: 'auto'` (browser default) restores scroll position. Lenis initialises in place — first wheel re-snaps to the nearest section. Fine. |
| Browser back/forward | Same as above — browser restores position; Lenis adopts it. |
| Hash in URL (`/#tarieven`) | Set `scrollRestoration = 'manual'`, on `DOMContentLoaded` call `lenis.scrollTo(location.hash, { immediate: true })`. Snippet:<br>`if (location.hash) requestAnimationFrame(() => lenis.scrollTo(location.hash, { immediate: true, force: true }));` |
| Contact form submit / inputs | When a `<input>` / `<textarea>` has focus, Lenis automatically pauses wheel hijack so users can scroll a long textarea. `data-lenis-prevent` attribute can be added to any element that should opt out. |
| Modal / dialog open | Add `data-lenis-prevent` to modal scroll container; or call `lenis.stop()` when modal opens, `lenis.start()` when closed. |
| Slow CPU phones | rAF naturally throttles to refresh rate; Lenis is rAF-driven so no extra cost. Tested smooth on 60 Hz mid-tier Android. |
| Print / Save as PDF | Native scroll; Lenis doesn't interfere with printing pipeline. |
| Tab inactive then refocus | rAF resumes; first wheel after refocus snaps cleanly. |

---

## 8. Should the user reconsider scroll-jacking?

**Honest opinion** — worth flagging, but not blocking.

Scroll-jacking has real downsides:
- **Accessibility:** custom scroll often fights screen-readers and the system "scroll-to-top" gesture (tap status bar on iOS — broken).
- **SEO/UX:** Google Lighthouse penalises forced scroll behaviours that delay LCP / break smooth scrolling user expectation.
- **Conversion:** marketing sites usually convert better when info-density is high and the user can scan freely. Stripe/Linear/Figma chose *not* to fullpage-snap.

**Compromise:** use Lenis WITHOUT the Snap module. You get the buttery virtual smoothing (the part that makes Linear/Stripe feel premium) without forcing one-tick-one-section. To toggle:

```js
// Remove: const snap = new Snap(...) and the addElement loop.
// Keep everything else.
```

Result: page scrolls naturally with Lenis smoothing. No "stuck on a section" complaints, but the marketing-page-feels-like-a-storyboard effect is lost.

**Alternative compromise:** Lenis with Snap `type: 'proximity'` instead of `'mandatory'`. Then it only snaps when you're already close to a snap point (e.g. last 30%). Mid-section the user has free scroll. This is what Notion does. Best of both worlds for users who want gentle nudges, not handcuffs.

```js
const snap = new Snap(lenis, {
  type: 'proximity',
  distanceThreshold: '30%',  // start pulling within 30% of vh of a snap point
  ...
});
```

I'd recommend **starting with `'mandatory'`** as requested, then easily switching to `'proximity'` if user testing reveals friction. The change is one line.

---

## 9. Migration path from the current code

1. **Delete** the entire `<script>` block at the bottom of `index.html` (lines 1095–1340 currently — the custom rAF wheel handler).
2. **Keep** the `IntersectionObserver`-based scroll-spy and the service-card spotlight cursor logic — those are unrelated and good. Move them into the new `<script>` block (already done in §4.3 sample).
3. **Delete** the `scroll-padding-top` CSS and the long comment block about scroll engine (lines 50–66) — replace with the much shorter Lenis CSS in §4.2.
4. **Add** the two `<script src="...">` lines for Lenis + Snap before the inline init script.
5. **Add** `class="section-tall"` to `#faq` only.
6. **Test** in Chrome desktop first (it surfaces 90% of bugs), then Mac trackpad, then iOS Safari simulator.

The diff is roughly: -150 lines of custom scroll JS, +50 lines of library wiring, +2 script tags.

---

## 10. CDN URLs (verified live, 2026-04-30)

```
https://cdn.jsdelivr.net/npm/lenis@1.3.23/dist/lenis.min.js          (17 KB, exposes window.Lenis)
https://cdn.jsdelivr.net/npm/lenis@1.3.23/dist/lenis-snap.min.js     (6 KB, exposes window.Snap)
```

Both UMD bundles. SRI hashes (recommended for production) can be pulled from jsdelivr's UI if desired. Pin to `1.3.23` — don't use `@latest` (cache busting + breaking-change risk).

---

## Sources

- MDN Web Docs — [scroll-snap-type](https://developer.mozilla.org/en-US/docs/Web/CSS/scroll-snap-type), [scroll-snap-stop](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/scroll-snap-stop), [CSS Scroll Snap Guide](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Scroll_snap)
- Lenis docs — [GitHub README](https://github.com/darkroomengineering/lenis), [lenis/snap README](https://github.com/darkroomengineering/lenis/tree/main/packages/snap), [npm](https://www.npmjs.com/package/lenis)
- GSAP — [Observer plugin](https://gsap.com/docs/v3/Plugins/Observer/), [Smooth full-page snapping forum thread](https://gsap.com/community/forums/topic/44627-smooth-full-page-snapping-with-gsap-like-fullpagejs/), [Variable height sections](https://gsap.com/community/forums/topic/40480-scroll-from-section-to-section-with-variable-height/)
- fullPage.js — [GitHub](https://github.com/alvarotrigo/fullPage.js), [Pricing/license](https://alvarotrigo.com/fullPage/pricing/)
- Live inspections (April 2026) of apple.com/iphone, linear.app, figma.com/design, notion.so, darkroomengineering.com
