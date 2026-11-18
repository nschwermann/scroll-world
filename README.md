# scroll-world (Replicate edition)

An agent skill — for Claude Code, Codex, and any `SKILL.md`-compatible agent — that builds
an immersive, **scroll-scrubbed "fly through the world" landing page**: as you scroll, a
camera flies from *outside* each scene *into* it, then flows on to the next with **no
cuts**. One continuous flight through a small generated world.

This is a fork of [oso95/scroll-world](https://github.com/oso95/scroll-world) that runs
the whole generation pipeline on **[Replicate](https://replicate.com)** — one API token,
pay-per-use — instead of Higgsfield / Monid / Codex. It also folds in the engine and
pipeline hardening from [cth9191/scroll-world](https://github.com/cth9191/scroll-world).

## Install

### Claude Code — as a plugin

```
/plugin marketplace add nschwermann/scroll-world
/plugin install scroll-world@scroll-world
```

### Codex & other agents — via the skills CLI

```bash
npx skills add nschwermann/scroll-world
```

### Manually

```bash
git clone https://github.com/nschwermann/scroll-world
cp -R scroll-world/skills/scroll-world ~/.claude/skills/
```

## Requirements

- A Replicate API token in `REPLICATE_API_TOKEN`, with billing set up.
- Optional: a Grok session (`grok login` → `~/.grok/auth.json`) or `XAI_API_KEY`, for
  free drafts and stills on Grok Imagine.
- `ffmpeg` / `ffprobe`, `jq`, `cwebp`.
- Python 3 + Pillow (background knockout and mobile portrait canvases; optional).

## What it does

1. **Interviews you** — subject, brand kit, art direction, camera style, the ordered
   scenes, whether you want a native 9:16 mobile chain, and a **dollar budget** approved
   before anything renders.
2. **Generates the assets on Replicate** — stills with `openai/gpt-image-2` (one anchor
   approved first, the rest style-locked to it), camera clips with
   `bytedance/seedance-2.0` frame-locked at every seam (each clip starts on its
   neighbour's actual rendered frame), a 480p previz pass before the full-resolution
   chain, and `kwaivgi/kling-v3-video` as the fallback for content-filter false
   positives.
   Optionally, the whole world is **drafted first on Grok Imagine** (xAI) — free on a
   Grok subscription session — and stills can come from Grok too.
3. **Verifies and wires it up** — encodes for smooth scrubbing, extracts posters from the
   encoded clips, SSIM-gates every seam, and mounts a config-driven vanilla-JS scrub
   engine (works in plain HTML, Next.js, anything).

Rough cost for a 6-scene world (Replicate list prices, Nov 2026): ≈ $6 previz + ≈ $33 at
1080p for the dive-and-connector style, ≈ $22 for the continuous-walkthrough style, plus
re-roll headroom. The skill computes the estimate for your build and keeps a running
spend ledger.

## Differences from upstream

| Area | oso95 upstream | this fork |
|---|---|---|
| Backend | Higgsfield CLI + Monid (+ Codex for stills) | Replicate HTTP API, plus an optional free draft pass on Grok Imagine |
| Stills | 3:2, re-cropped by the video model | rendered at the video's 16:9 / 9:16 aspect |
| Spend | estimate + approval | + anchor-still gate, previz by default, idempotent resumable pipeline, spend ledger |
| Seam QA | eyeball | + automated composition-scale SSIM gate |
| Posters | source still (pops on first paint) | encoded clip's own first frame |
| Engine | — | + iOS Low Power Mode / data-saver stills fallback, phone-vs-tablet clip tiering, longer mobile scroll, crawlable `data-sw-seo` copy block (from cth9191) |
| Arch A | described | + a sequential leg script |

## What's in the skill

```
skills/scroll-world/
├── SKILL.md                 the procedure, the seam rule, top gotchas
└── references/
    ├── prompts.md           intake checklist + every prompt template
    ├── pipeline.md          Replicate helpers + copy-paste batch scripts
    ├── gotchas.md           symptom → cause → fix
    ├── scrub-engine.js      portable scrub engine
    ├── index-template.html  minimal standalone page
    └── knockout.py          background knockout for floating scenes
```

## License

MIT — see [LICENSE](LICENSE). Original work © cyw (oso95); hardening by cth9191.
