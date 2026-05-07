# Mentink op maat — Visual Polish Review

**Reviewer lens:** senior visual designer + taste critic. Goal: lift the page from "well-built marketing demo" to "memorable premium boekhouder brand."

**Calibration read:** the page sits at roughly DESIGN_VARIANCE=3 / MOTION_INTENSITY=4 / VISUAL_DENSITY=3. It is tasteful, quiet, well-engineered, but slightly generic-Superhuman — a notch more *brand-specific eccentricity* would make it unforgettable. The core risk: it could be any premium freelancer site (designer, copywriter, consultant). It needs Job-shaped fingerprints.

---

## Big-bet ideas — bold visual moves

### B1. Make "op maat" the brand's *interactive* signature, not a static italic
**The change.** "Op maat" is the brand's whole positioning. Right now it appears as a static orange italic in three places (hero, tarieven, cta). Promote it to a recurring *moment*: every time the phrase appears, animate a hand-drawn underline swash that draws in on scroll-into-view (SVG `stroke-dashoffset` reveal, ~700ms, easeOutQuart). Also add it as a tiny ornamental glyph in the eyebrow lock-up (e.g. `№ 02 · op maat / Wat ik doe`). Carry the swash into the favicon, footer mark, and email signature so the underline is the brand mark.
- **Skill ref.** `taste` (variance dial), `gsap` ScrollTrigger for the draw-on-reveal, `algorithmic-art` for the SVG swash construction.
- **Effort.** medium (one SVG asset + one IntersectionObserver wrapper class + 4 substitutions).
- **Why it matters.** Right now the orange italic is *decoration*; this turns it into the brand's verb. Every section gets the same micro-payoff, which builds recognition by repetition — the cheapest way for a one-page site to feel like a brand system.

### B2. Replace the stylised Job placeholder with a real photographic moment
**The change.** The 4:5 SVG wireframe Job-figure reads "demo, not yet shipped" louder than anything else on the page. The hero's two glass cards are doing all the storytelling work; the photo slot is dead pixels. Two options, in priority:
1. **Photographic.** Commission/shoot a single warm-window-light portrait of Job at a desk, paper coffee, a notebook with handwritten BTW figures, lens at 50mm f/1.8. Apply a *very subtle* split-tone (warm cream highlights, deep forest shadows) so it lives inside the existing palette.
2. **Until that exists.** Replace the wireframe with a *typographic portrait* — a duotone full-bleed of just Job's signature ("Job Mentink") rendered in a deep ink ink-stroke at 600pt with the photo-slot's cream gradient behind it, then the floating cards on top. Better-looking than a stick figure, and intentionally signature-as-brand.
- **Skill ref.** `frontend-design-pro` (production polish), `refero-superhuman` (the cinematic-portrait moment).
- **Effort.** low (typographic fallback) → medium (real shoot).
- **Why it matters.** The hero is the entire pitch. A wireframe of a person undermines the credibility of every premium signal elsewhere on the page.

