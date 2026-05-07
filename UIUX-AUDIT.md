# Mentink op maat — UI/UX Audit

Senior critique pass against `index.html` (1239 lines). Aesthetic baseline: refero-superhuman + refero-mercury. Calibration dials read as **DESIGN_VARIANCE = 4** (intentional asymmetry — italic-orange swashes, offset eyebrow column), **MOTION_INTENSITY = 5** (Lenis snap, scroll-reveal, float cards), **VISUAL_DENSITY = 3** (luxury, single focal point per page). The current build mostly honours the dials, but Pages 2 and 3 violate the "single focal point per viewport" rule because they overflow the snap viewport.

---

## Diagnostic — measured section heights vs. 100svh

Assuming a 1440 × 900 desktop viewport (typical macbook / 16:10), and reading the CSS literally:

| Section | Vertical padding (md) | Inner content estimate | Total ≈ | Viewport | Verdict |
|---|---|---|---|---|---|
| Hero | flex-center, padded | uses min-h:100svh + flex-justify-center | = 100svh | 900 | OK (centred by flex) |
| Diensten | `py-36` = 288px | heading 190 + mb-20 80 + 4 × `py-14` rows ≈ 4 × 220 = 880 | **1438** | 900 | **OVERFLOWS by ~540px** |
| Werkwijze | `py-32` = 256px | heading 240 + mb-16 64 + 3 step cards (max 330) | **890** | 900 | borderline; on a 1366×768 laptop it bleeds | 
| Tarieven | `py-32` = 256px | heading 240 + mb-20 80 + price cards 540 + perhour 200 + footnote 60 | **1376** | 900 | **OVERFLOWS by ~480px** (cards likely cropped) |
| FAQ | `py-32` + 6 details | tall by design — `.section-tall` with snap at start AND end | n/a | n/a | OK by design |
| Contact | `py-36` = 288px | heading 220 + body 80 + buttons 60 + side col | ≈ 740 | 900 | OK, slightly empty |
| Footer | `py-14` = 112px | 4-col + bottom row | ≈ 320 | n/a | OK as appendix |

The user's two complaints are confirmed:

1. **Diensten overflows by ~540px.** The four service rows + a 190px headline block simply do not fit in 900px. Snap-to-start parks the headline at the top, leaving rows 3 and 4 below the fold.
2. **Werkwijze overflows on common laptop heights** (768px / 800px), and any tarieven content that begins to render below it leaks into view because both sections use `min-height: auto` (no `100svh` floor) and snap aligns to start, not centre.

The hero is the only section with `min-height: 100svh` + `display:flex; justify-content:center`. Every other section is "as tall as content + padding," so when content is short the viewport has empty space below; when content is long the viewport clips it. Snap-align-start + variable heights = the exact bleed problem the user is reporting.

---

## Must-fix (red)

### 1. Diensten section overflows the viewport — content below the fold
**Where:** lines 658–745, `<section id="diensten" class="diensten-section">` and the `<div class="...py-24 md:py-36">` inner wrapper.

**Why it's wrong:** Four `py-14` (112px vertical) rows × 4 = 448px of padding alone, before content. Heading block adds ~270px including `mb-20`. Plus 288px of section padding. Total ≈ 1438px. A 900px viewport clips the bottom two services entirely.

**Fix — three layered changes (apply all three):**

```css
/* 1. lock the section to the viewport so snap centres it */
.diensten-section {
  min-height: 100svh;
  display: flex;
  align-items: center;        /* vertical centring */
  padding-top: 0; padding-bottom: 0;  /* override py-36 below */
}
```

Then in the HTML (line 663) **drop `py-24 md:py-36` to `py-12 md:py-16`**:
```html
<div class="relative max-w-[1400px] mx-auto px-6 md:px-10 py-12 md:py-16 w-full">
```

**2.** Tighten the heading-to-rows gap. Line 664: `mb-12 md:mb-20` → `mb-8 md:mb-12`.

**3.** Compress each service row's vertical padding from `py-10 md:py-14` → `py-6 md:py-8` (lines 680, 696, 712, 728). And shrink the service heading from `text-[30px] md:text-[40px]` → `text-[24px] md:text-[30px]`.

Net result: 4 rows go from ~880px to ~520px, heading block from 270 to 200, total ~720+padding ≈ 850px. Fits a 900px viewport.

