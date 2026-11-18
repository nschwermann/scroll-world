---
name: scroll-world
description: >
  Build an immersive scroll-scrubbed "fly through the world" landing page for any
  industry or brand, generated on Replicate. As the visitor scrolls, a pre-rendered
  camera flies from outside each scene into its interior, then flows on to the next
  scene with NO cuts — one continuous connected flight (Emons-style isometric diorama
  world, or any art direction). Interviews the user for the topic, story beats, brand
  kit, camera style and budget; drafts the whole world free on Grok Imagine when a Grok
  session exists, then renders cohesive stills (gpt-image-2 or Grok) and frame-locked
  camera clips (Seedance 2.0) through the Replicate API; wires a portable,
  framework-agnostic scroll-scrub engine. Use when the user wants a "3D world" /
  "browse-through-the-industry" hero, a scroll cinematic, a diorama landing, or to turn
  a business into a scrollable world.
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion, Skill
---

# scroll-world

Produces a landing page where **scroll drives a camera**: it dives from outside a scene
into its interior, then flies on into the next scene, continuously, with no visible cuts.
The visuals are AI-generated on **Replicate** — stills with `openai/gpt-image-2`, the
camera chain with `bytedance/seedance-2.0` — and the page just scrubs pre-rendered video
by scroll position. Same technique as Apple's scroll-through product pages: the camera
genuinely moves, scroll only drives time.

**What you generate:** N scene stills → N camera clips → (architecture B) N-1
"connector" clips joining consecutive scenes → a portable scrub engine that plays the
chain as one flight.

**The one rule that makes or breaks it:** seams must be *frame-identical*. Every clip
that continues another starts on that clip's **actual rendered frame**, never on a fresh
still. Getting this wrong is the most common failure and produces a visible "pop".

**The other rule: nothing bills before the user says go.** The interview ends with a
dollar estimate and an explicit approval; the first still is an approval gate; a draft
pass (free on Grok Imagine when available, else a 480p previz) runs before the
full-resolution chain.

Don't assume a frontend framework. `references/scrub-engine.js` is self-contained
vanilla JS (it builds its own DOM and injects its own CSS into a container you give it),
so it drops into plain HTML, Next.js, Vue, a Python-served page, anything.

---

## Step 0 — Bootstrap

1. **Replicate token** in `REPLICATE_API_TOKEN`. If it's unset, ask the user where it
   lives (env file, secret manager) — never guess. Smoke test (free):
   `curl -fsS -H "Authorization: Bearer $REPLICATE_API_TOKEN" https://api.replicate.com/v1/account`.
2. **Roster check** (free) — model schemas move. Confirm the inputs the pipeline relies on:
   ```bash
   for m in bytedance/seedance-2.0 kwaivgi/kling-v3-video openai/gpt-image-2; do
     curl -fsS -H "Authorization: Bearer $REPLICATE_API_TOKEN" https://api.replicate.com/v1/models/$m \
     | jq -c --arg m "$m" '{m:$m, inputs:(.latest_version.openapi_schema.components.schemas.Input.properties|keys)}'
   done
   ```
   seedance-2.0 must list `image` + `last_frame_image`; kling-v3-video `start_image` +
   `end_image`. If a field moved, fix `vinput` in pipeline.md and re-run the
   qualification probes (pipeline §7) before spending on a chain.