### B3. Cinematic transition Hero → Diensten — earn the duotone Tokyo cut
**The change.** Going from cream parchment to a dark Tokyo skyline is a *bold* aesthetic move that currently happens on a hard cut at the section boundary. It feels like two sites rather than one cinematic story. Add a 120vh transition belt:
- last 20vh of hero: cream darkens via a `linear-gradient(180deg, #FBF9F2 0%, #1a1612 100%)` overlay tied to scroll progress (use Lenis's `progress` event, no extra lib needed)
- diensten background scales from 1.08 → 1.0 with `transform` over the same scroll range (poor-man's parallax)
- a single faint horizon-line in orange (1px, opacity 0.25) drawn across the seam — the only chromatic element bridging the two worlds
- **Skill ref.** `gsap` ScrollTrigger / Lenis `on('scroll')`, `refero-monopo-saigon` (atmospheric gradient transitions).
- **Effort.** medium.
- **Why it matters.** The current cut feels like switching templates. A bridging belt makes it feel like *the camera moved* — which is the difference between "two sections" and "one cinematic experience."

### B4. A single editorial/typographic break — the bold move the page is missing
**The change.** Between Werkwijze and Tarieven, add a full-bleed cream section with one oversized italic phrase set in Inter Tight at ~22vw, e.g. *"Geen pakje formulieren. Eén persoon."* (or a Job quote). No image, no card, no button. Just one giant sentence, typeset like an editorial pull-quote, with a thin orange rule above and the eyebrow `№ — interlude` in JetBrains Mono. Acts as a palate cleanser between the dense Werkwijze and the dense Tarieven.
- **Skill ref.** `refero-monopo-saigon` (typographic moments), `refero-style-4` Hyperstudio (compressed line-height drama), `taste` variance dial.
- **Effort.** low.
- **Why it matters.** Every "premium" landing page has *one moment that breaks the grid*. This is a one-paragraph-of-CSS change that makes the page memorable. Right now every section has the same rhythm (left eyebrow + headline, right paragraph). The brain stops noticing. This breaks the pattern once.

### B5. Make one of the two hero glass cards the *protagonist*
**The change.** Both glass cards (BTW + Factuur) are equal-weight 230–260px decoration. Pick one to be the hero subject: enlarge to ~360–400px, tilt-on-mousemove (3deg max, soft spring), add a live-feeling micro-detail — a 4-bar BTW timeline with the current quarter highlighted in orange and a `<time>` showing "ingediend 2 dagen voor deadline." Demote the other card to a smaller satellite, half-overlapping behind it. Composition becomes 1 hero + 1 supporting, which is a stronger story than 2 equally-sized siblings.
- **Skill ref.** `aceternity-ui` (3d-card / tilt patterns), `elite-frontend-ux` (depth via z-axis layering).
- **Effort.** medium.
- **Why it matters.** The two cards currently compete for attention and lose to the headline; a clear hero card creates a focal hierarchy and gives the eye a place to land after the headline.

---

## Polish wins — high impact, low effort

### P1. Tighten the headline weight contrast
The `display-huge` is `font-weight: 500`. The italic-swash is also `500`. The italic doesn't stand out as much as the colour shift would suggest because the weights match. Drop the italic to `400` and the regular to `500` — the italic will feel lighter and more elegant (and matches the Mercury "wisp" aesthetic). Effort: low. Two CSS lines.

### P2. The hero status pill is wonderful — replicate the energy elsewhere
The pulsing-dot status pill ("3 plekken vrij · Q2") is the most *alive* element on the page. Right now it appears once. Add one more, smaller: `Reactietijd · &lt; 1 werkdag` next to the phone number in the header, same green dot. Two `status-pill` instances make it feel like a system; one feels like a one-off flourish. Effort: low.

### P3. The 01/02/03 italic-orange numerals are doing too much
You have `01 02 03` as huge italic 88px in Werkwijze, then `01 02 03 04` as small mono `text-mute` at 13px in Diensten, then no numbers in Tarieven. Pick one expression of "section ordinality" and run it everywhere. Recommend: keep the huge italic numerals only in Werkwijze (it's the section's signature), and *remove* the mono numerals in Diensten — they're competing with the huge type and adding visual debt. Effort: low.

### P4. The "Populair" badge needs a notch more presence
It reads as quiet. Increase to font-mono 12px, add 6px more horizontal padding, and put a 1px outset border in `cream/15` so it lifts off the dark featured card. Optionally: replace the word "Populair" with `★ Aanrader` — slightly less generic, slightly more Job-voice. Effort: low.

### P5. FAQ open transition is instant — add a content reveal
`<details>` opens with no transition by default. Wrap the answer paragraph in a CSS-grid `1fr` → `0fr` template-rows trick so it animates height smoothly:
```css
details > p { display: grid; grid-template-rows: 0fr; transition: grid-template-rows .35s cubic-bezier(.22,1,.36,1); overflow: hidden; }
details[open] > p { grid-template-rows: 1fr; }
```
Better: replace `<details>` with a controlled `<button aria-expanded>` + `<div hidden>` pattern so the easing is consistent. Effort: low.

### P6. The CTA "m" decoration looks like a mistake
The opacity-0.07 logo at top-right of the dark CTA reads as "leftover layer," not intentional. Two fixes, pick one:
- **Subtle:** tighten opacity to `0.04`, scale to 110vh tall, anchor it bottom-right to bleed off-page so only one curve of the m is visible. Reads as architecture, not artwork.
- **Bold:** make it a giant italic "M" instead of the logo (set in Inter Tight italic at 90vh, opacity 0.06, color `#E9A76A`) — echoes the "op maat" signature italic and ties the whole italic-orange motif together.
Effort: low.

### P7. Footer is missing the trust scaffolding a boekhouder needs
A boekhouder lives or dies on credibility signals. Add:
- A small KvK + BTW row (you already have `[TBD]` placeholders)
- One certification/affiliation logo if applicable (NOAB, NBA, Register Belastingadviseurs, Moneybird Partner) — even one tiny logo would do enormous heavy lifting
- A discreet `Privacy & AVG` link (Dutch users expect it)
- LinkedIn icon (Job, single channel — don't fake-stack five empty social icons)
Effort: low.

### P8. The scroll-dot pill — add one micro-affordance
The dots are tasteful but slightly invisible. On hover of *any* dot, expand the active dot's label inline (the labels are already in DOM as `sr-only`). 250ms ease, max-width 0 → 80px, padding shifts. Costs nothing in DOM but turns the dots from "abstract markers" into "named anchors" right when the user wants info. Effort: low.

### P9. Eyebrow numbering — go all the way or drop it
You use `№ 01`, `№ 02`, `№ 03`, `№ 04`, `№ 05` — but Hero is `№ 01 · Boekhouding` (descriptive) while Tarieven is `№ 04 · Tarieven` (literal section name). Pick a register: either *all* literal section names (cleanest, most editorial), or *all* descriptive labels (more voicey). Right now it's mixed and the eye notices. Effort: low.

### P10. Header on hero feels demo-y because the logo is small
At 36px the logo-mark + wordmark sits at roughly the same weight as the nav-pills and the phone number. A premium service-business header treats the logo as *the* anchor. Bump the logo-mark to 44px and increase the wordmark to 22px. Or: drop the wordmark entirely on hero (just the mark + a single-letter "M") and reveal the wordmark only on scrolled state. Effort: low.

---

## Taste calibration — micro-decisions

### T1. Body letter-spacing
Body paragraphs use Inter at default tracking. At 17–19px, Inter wants `letter-spacing: -0.005em` for a slightly more confident feel without being tight. Your headlines are at `-0.045em` (great) but body could lose its "neutral SaaS" texture with that one-line tweak.

### T2. Orange is too saturated in one place
`#CC4A1D` is beautiful as the swash and CTA hover, but on the `Populair` badge background it's slightly hot. Drop badge background to `#A83A15` (your already-defined `orangeDeep`) — cooler, more authoritative, less "alert-orange."

### T3. The italic-swash colour on dark sections is `#E9A76A` (peach)
This is a smart move — `#CC4A1D` would burn out on the cinematic dark backdrop. But the peach reads slightly different from the orange, like two accent colours. Consider keeping a single accent and using *opacity*: `#CC4A1D` at 100% on cream, `#CC4A1D` at 75% white-overlay (visually approximating peach) on dark. One accent feels more disciplined.

### T4. Easing
`cubic-bezier(.22,1,.36,1)` (easeOutQuart) is used everywhere — buttons, nav pills, glass-card hovers, scroll-reveal. It's good, but using *one* easing across the entire site flattens hierarchy. Reserve easeOutQuart for primary actions (button hovers, headline reveals) and use `cubic-bezier(.4,0,.2,1)` (Material-ish smoother) for ambient/decorative motion (gradient blobs, floating cards). Two easings = readable hierarchy.

### T5. The grain texture
`--grain-opacity: 0.03` is correct for cream sections but invisible on the dark Diensten/CTA sections (multiply blend on near-black does nothing). Add a screen-blend variant for dark sections at 0.04 — keeps texture continuity across themes. Three-line CSS change, large perceived-quality lift.

### T6. Headline line-height
`display-huge` uses `line-height: 0.94`. On the 112–136px hero this is right — confident, compressed. But the same class applied at 80px (Diensten "Vier diensten / één aanspreekpunt") looks slightly cramped. Consider a `display-huge--md` variant at `line-height: 0.98` for the secondary headlines.

### T7. Glass-card border
`border: 1px solid rgba(255,255,255,0.65)` is correct on cream — it sells the "frosted glass." But it's invisible to many displays in HDR/wide-gamut. Add a hairline inner shadow `inset 0 0 0 0.5px rgba(255,255,255,0.4)` to make the glass edge survive HDR rendering.

### T8. JetBrains Mono is doing more work than acknowledged
Mono appears in: eyebrow, status pill, captions, prices `/mnd`, footer. That's a lot. It's a brand signal — lean *into* it: use it in the announcement bar date too ("Mei 2026" is already mono — good), and consider one bigger mono moment, e.g. set `06 2821 5898` in mono *with* tabular-nums (`font-feature-settings: 'tnum'`) so the digits align beautifully. Mono-as-brand-voice is more distinctive than Inter Tight italic alone.

---

## Layout rhythm

### L1. Vertical pacing is uniform — alternate density
Every section uses ~`py-20 md:py-32`. Identical pacing creates a metronome effect. Suggested rhythm:
- Hero (full viewport, `min-height: 100svh`) — done
- Diensten (`py-36`) — heavy, cinematic, breath
- Werkwijze (`py-24`) — tighter, denser, "we move fast"
- *Editorial pull-quote interlude* (B4) (`py-40`) — extreme breathing
- Tarieven (`py-24`) — denser, decision-mode
- FAQ (`py-32`) — generous, reading mode
- CTA (`py-36`) — heavy closer

This alternation between dense decision sections and breath sections creates *narrative pacing*.

### L2. The diensten 4-row section is too uniform
Four service rows of identical height feel like a list; they should feel like a portfolio. Stagger: row 1 normal, row 2 indent right by `col-start-2`, row 3 normal, row 4 indent right. Tiny visual change, makes the section feel *composed* rather than *generated*. Reference: `refero-monopo-saigon` zigzag rhythms.

### L3. Tarieven needs a comparison anchor
Three cards side-by-side without a "from / to" anchor leaves the eye floating. Add a thin horizontal rule above all three cards labeled `Maandprijs · vanaf €50` that visually ties them. Or: add a single `← lichter / zwaarder →` axis label at the top in mono 11px. Costs nothing, gives the comparison structure.

### L4. FAQ is too tall on desktop
Six questions × ~120px each stacked = a long scroll. Consider a 2-column FAQ on desktop (3+3 split, masonry-flow) which doubles the scan-density. The "ask the right question" eyebrow + headline column on the left already wastes vertical space; let the right column be 2 columns wide on desktop.

### L5. The CTA's right column is starving
`md:col-span-3 md:col-start-10` for the bereikbaar/werkgebied block is a 25%-width afterthought. Either give it real status — turn each into a glass-card with an icon (calendar, map-pin) — or fold it into the body of the CTA as a single inline mono line: `Online · NL  ·  Ma–Vr  ·  &lt; 1 werkdag reactie` set under the headline. The current sidebar feels like leftover wireframe.

---

## Animation / motion

### M1. Scroll-driven parallax on the cinematic Tokyo background
`background-attachment: fixed` was correctly avoided for mobile, but you can use Lenis's scroll progress to translate the `.diensten-bg` by `translateY(progress * -80px)` for a real parallax. Locks the cinematic feel without the mobile bug.
- **Skill ref.** `gsap` (ScrollTrigger), or hook directly into Lenis `on('scroll')`.
- **Effort.** low.

### M2. Glass cards on hero — magnetic hover + tilt
The two cards currently `floatA`/`floatB` autonomously. Add: on `mousemove` over the right column, both cards gently translate towards the cursor (max 8px) and tilt 2deg. Reads as physical, alive, premium. Disable on `prefers-reduced-motion` (already handled globally).
- **Skill ref.** `aceternity-ui` 3d-card patterns.
- **Effort.** medium.

### M3. Numerical count-up in the BTW glass card
The BTW card says "Aangifte ingediend." On scroll-into-view, animate a count-up `Q2 · 2026` → fade — or animate the green checkmark draw-in via `stroke-dashoffset` (the path is already SVG). 600ms, easeOutQuart. Tells the user what the card *means*: "this just happened, on time."
- **Skill ref.** `gsap` for the path-draw.
- **Effort.** low.

### M4. Service-card hover spotlight is good — extend to "pre-load"
When the user enters the Diensten section, fire a one-off ambient sweep across all 4 cards' spotlights (left → right, 800ms apart). Tells the user the rows are interactive. Currently the spotlight only appears on hover, which means many users never discover it. Subtle "rolling intro" makes the affordance visible without explanation.
- **Effort.** low.

### M5. The italic-swash word — animate on scroll-into-view
Currently the orange italic words just appear with the headline reveal. Make them more deliberate: keep them invisible for the first 200ms of the scroll-reveal, then fade + draw-underline in (see B1). Each italic word becomes a *moment*.
- **Skill ref.** `gsap` SplitText or just a CSS `@keyframes` with `animation-delay`.
- **Effort.** low.

### M6. Lenis snap is great — add a snap-progress micro-indicator
Right now the snap is invisible motion. Add a 1px orange progress line at the very top of the viewport (or thinner: 0.5px on hi-dpi) that fills section-by-section as snap fires. Telegraphs the "single-page deck" architecture and gives the snap a legible visual rhythm. Hide on touch devices (where snap is more disorienting).
- **Effort.** low.

### M7. CTA — magnetic primary button
The mailto button is the page's most important conversion. Give it a true magnetic hover: button translates 4–8px toward cursor when within 80px, springs back on leave. Current `translateY(-1px)` is too subtle for the closer.
- **Skill ref.** `aceternity-ui` magnetic-button.
- **Effort.** medium.

### M8. Reduced-motion respect — already great, one addition
Your `@media (prefers-reduced-motion: reduce)` block is thorough. One thing missing: the gradient blobs (which technically don't animate but suggest motion). Consider also reducing their `filter: blur()` on reduced-motion to flat radial fills — visually calmer for users with vestibular disorders.

---

## Synthesis — top 5 in priority order

The five changes most worth implementing, ranked by impact-per-effort and brand significance:

1. **Replace the SVG Job-figure placeholder (B2).** Single biggest credibility gap. Until a real photo exists, the typographic-portrait fallback is a 30-minute job and immediately reads "intentional," not "demo." Without this, every other improvement is polish on a TODO note.

2. **Promote "op maat" from static italic to animated brand signature (B1).** This is the cheapest way to give the site a memorable verb. The orange italic + draw-in swash, repeated in 3–4 places, is what people will remember. It's the "voice" of the visual system.

3. **Bridge the Hero → Diensten cut with a parallax/gradient transition belt (B3).** Currently the boldest aesthetic move on the page (cream → cinematic dark) lands as a hard cut. A 120vh transition belt makes it feel cinematic rather than templated, and it's pure CSS + Lenis hookup. Low effort, transformational difference.

4. **Add the editorial typographic interlude between Werkwijze and Tarieven (B4).** Every premium landing page has one moment that breaks the grid. The page is missing it. One full-bleed sentence in 22vw italic Inter Tight is the bold move that elevates the site from "tasteful" to "designed."

5. **Footer trust scaffolding + the Tarieven anchor (P7 + L3).** A boekhouder's website needs to project credibility above all. KvK/BTW/AVG/affiliation logos in the footer + a single horizontal anchor above the price cards convert "looks nice" to "I trust this person with my money." Both are 30-minute changes; both move conversion more than any animation.

Everything else is polish on top of those five. Ship them in order; each successive item starts looking better once the one before is in place.