**Skill ref:** per refero-mercury "spacious 80–120px section gap" applies between sections — *inside* a snap viewport it must respect 100svh first. Per taste/VISUAL_DENSITY=3, "Never more than one strong focal point per viewport" — currently you have a focal point AND clipped content fighting for the eye.

---

### 2. Werkwijze isn't vertically centred — tarieven bleeds in below
**Where:** lines 747–791, `<section id="werkwijze" class="bg-forest text-cream relative">`.

**Why it's wrong:** The section has no `min-height` and no flex-centring. It's exactly as tall as content + `py-20 md:py-32` (= 640 + content). On 1366×768 laptops the section is taller than the viewport; on 1920×1080 it's shorter, and because snap aligns to start, the user sees the green section glued to the top of the screen with cream tarieven padding bleeding in below.

**Fix:**
```css
/* add to the <style> block */
#werkwijze {
  min-height: 100svh;
  display: flex;
  align-items: center;
}
#werkwijze > div { width: 100%; }
```

Then trim padding (line 749): `py-20 md:py-32` → `py-12 md:py-20`. And reduce the giant `text-[88px]` italic numerals to `text-[64px] md:text-[72px]` (lines 766, 774, 782) — they currently push each step card to ~330px tall.

**Universal pattern — apply to all 100svh sections:** add a single utility class so you don't repeat the rule.

```css
.snap-viewport {
  min-height: 100svh;
  display: flex;
  align-items: center;
}
.snap-viewport > * { width: 100%; }
```

Then on the HTML side:
```html
<section id="werkwijze" class="snap-viewport bg-forest text-cream relative" data-section data-theme="dark">
<section id="tarieven"  class="snap-viewport relative z-10" data-section data-theme="light">
<section id="contact"   class="snap-viewport bg-ink text-cream relative overflow-hidden" data-section data-theme="dark">
```

The hero already does this (lines 287–292). Diensten gets the override above. Werkwijze, tarieven, contact all need it. FAQ is intentionally tall — leave it as `section-tall`.

---

### 3. Tarieven cards likely clip on common laptop heights
**Where:** lines 793–905. Three `.price-card` blocks at `p-8 md:p-10` + the `.perhour-strip` row + footnote.

**Why it's wrong:** Pricing cards alone are ~540px (header + €64px price + 5-item list + button + 80px padding). Add the heading block (240) and `perhour-strip` (200) and footnote (60) plus 256px of section padding → ≈ 1296px. Clips on 900px–1024px viewports.

**Fix:**
1. Apply `.snap-viewport` from above.
2. Remove the `perhour-strip` from this snap viewport — promote it to a **micro-row inline with the per-hour FAQ answer**, OR collapse it into a single-line CTA strip below the cards (one row, 80px tall).
3. Drop card padding to `p-6 md:p-8`.
4. Drop the price `text-[64px]` → `text-[52px] md:text-[60px]` and tighten `mt-8 mb-6` rules to `mt-6 mb-4`.
5. Section padding `py-20 md:py-32` → `py-12 md:py-20`.

If you want to keep the per-hour strip on this page, give it its own snap section (Page 4b) — but that fragments the journey. Better: kill it, surface "€40/uur" inside the Tarieven heading subtitle.

**Skill ref:** refero-superhuman "cards (large) 24px radius, 16px padding" — your `p-10` is double that. Pull it back.

---

## Should-fix (yellow)

### 4. Diensten overlay isn't strong enough at the top — heading reads as muddy cream-on-cinematic
**Where:** lines 310–316, `.diensten-overlay`.

The current overlay is `0.78 → 0.55 → 0.55 → 0.85` — middle is intentionally translucent so the photograph shows through, but the **top 30%** drops to 0.55 by the time the heading lands, and a Tokyo skyline duotone has a lot of competing high-contrast detail there.

**Fix:** make the top heavier and shorter:
```css
.diensten-overlay {
  background:
    linear-gradient(180deg, rgba(23,20,15,0.88) 0%, rgba(23,20,15,0.65) 22%, rgba(23,20,15,0.62) 78%, rgba(23,20,15,0.92) 100%),
    radial-gradient(120% 80% at 70% 30%, rgba(204,74,29,0.10), transparent 60%);
}
```
Plus add a `text-shadow: 0 1px 2px rgba(0,0,0,0.25)` to the heading on dark sections so high-frequency BG detail doesn't fight the type.