3. **(Optional) Grok Imagine — the free draft backend.** If `~/.grok/auth.json` exists
   (the `grok` CLI's login session) or `XAI_API_KEY` is set, the draft pass and,
   optionally, the stills can render on xAI at no per-clip cost on the subscription
   (pipeline §1b). Check with one 1k still. A 401 means the session expired: ask the user
   to run `grok login` (interactive — you can't), and don't run other `grok` subcommands
   to force a refresh — a failed refresh deletes `auth.json`. Absent → drafts use the
   480p Replicate previz instead.
4. `ffmpeg` / `ffprobe`, `jq`, `cwebp` on `$PATH`. Python 3 + Pillow for the knockout
   and the mobile portrait canvases (optional otherwise).
5. Caveats: macOS ships **bash 3.2** (no `declare -A`), and zsh arrays are 1-indexed —
   keep chain loops in `#!/bin/bash` scripts. Generations take minutes; always run them
   **in the background** and poll.

---

## Step 1 — Interview the user

The **subject is the user's to state — ask it as an open question in plain prose**, never
a fabricated multiple-choice. Reserve `AskUserQuestion` for the genuinely enumerable
choices (art direction, camera style, mobile, budget tier), always with room to go their
own way. Ask only what you can't sensibly default. Cover:

1. **Subject** (open) — "What should this world be about? Your business, a client's, or
   any idea — a word or a sentence is fine." Capture industry/product, a one-line pitch,
   a brand name.
2. **Brand kit** — import from their site (read the page and its CSS for palette, name,
   tone — a logo or hero image helps), take what they hand you, or propose one for
   approval. Capture **4–6 named hex values**, a display name, a tone word or two.
3. **Art direction** — default "soft matte low-poly **clay diorama**, isometric,
   tilt-shift miniature, warm light." Alternatives in prompts.md (papercraft, glossy toy,
   claymation, neon night, photoreal architectural). The choice becomes the **style
   preamble**, reused verbatim in every still prompt.
4. **Camera style — always ask; it's the film's personality.** Record it as `CAMERA`:
   - **"Fly through the world"** — dives into each scene, pulls up and out, hops across
     the miniature world. → Architecture B. Default for diorama/miniature directions.
   - **"One continuous walkthrough"** — a single forward flight that glides through each
     scene into the next, never pulling back. → Architecture A. Default for
     grounded/photoreal directions.
   - **"Locked isometric glide"** — one fixed angle for the whole film, the world slides
     past. → Architecture A + the locked-iso clause (prompts.md).
   One line each on trade-offs: B reverses direction at seams (charming in miniature,
   jarring in realism) and costs the most; locked-iso is the calmest and cheapest to
   re-roll; walkthrough sits between.
5. **The journey** — ordered scenes derived from the subject's own value chain; propose,
   let them edit. 4–7. Each: a subject description (what's IN the scene), eyebrow,
   headline, one line of body, 0–3 tags. The last is usually the hero product + CTA.
6. **Mobile version — always ask; never silently generate both.** "Want a
   mobile-optimized version? It's a second camera chain rendered natively in 9:16
   portrait — composed for phones, not a crop — and roughly doubles the video spend
   (≈ $X)." Options: desktop only / desktop + native mobile. Desktop-only still gets the
   engine's phone hardening (it degrades gracefully). Never ship the centre-crop as the
   mobile version unless the user explicitly approves it as a stopgap.
7. **Budget — a dollar estimate, approved before anything renders.** Video is billed per
   output second; the estimate is `Σ clip seconds × rate + N × $0.13 stills`, plus
   **20–30% re-roll headroom** (more for interiors — content-filter re-rolls), doubled
   video for mobile. Rates (Replicate list prices, 2026-11-18 — re-check on the model
   page):

   | Tier (`VRES` on seedance-2.0) | $/s | N=6, arch B (73 s) | N=6, arch A (48 s) |
   |---|---|---|---|
   | Previz 480p | 0.08 | ≈ $6 | ≈ $4 |
   | Lean final 720p | 0.18 | ≈ $13 | ≈ $9 |
   | Standard 1080p (default) | 0.45 | ≈ $33 | ≈ $22 |
   | 4k | 1.00 | ≈ $73 | ≈ $48 |

   Arch B = N dives × 8 s + (N-1) connectors × 5 s; arch A = N legs × 8 s.

   **Draft pass — always offered, default for 5+ scenes.** Render the whole chain cheap,
   wire it into the real page, review, then render the final. Two ways:
   - **Grok Imagine draft ($0 on the subscription — default when a session exists).**
     Validates stills, journey, camera moves, copy, pacing and page wiring. It's a
     different model, so it doesn't preview Seedance's seam character, and it has no
     end-frame input: arch B connectors only *aim* at the next scene, so their arrival
     seams pop in the draft (expected; arch A drafts chain cleanly).
   - **Replicate 480p previz (≈ $4–6 for N=6).** Same model as the final, so it also
     previews seams. Use when there's no Grok session, or for arch B when the user wants
     to judge connector seams before paying for 1080p.

   **Stills source**: `gpt-image-2` on Replicate (≈ $0.13 each) or Grok Imagine ($0).
   One source for all stills of a build — mixing reads as style drift. Grok stills are
   fine finals; the video model doesn't care where its start image came from.

   Present the tier and draft choice, state the total (draft + final + headroom), and get
   a go. Fewer scenes → raise per-scene `scroll`/`linger` so a short world still reads
   complete.

If the user names a video model outside the roster, honor it **only if it can frame-lock
seams** (Step 4): it must accept a start image, and an end image for connectors. A model
that can't is declined with a one-line why, not substituted.

See `references/prompts.md` for the intake checklist and copy structure.

---

## Step 2 — Scene stills (anchor gate, then batch)

One image per section, all sharing the byte-identical style preamble, rendered at the
video's own aspect (16:9; 9:16 for the mobile chain) so each still is literally frame 0
of the clip it starts. Source per Step 1.7: `openai/gpt-image-2` (`quality: high`,
`2048x1152`) or Grok Imagine (`SMODEL=grok`, 2k).

1. **Anchor first.** Generate ONE still (the most representative scene) and show it to
   the user. Iterate the preamble until they approve the art direction. A style miss
   here costs one still, not N.
2. **Batch the rest style-locked** to the approved anchor (`STYLE_LOCK`, passed as an
   `input_images` reference) — except photoreal full-bleed scenes, where a reference
   clones the room; cohesion there comes from the preamble alone.
3. **Review for cohesion** (same angle, palette, light). Re-roll any off-style scene.

Functions and loops: pipeline.md §1–§2.

---

## Step 3 — (Optional) Float the scenes

To float dioramas over an atmospheric background instead of a solid box, knock the flat
background out with `references/knockout.py` (border-connected flood fill — keeps
interior colour that matches the bg). Otherwise make the page background the scene's
background colour. Keep the solid-bg PNGs — they're the video start images.

---

## Step 4 — Camera architecture + model (implements the Step 1.4 choice)

### Roster — one model for the whole chain

| Model (Replicate) | start / end image | Role |
|---|---|---|
| `bytedance/seedance-2.0` | `image` / `last_frame_image` | **Default** for everything. 480p–4k, 16:9 or 9:16, up to 15 s. `generate_audio` defaults **true** — always send `false`. |
| `kwaivgi/kling-v3-video` | `start_image` / `end_image` | Moderation fallback for a single stubborn clip (a different content filter). `mode: pro`, audio off. |
| `bytedance/seedance-2.0-fast` | `image` / `last_frame_image` | Faster turnaround only; 480p/720p, barely cheaper. Not a previz model — previz is seedance-2.0 at 480p. |
| Grok Imagine `grok-imagine-video-1.5` (xAI, `VMODEL=grok`) | `image` / — | **Draft pass only** (Step 1.7). Start frame locks; no end frame, so it can't make final arch-B connectors. 480p–1080p, 1–15 s. |

Rules:
- **One model for every chained clip.** Each renderer has its own motion/colour/grain;
  mixing models keeps position continuity but the character shift reads as a pop. The
  one sanctioned exception is the Kling fallback for one stubborn clip.
- A model qualifies only if it frame-locks: frame 0 reproduces the start image, and for
  connectors the last frame lands on the end image. **Qualify before the first build and
  whenever a model version changes** — two 480p probes, ≈ $1 (pipeline §7). Status:
  *not yet probed on Replicate* — record results and the date here.

### A) Continuous forward take — for grounded / realistic / walkthrough / locked-iso

One camera that only glides **forward**, first scene to last. Legs render
**sequentially**: leg 0 starts on scene 0's still; each later leg starts on the
**previous leg's actual last frame**, with **no end image** (an end image of a wide
establishing shot forces a pull-back — the #1 cause of stutter). No connectors; the legs
ARE the journey. Wire each leg as a section clip, `connectors: []`, `crossfade ≈ 0.08`.
Sequential = slower, and interiors trip the content filter — budget re-rolls.
Script: pipeline §3A.

### B) Dive-in + aerial connector — for diorama / miniature / god's-eye worlds

A dive into each scene (parallel, start image = the still) plus a connector that pulls
**up and out** and flies to the next scene (Step 5). The pull-out reverses camera
direction at every seam — reads as "zoom out to the map, fly to the next island" in
miniature, as a rewind stutter in a grounded walkthrough. If the user picked B for a
photoreal direction, explain that and confirm before rendering. Script: pipeline §3B.

### Camera grammar — the move should fit the concept

"Forward only" is the *seam* rule, not the *leg* rule:
- **Position continuity** at a seam comes from the frame handoff.
- **Velocity continuity** means the camera never *reverses across a seam*.
- **Inside one clip the camera is free** — orbits, crane-ups, lateral tracks, a push-in
  that eases back out are all safe mid-clip.

So each leg gets an expressive move under a **motion-handoff contract**: every leg
**ends by settling into a slow, steady forward drift** (final ~1 s) and **begins by
continuing that drift** (verbatim clauses in prompts.md).

| Concept / tone | Mid-leg move |
|---|---|
| Product / luxury retail | slow half-orbit around the hero object, then continue past it |
| Real estate / hospitality | steadicam glide through doorways; gentle crane-up in atria |
| Industrial / process / logistics | low lateral track alongside the line, foreground parallax |
| Travel / outdoors / campus | drone-style rise-and-reveal, then a descending swoop |
| Food / craft / detail-driven | push in close to the craft moment, ease back, carry on |
| Software / services / abstract | glide through a stylised "machine" of the product's workflow, one stage per scene |
| Playful miniature (arch B) | dives + aerial hops — the connector IS the grammar |

Expressive moves raise re-roll odds. Keep the settle clause verbatim and **eyeball each
leg's last frame before chaining the next** — it must look like a frame from a gentle
forward glide. A plain forward glide stays the zero-risk default.

**Locked-iso**: architecture A with the locked-iso clause in every leg instead of a
mid-leg move. Seedance drifts the angle slightly on long legs — re-roll a leg whose view
has turned.

Visitors scroll **up** too, so every move also plays in reverse — another reason seam
velocity must never flip.

---

## Step 5 — Connectors (architecture B only)

A connector flies from the end of scene i into the start of scene i+1. **Both endpoints
are actual rendered frames of the neighbouring dives — never the stills:**

```
start image = LAST frame extracted from dive_i
end image   = FIRST frame extracted from dive_{i+1}
```

Every render differs slightly, so a connector ending on a fresh render of "the kitchen"
won't match the kitchen dive's own first frame. Hand off the exact pixels. Seedance lands
*close* to the end image but not always pixel-perfect, so the engine adds a short
crossfade at each seam — insurance, never a substitute for the handoff. Script:
pipeline §4.

---

## Step 6 — Encode, posters, seam gate

1. **Encode** at native resolution, `crf 20`, GOP 8, light `unsharp`, no audio,
   faststart (pipeline §5). Don't downscale or go all-intra: seekability comes from the
   engine's blob loading, not keyframe density.
2. **Posters** = each encoded clip's own first frame (pipeline §5b), so the still→video
   swap on first paint can't pop.
3. **Seam gate** — SSIM every seam at composition scale (pipeline §5c) before opening a
   browser, and again after every re-roll. Thresholds are provisional until calibrated on
   a real build. On a Grok draft, connector→dive seams are expected to fail (no end
   frame); every other seam should still pass.
4. **Mobile (opt-in)** — the native 9:16 chain, encoded 720 wide at GOP 4, with its own
   posters and its own seam gate (pipeline §6).

---

## Step 7 — Assemble the page

Copy `references/scrub-engine.js` (and, for a standalone page,
`references/index-template.html`) into the project, or adapt it into the framework:

```js
mountScrollWorld(document.getElementById('world'), {
  brand: { name: 'Pearl & Co.' },
  diveScroll: 1.3, connScroll: 0.9,          // viewport-heights of scroll per clip
  sections: [
    { id:'farm', label:'The Farms', still:'assets/farm.webp',
      poster:'assets/farm-poster.webp',        // encoded clip's first frame (§5b)
      clip:'assets/vid/farm.mp4',
      clipMobile:'assets/vid/farm-m.mp4',      // mobile opt-in: native 9:16 render
      posterMobile:'assets/farm-poster-m.webp',
      scroll: 1.6, linger: 0.45,               // optional pacing
      accent:'#8FB98A', eyebrow:'From leaf to last sip', title:'It starts in the hills.',
      body:'…', tags:['Single-origin','Hand-picked'] },
    // …one per section; the last may carry a `cta`
  ],
  connectors:       ['assets/vid/conn1.mp4', /* … N-1, null = direct crossfade; [] for arch A */],
  connectorsMobile: ['assets/vid/conn1-m.mp4' /* same length; mobile opt-in only */],
});
```

The engine handles the chain, scroll→`currentTime` with rAF smoothing, blob loading
(fully seekable on any host), lazy prefetch, seam crossfades, pinned per-section copy,
a route rail, `prefers-reduced-motion`, and phones: phone-class screens (short side
≤ 600 CSS px) get the `-m.mp4` encodes while tablets get the master; seeks coalesce;
posters stay up until the clip paints; videos prime on first touch (iOS); iOS Low Power
Mode and data-saver fall back to cross-fading stills; mobile gets a longer scroll run
(`scrollMobileFactor`). Pacing: `scroll` lengthens a scene's dwell, `linger` (0–0.6)
settles the camera while the copy peaks — give hero and finale more. Theme with CSS
variables (`--sw-bg`, `--sw-ink`, `--sw-accent`, …; see the engine header).

**SEO:** the engine renders copy client-side. Put a plain-markup copy block marked
`data-sw-seo` inside the container (h1 + one h2/p per scene + real CTA links); the engine
hides it on mount. In a server-rendering framework (Next.js, etc.), render that block on
the server and mount the engine from a client component (`'use client'` + `useEffect`,
unmount-safe) — crawlers get real copy, visitors get the world.

---

## Step 8 — QA (don't skip)

Drive the page in a headless or real browser:

- **Seams**: screenshot just before and after each seam; frames must be near-identical.
  Judge by *composition*, not raw PSNR — a correctly locked seam can read 18–25 dB from
  detail shimmer; a real mismatch shows different composition or props.
- Console clean; `video.seekable.end(0) > 0` (blob working); `currentTime` tracks scroll.
- **First paint**: the first scene's poster and its clip's first frame are identical.
- **Desktop-only build**: sanity-check a phone viewport once — posters show, nothing
  overlaps.
- **Mobile build**: phone viewport with CPU throttled 4–6×, scroll fast — no freezing;
  the `-m.mp4` variants are served on phones and the master on desktop/iPad (Network
  panel); portrait clips are natively portrait (`videoWidth < videoHeight`); URL-bar
  collapse doesn't jump the page; rotation recomposes. Test iOS Safari specifically, and
  once with Low Power Mode on (should fall back to stills, not freeze).
- Reduced-motion → stills, no video, no particles.
- `spend` (pipeline §1) against the approved estimate — report the total to the user.

---

## Top gotchas (full list: `references/gotchas.md`)

- **Seam pop** → a connector or leg started/ended on a still instead of the neighbour's
  actual rendered frame. Re-extract and re-roll.
- **Seam stutter / camera jumps backward** → velocity reverses at a seam (architecture B
  in a grounded world). Use architecture A.
- **Paying for audio** → seedance-2.0's `generate_audio` defaults to true. Send `false`.
- **Content-filter false positives** on interiors (bed, pool, spa, wine, swim…) → re-roll,
  then reword ("empty, unoccupied, architectural, tasteful"), then that one clip on
  kling-v3-video, then a `null` connector (direct crossfade).
- **Frozen video at frame 0** → host doesn't serve byte ranges; the engine's blob loading
  fixes it — keep it if you port the engine.

## References

- `references/prompts.md` — intake checklist, style preambles, every prompt template.
- `references/pipeline.md` — Replicate helpers and copy-paste scripts: stills → legs or
  dives → connectors → encode → posters → seam gate → mobile chain → qualification
  probes; spend ledger; prices.
- `references/gotchas.md` — symptom → cause → fix, plus the canvas frame-sequence upgrade
  path.
- `references/scrub-engine.js` — the portable, config-driven scrub engine.
- `references/index-template.html` — a minimal standalone page that mounts it.
- `references/knockout.py` — border-connected background knockout for floating scenes.