---

### 5. The `text-ink2` colour on Diensten descriptions is barely above mute
**Where:** line 319 — `.diensten-section .text-ink2 { color: rgba(251,249,242,0.85); }`. 0.85 cream on a 0.62-opacity dark BG that lets through Tokyo highlights gives a contrast that fluctuates. Service descriptions deserve 0.92 minimum.

**Fix:** bump to `0.95`. Per elite-frontend-ux, "minimum 4.5:1 contrast on dark, never lighter than #888" — your service descriptions need to clear that floor against the brightest visible background pixel.

---

### 6. The Diensten heading subtitle sits in a `flex items-end` column — alignment is a touch fragile
**Where:** line 672, `<div class="col-span-12 md:col-span-7 md:col-start-6 flex items-end">`. With the headline at 80px display and the body at 17–19px, the body's baseline lands roughly at the bottom of the H2 — fine on desktop but in shrinking viewports it will collide with the bottom of the headline before stacking.

**Fix:** add a `min-h-[var(--heading-height)]` or simpler: change `flex items-end` → `flex items-end pb-2`. Or move the body sentence into the eyebrow column to give the page a single hierarchical flow rather than competing left/right reads.

---

### 7. Two floating glass cards on hero overlap the photo aggressively
**Where:** lines 620–650.
- Top card: `top-6 -right-3 md:-right-8 w-[230px]`
- Bottom card: `-bottom-5 -left-3 md:-left-10 w-[260px]`

At a 420px-wide photo placeholder, the bottom-left card occupies roughly 45% of the photo's width and its top edge sits ~85px above the photo's bottom edge. Combined, the two cards cover ~25–30% of Job's portrait area. When the placeholder gets replaced with a real photo of Job, the cards will likely cover his shoulders/torso.

**Fix:** translate both cards further outboard so they touch but don't overlap:
- Top: `top-12 -right-6 md:-right-16 w-[220px]`
- Bottom: `-bottom-2 -left-6 md:-left-16 w-[240px]`

This preserves the "floating instrument" aesthetic of refero-superhuman without occluding the human.

**Skill ref:** refero-superhuman recipe — glass panels are *floating chrome*, not occluders. They sit on the periphery, gesture toward the human focus, never cover the face.

---

### 8. Hero headline is enormous (`text-[136px]` on lg) — risk of breaking on narrow desktop windows
**Where:** line 579, `text-[60px] sm:text-[84px] md:text-[112px] lg:text-[136px]`.

At 1280–1440 widths with the photo placeholder pinned right, the lg breakpoint kicks in and "Jouw boekhouding," at 136px / -0.045em tracking is ~860px wide. That fits — but only just. Add a `xl:` step or cap with `text-[clamp(60px,9vw,128px)]` so it stays in proportion.

**Fix:**
```html
<h1 class="display-huge reveal" style="font-size: clamp(60px, 9vw, 128px); animation-delay:.15s">
```
Then drop the responsive Tailwind sizing classes on that h1.

---

### 9. The header overlay is invisible on dark sections — and it never disappears
**Where:** lines 69–86, `.site-header { position: absolute; }`.

Because it's `absolute` (not `sticky` or `fixed`), the header **only exists on the hero section's coordinate space**. Once you snap to Diensten, the header is gone. Good for immersion — but the user might expect persistent nav. With snap-fullpage, a fixed header is the more conventional pattern, and the scroll-dot rail already gives navigation, so removing it is defensible. Document the intent so it doesn't look like a bug.

**Fix (option A — keep current immersion):** add a brief comment explaining intent. Nothing else.

**Fix (option B — persistent nav):** convert to `position: fixed`, add a transparent → cream-blur transition past hero:
```css
.site-header {
  position: fixed;
  background: transparent;
  transition: background .4s ease, backdrop-filter .4s ease;
}
.site-header.is-stuck {
  background: rgba(251,249,242,0.78);
  backdrop-filter: blur(14px) saturate(140%);
  border-bottom: 1px solid rgba(229,220,196,0.5);
}
```
And toggle `is-stuck` when the scrollY > hero height. On dark sections, swap text colour:
```css
.site-header.on-dark { color: var(--c-cream); }
```

I lean toward option B — the scroll-dots are great but a returning visitor expects to click the brand to go home from any section.

---

### 10. Scroll-dot pill on dark sections — colour passes, but the pill itself is barely visible
**Where:** lines 486–489.

`.scroll-dots.is-dark ul { background: rgba(23,20,15,0.50); border-color: rgba(255,255,255,0.15); }` — on the Tokyo cinematic, that 50% black on already-dark BG is almost invisible. The dots inside become the only signal.

**Fix:** raise the pill background opacity on dark contexts: `rgba(23,20,15,0.65)` and bump border to `rgba(255,255,255,0.25)`. Optionally add a 1px dark outer ring for separation:
```css
.scroll-dots.is-dark ul {
  background: rgba(23,20,15,0.65);
  border-color: rgba(255,255,255,0.25);
  box-shadow: 0 0 0 1px rgba(0,0,0,0.3), 0 8px 24px -8px rgba(0,0,0,0.5);
}
```

---

### 11. CTA section feels under-filled vs. its 100svh neighbours
**Where:** lines 989–1032.

After applying `.snap-viewport`, `#contact` will land vertically centred — but its content (heading + body + 2 buttons + side column) only occupies ~600px of vertical space. On a 1080px viewport that leaves 480px of empty ink-coloured surface, with the giant logo watermark trying to fill it.

**Fix options:**
1. Bump the headline to `text-[64px] md:text-[120px]` (it's the closing crescendo — make it bigger than the diensten heading).
2. Add a subtle horizontal rule of recent work / trust signals above the heading: "3 KvK-geregistreerde klanten · Sinds 2024" eyebrow row.
3. Move the right-column "Bereikbaar / Werkgebied" stack into a horizontal strip below the buttons so the section reads top-to-bottom and the empty bottom-half is filled.

I'd take option 1 + light option 2.

---

### 12. The `display-huge` headline weight (500) feels light against `font-semibold` (600) service titles inside diensten
**Where:** lines 127–132 and 683 / 699 / 715 / 731.

Hero h1 is weight 500. Service h3s are weight 600. The visual hierarchy actually inverts — your supporting headings look heavier than your hero. Per refero-superhuman, display headlines run 540–600 *with* sub-1.0 line-height; you have the line-height right but the weight too low.

**Fix:** raise `display-huge` font-weight to `600`. Or, keep 500 on hero and pull service h3s down to `font-medium` (500) for consistency. Pick one — I'd go heavier hero (600) since this is a marketing page that wants impact.

---

### 13. Eyebrow `№ 01 · Boekhouding` runs into the status pill on narrow screens
**Where:** lines 571–577.

`flex flex-wrap items-center gap-4` — at narrow widths the wrap puts them on two rows but gap-4 (16px) feels stingy when they wrap. Add `gap-y-2` so vertical wrap has more breathing room, or change to a column layout below 480px.

---

## Nice-to-have (green)

### 14. Italic-orange "swash" is the brand. Use it more sparingly to keep it surprising
Currently used on: hero (`op maat`), diensten (`één`), werkwijze (`drie`), tarieven (`Geen`), CTA (`heldere cijfers?`), and the per-hour strip (`inclusief`). That's 6 instances across 6 sections — predictability dilutes surprise. Drop one or two (the `inclusief` and `Geen` are weakest) so the device retains its "punctuation moment" quality.

### 15. The cinematic Tokyo BG is generic — consider a Dutch landscape
A bookkeeping startup in NL, even a premium one, with a Tokyo skyline reads as *aspirational stock asset* rather than *grounded craft*. A misty Dutch polder, an Amsterdam canal at dusk, or even an abstract organic gradient mesh would feel more authored. Per refero-superhuman, "cinematic dusk photography" is the recipe — but it should feel chosen, not licensed. Keep the duotone treatment, swap the source.

### 16. JetBrains Mono mono labels feel a bit "developer-portfolio" — consider a humanist mono
The eyebrow and price labels lean coder-aesthetic. For a bookkeeper targeting MKB owners, a softer humanist mono like Commit Mono, IBM Plex Mono, or Söhne Mono will land warmer. JetBrains Mono is fine, but consider if it matches the warm-cream + Inter Tight pairing — Söhne Mono Buch in particular rhymes with Inter Tight beautifully.

### 17. "Plan een gesprek" arrow icons are good — but inconsistent stroke widths
Some SVG arrows are `stroke-width="1.4"`, some `1.6`, some `1.3` (CTA phone icon line 1011). Lock the icon system to a single stroke (1.5 is the sweet spot for 14px icons).

### 18. Footer is 14px text on `text-cream/70` — borderline contrast
4.5:1 contrast ratio on cream/70 over ink (#17140F) computes to ~10.5:1, technically fine. But the 12px footer mono text at `text-cream/50` is below 4.5:1. Per WCAG 2.2 AA — bump to `cream/65` for the legal/copyright row.

### 19. Lenis duration 1.15 + snap duration 1.1 = the snap can outrace the wheel
Visually fine, but the snap module's `lerp: 0.1` plus `easing: easeInOutCubic` over 1.1s means a quick double-wheel within 1.1 seconds may chain destinations correctly (good — that's the fix you noted in the comment) but the user perceives ~2 full seconds of motion. Consider snap `duration: 0.85` for snappier feel — same easing, just shorter. Per elite-frontend-ux: "Spring `cubic-bezier(0.22, 1, 0.36, 1)` for reveals" — that's slightly punchier than easeInOutCubic.

### 20. Status pill ("3 plekken vrij · Q2") and announcement bar ("Beschikbaar voor 3 nieuwe klanten dit kwartaal") say the same thing
Two scarcity signals in the same viewport. Drop one — keep the announcement bar (more authoritative position) and remove the status pill from the eyebrow row, OR keep the status pill (more visual) and replace the announcement bar with a piece of differentiating copy ("Vaste maandprijs vanaf €50 — geen instapkosten").

### 21. Mobile responsiveness — the floating cards on hero are not hidden on small screens
Lines 620 and 635: cards have `md:` overrides for offsets but no `hidden md:block` — at 375px the photo + cards + headline all stack and the cards float in a way that is not designed for. Add `hidden md:block` to both glass cards, or render them below the photo as a stacked pair on mobile.

### 22. "Vier diensten, één aanspreekpunt" headline could be sharper
"Een" already implies one — the "één" italic is doing double work. Consider: "Alles bij één persoon." or "Vier diensten. Eén gezicht." (shorter, punchier, more confident). Per refero-superhuman copywriting: one declarative sentence beats one declarative phrase + clarifier.

---

## Synthesis — top 5 changes to ship now

1. **Add a `.snap-viewport` utility (`min-height: 100svh; display: flex; align-items: center`) and apply to werkwijze, tarieven, contact.** This single change fixes the user's "page 3 not centred / page 4 bleeding in" complaint across three sections at once. Hero already does this; diensten gets a custom variant in fix #2.

2. **Trim Diensten vertical metrics so 4 services + heading fit one viewport.** Section padding `py-36` → `py-16`, heading-to-rows gap `mb-20` → `mb-12`, row padding `py-14` → `py-8`, service h3s `text-[40px]` → `text-[30px]`. Apply `min-height: 100svh; display: flex; align-items: center` to `.diensten-section`. This is fix #1 above and resolves the user's complaint about overflow.

3. **Demote the Tarieven `perhour-strip` out of the snap viewport** — either inline it under the cards as a one-line CTA, or surface "€40/uur" as a Tarieven heading subtitle. Keeping three glass-cards as the only focal point lets the section feel premium and stay within 100svh per the taste/VISUAL_DENSITY=3 constraint.

4. **Reduce the giant `text-[88px]` italic numerals on werkwijze step cards to `text-[64px]–[72px]`.** Combined with the `.snap-viewport` change, this brings the section to ~720px and lets snap centre it cleanly on every common viewport (768px and up).

5. **Strengthen the Diensten dark overlay top-band from `0.78 → 0.88` and bump the heading-to-rows hierarchy.** Currently the headline reads muddy against the Tokyo duotone, and the service-row weight (600) is heavier than the section heading weight (500), inverting the hierarchy. Push display headlines to weight 600 and harden the overlay so the cinematic photograph supports — rather than competes with — the typography.

These five compose. Land them as one commit. After they ship, revisit the floating-card overlap on hero (#7), the header persistence decision (#9), and consider the Tokyo → Dutch landscape swap (#15) as a follow-up.
