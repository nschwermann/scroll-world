# Pipeline: copy-paste scripts (Replicate, bash 3.2 safe)

Every generation goes through the Replicate HTTP API with `curl` + `jq` — no CLI, one
token. Put the setup block and the functions you need into ONE `#!/bin/bash` script
and run it with `bash script.sh` (never paste array loops into zsh — 1-indexed arrays,
see gotchas.md). Generations take minutes: launch scripts **in the background** and poll
their log; never block the foreground.

## 0. Setup

```bash
WORK=/tmp/scroll-world           # scratch dir: prompts, sources, frames, run state
ASSETS=./assets                  # where the site reads stills (webp) + clips (mp4)
mkdir -p "$WORK" "$ASSETS/vid"
NAMES="farm kitchen shop delivery plaza finale"   # <-- your section ids, in order

: "${REPLICATE_API_TOKEN:?export REPLICATE_API_TOKEN first (SKILL Step 0)}"
API=https://api.replicate.com/v1

# Chain video model — ONE for every chained clip (SKILL Step 4 roster).
VMODEL=bytedance/seedance-2.0    # default; the moderation fallback is kwaivgi/kling-v3-video
VRES=1080p                       # 480p = previz, 720p = lean final, 1080p = standard
ASPECT=16:9                      # 9:16 for the mobile chain (§6)
DIVE_DUR=8; CONN_DUR=5           # seconds; arch-A legs use DIVE_DUR

# Stills: rendered straight at the VIDEO aspect, so the still IS frame 0 of the clip
# it starts (no 3:2 → 16:9 re-crop at the first seam). ONE source for every still.
SMODEL=openai/gpt-image-2        # or: grok  (Grok Imagine, §1b)
SSIZE=2048x1152                  # 1152x2048 for native portrait stills (Replicate only)

# DRAFT pass on Grok Imagine (§1b) — same scripts, different backend:
#   VMODEL=grok VRES=480p        (and SMODEL=grok for the stills, if they're Grok's too)
```

## 1. Replicate helpers

```bash
# Upload a local file → print an https URL usable as a model input (valid ~24 h).
# Cached next to the file so re-runs and re-rolls don't re-upload.
rep_upload() { # localPath
  local c="$1.url" u
  if [ -s "$c" ] && [ -n "$(find "$c" -mmin -1200 2>/dev/null)" ]; then cat "$c"; return 0; fi
  u=$(curl -fsS -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
        -F "content=@$1" "$API/files" | jq -r '.urls.get // empty')
  case "$u" in https://*) echo "$u" > "$c"; echo "$u" ;; *) echo "upload failed: $1" >&2; return 1 ;; esac
}

# Create a prediction on an official model and poll it to a terminal state (≤ 40 min).
# Writes the final prediction JSON to outJson; returns 0 only on "succeeded".
rep_run() { # model inputJsonFile outJson
  local id st k=0
  id=$(jq -c '{input: .}' "$2" | curl -fsS -X POST \
    -H "Authorization: Bearer $REPLICATE_API_TOKEN" -H "Content-Type: application/json" \
    -d @- "$API/models/$1/predictions" | jq -r '.id // empty')
  [ -z "$id" ] && { echo '{"status":"failed","error":"create failed (check input / token)"}' > "$3"; return 1; }
  while [ $k -lt 240 ]; do
    k=$((k+1)); sleep 10
    curl -fsS -H "Authorization: Bearer $REPLICATE_API_TOKEN" "$API/predictions/$id" > "$3" || continue
    st=$(jq -r '.status' "$3")
    case "$st" in succeeded) return 0 ;; failed|canceled|aborted) return 1 ;; esac
  done
  return 1
}

# Download a prediction's output (string or array). Output URLs expire after 1 HOUR.
rep_fetch() { # predJson outFile
  local u
  u=$(jq -r 'if (.output|type)=="array" then .output[0] else .output end // empty' "$1")
  [ -n "$u" ] && curl -fsSL "$u" -o "$2"
}

# Spend ledger: one line per billed gen; `spend` prints the running total.
rate() { # model res → USD per output second (Replicate list price — §8)
  case "$1:$2" in
    bytedance/seedance-2.0:480p) echo 0.08 ;;  bytedance/seedance-2.0:720p) echo 0.18 ;;
    bytedance/seedance-2.0:1080p) echo 0.45 ;; bytedance/seedance-2.0:4k)  echo 1.00 ;;
    bytedance/seedance-2.0-fast:480p) echo 0.07 ;; bytedance/seedance-2.0-fast:720p) echo 0.15 ;;
    kwaivgi/kling-v3-video:*) echo 0.224 ;;    # pro mode, audio off
    grok:*) echo 0 ;;                          # subscription session (§1b)
    *) echo 0.45 ;;
  esac
}
ledger() { echo "$1 $2" >> "$WORK/spend.log"; }    # label usd
spend()  { awk '{s+=$2} END {printf "spent ≈ $%.2f over %d gens\n", s, NR}' "$WORK/spend.log" 2>/dev/null; }

# Build the video input JSON for $VMODEL (field names differ per model).
vinput() { # promptFile startUrl endUrl|"" duration outJson
  case "$VMODEL" in
    kwaivgi/kling-v3-video)
      jq -n --rawfile p "$1" --arg s "$2" --arg e "$3" --arg a "$ASPECT" --argjson d "$4" \
        '{prompt:$p, start_image:$s, aspect_ratio:$a, duration:$d, mode:"pro",
          generate_audio:false} + (if $e=="" then {} else {end_image:$e} end)' > "$5" ;;
    *) # seedance-2.0 / -fast
      jq -n --rawfile p "$1" --arg s "$2" --arg e "$3" --arg a "$ASPECT" --arg r "$VRES" --argjson d "$4" \
        '{prompt:$p, image:$s, aspect_ratio:$a, resolution:$r, duration:$d,
          generate_audio:false} + (if $e=="" then {} else {last_frame_image:$e} end)' > "$5" ;;
  esac
}

# One video clip. Idempotent (skips if the output exists); logs spend on success.
gen_clip() { # label promptFile startPng endPng|"" duration outMp4
  [ -s "$6" ] && { echo "$1 cached"; return 0; }
  [ "$VMODEL" = grok ] && { grok_clip "$@"; return; }
  local s e=""
  s=$(rep_upload "$3") || { echo "$1 FAIL upload"; return 1; }
  if [ -n "$4" ]; then e=$(rep_upload "$4") || { echo "$1 FAIL upload"; return 1; }; fi
  vinput "$2" "$s" "$e" "$5" "$WORK/$1.in.json"
  if rep_run "$VMODEL" "$WORK/$1.in.json" "$WORK/$1.json" && rep_fetch "$WORK/$1.json" "$6"; then
    ledger "$1" "$(awk -v r="$(rate "$VMODEL" "$VRES")" -v d="$5" 'BEGIN{print r*d}')"
    echo "$1 ok ($VMODEL $VRES ${5}s)"
  else
    echo "$1 FAIL: $(jq -r '.error // .status' "$WORK/$1.json")"; return 1
  fi
}

status() {
  for n in $NAMES; do
    printf '%-10s still:%s dive:%s leg:%s\n' "$n" \
      "$([ -s "$WORK/still_$n.png" ] && echo ok || echo --)" \
      "$([ -s "$WORK/dive_$n.mp4" ] && echo ok || echo --)" \
      "$([ -s "$WORK/leg_$n.mp4" ] && echo ok || echo --)"
  done
  ls "$WORK"/conn_*.mp4 2>/dev/null; spend
}
```

## 1b. Grok Imagine — the free draft backend

xAI's Grok Imagine renders stills and image-to-video on the user's Grok subscription
session (`grok login` → `~/.grok/auth.json`), or pay-per-use with `XAI_API_KEY`. Use it
for the **draft pass**: the whole journey rendered at no per-clip cost, wired into the
real page, reviewed, then re-rendered on Replicate.

What it can and can't do for this skill:
- **Start frame: yes.** The input image becomes frame 1 — architecture A legs chain
  exactly as on Replicate.
- **End frame: no.** There is no last-frame input, so an architecture B connector can
  only *aim* at the next scene; its arrival seam will pop in the draft. Fine for judging
  motion and pacing; the SSIM gate will (correctly) flag those seams — ignore them in the
  draft.
- **Different model than the final.** A draft validates stills, journey, camera moves,
  copy, pacing and page wiring — not Seedance's seam character. Run the Replicate
  qualification probes (§7) before the final, and eyeball its first seams.

```bash
GROK_API=https://api.x.ai/v1
GROK_IMG_MODEL=grok-imagine-image-2.0
GROK_VID_MODEL=grok-imagine-video-1.5

grok_token() {
  if [ -n "${XAI_API_KEY:-}" ]; then echo "$XAI_API_KEY"
  else jq -r '[.[]|objects|select(.key)][0].key // empty' ~/.grok/auth.json 2>/dev/null; fi
}
grok_datauri() { # image → writes "<image>.duri" (JPEG data URI; inline is how xAI takes images)
  ffmpeg -v error -y -i "$1" -q:v 2 "$1.grok.jpg"
  { printf 'data:image/jpeg;base64,'; base64 -i "$1.grok.jpg" | tr -d '\n'; } > "$1.duri"
}
grok_post() { # path inJson outJson
  curl -sS -H "Authorization: Bearer $(grok_token)" -H "Content-Type: application/json" \
    -d @"$2" "$GROK_API$1" > "$3"
}

# Still (SMODEL=grok). With STYLE_LOCK set it's an edit seeded by the anchor.
grok_still() { # name
  local o="$WORK/still_$1.png"
  if [ -n "$STYLE_LOCK" ]; then
    grok_datauri "$STYLE_LOCK"
    jq -n --rawfile p "$WORK/still_$1.txt" --rawfile u "$STYLE_LOCK.duri" --arg m "$GROK_IMG_MODEL" --arg a "$ASPECT" \
      '{model:$m, prompt:$p, image:{url:$u, type:"image_url"}, aspect_ratio:$a,
        resolution:"2k", response_format:"b64_json"}' > "$WORK/still_$1.in.json"
    grok_post /images/edits "$WORK/still_$1.in.json" "$WORK/still_$1.json"
  else
    jq -n --rawfile p "$WORK/still_$1.txt" --arg m "$GROK_IMG_MODEL" --arg a "$ASPECT" \
      '{model:$m, prompt:$p, n:1, aspect_ratio:$a, resolution:"2k", quality:"medium",
        response_format:"b64_json"}' > "$WORK/still_$1.in.json"
    grok_post /images/generations "$WORK/still_$1.in.json" "$WORK/still_$1.json"
  fi
  jq -r '.data[0].b64_json // empty' "$WORK/still_$1.json" > "$WORK/still_$1.b64"
  if [ -s "$WORK/still_$1.b64" ] && base64 -d -i "$WORK/still_$1.b64" -o "$WORK/still_$1.jpg" \
     && ffmpeg -v error -y -i "$WORK/still_$1.jpg" "$o"; then
    ledger "still_$1" 0; echo "still $1 ok (grok)"
  else echo "still $1 FAIL (grok): $(jq -c '.error // .code // .' "$WORK/still_$1.json" | head -c 300)"; fi
}

# Clip (VMODEL=grok). Same signature as gen_clip; the end frame is ignored (no input for it).
grok_clip() { # label promptFile startPng endPng|"" duration outMp4
  local id st k=0
  [ -n "$4" ] && echo "$1: grok has no end-frame input — arrival seam is approximate (draft only)"
  grok_datauri "$3"
  jq -n --rawfile p "$2" --rawfile u "$3.duri" --arg m "$GROK_VID_MODEL" --arg a "$ASPECT" \
    --arg r "$VRES" --argjson d "$5" \
    '{model:$m, prompt:$p, image:{url:$u}, duration:$d, aspect_ratio:$a, resolution:$r}' \
    > "$WORK/$1.in.json"
  grok_post /videos/generations "$WORK/$1.in.json" "$WORK/$1.start.json"
  id=$(jq -r '.request_id // empty' "$WORK/$1.start.json")
  [ -z "$id" ] && { echo "$1 FAIL (grok): $(jq -c . "$WORK/$1.start.json" | head -c 300)"; return 1; }
  while [ $k -lt 120 ]; do
    k=$((k+1)); sleep 5
    curl -sS -H "Authorization: Bearer $(grok_token)" "$GROK_API/videos/$id" > "$WORK/$1.json" || continue
    st=$(jq -r '.status // empty' "$WORK/$1.json")
    case "$st" in done|failed|expired) break ;; esac
  done
  if [ "$st" = done ] && [ "$(jq -r '.video.respect_moderation' "$WORK/$1.json")" != false ] \
     && curl -fsSL "$(jq -r '.video.url' "$WORK/$1.json")" -o "$6"; then
    ledger "$1" 0; echo "$1 ok (grok $VRES ${5}s)"
  else
    echo "$1 FAIL (grok ${st:-timeout}): $(jq -c '.error // .video // {}' "$WORK/$1.json" | head -c 300)"; return 1
  fi
}
```

A 401 means the session expired: the user runs `grok login` (interactive — you can't),
or exports `XAI_API_KEY`. Don't try to refresh the session by running other `grok`
subcommands; a failed refresh can clear `auth.json`.

**Resume / idempotency.** `$WORK` *is* the run state: every generator skips an output
that already exists, so a crash, a failed clip, or a moderation re-roll never re-bills
finished work — just re-run the same loop. To force a re-roll, delete that file first
(`rm "$WORK/dive_shop.mp4"; gen_dive shop`). A replaced clip has new frames, so whatever
chains off it (later arch-A legs; the connectors on both sides of a dive) must be deleted
and re-rolled too.

## 2. Scene stills (Step 2) — anchor first, then batch

Write one prompt file per section to `$WORK/still_<name>.txt` (prompts.md).

```bash
STYLE_LOCK=""   # set to the approved anchor PNG after the gate
gen_still() { # name
  local o="$WORK/still_$1.png" ref="[]"
  [ -s "$o" ] && { echo "still $1 cached"; return 0; }
  [ "$SMODEL" = grok ] && { grok_still "$1"; return; }
  [ -n "$STYLE_LOCK" ] && ref=$(jq -n --arg u "$(rep_upload "$STYLE_LOCK")" '[$u]')
  jq -n --rawfile p "$WORK/still_$1.txt" --arg a "$SSIZE" --argjson r "$ref" \
    '{prompt:$p, aspect_ratio:$a, quality:"high", background:"opaque", output_format:"png",
      number_of_images:1} + (if ($r|length)>0 then {input_images:$r} else {} end)' \
    > "$WORK/still_$1.in.json"
  if rep_run "$SMODEL" "$WORK/still_$1.in.json" "$WORK/still_$1.json" && rep_fetch "$WORK/still_$1.json" "$o"; then
    ledger "still_$1" 0.13; echo "still $1 ok"
  else echo "still $1 FAIL: $(jq -r '.error // .status' "$WORK/still_$1.json")"; fi
}

# 1. Anchor + approval gate — the scene that best expresses the world:
gen_still farm
# → SHOW the user still_farm.png. Iterate the preamble until approved. A rejected
#   anchor: fix the preamble in ALL prompt files, rm the png, re-roll.

# 2. Batch the rest, style-locked to the approved anchor (skip the lock for photoreal
#    full-bleed scenes — it clones the room; prompts.md):
STYLE_LOCK="$WORK/still_farm.png"
for n in $NAMES; do gen_still "$n" & done; wait
```

Site copies (run `references/knockout.py` first if the scenes float):

```bash
for n in $NAMES; do cwebp -quiet -q 84 -resize 1800 0 "$WORK/still_$n.png" -o "$ASSETS/$n.webp"; done
```

## 3A. Architecture A — sequential legs (continuous forward take)

No connectors. Leg 0 starts on the first still; every later leg starts on the previous
leg's ACTUAL last frame. Strictly sequential — look at each last frame before going on.

```bash
gen_legs() {
  local prev="" n start
  for n in $NAMES; do
    if [ -z "$prev" ]; then start="$WORK/still_$n.png"; else start="$WORK/last_leg_$prev.png"; fi
    gen_clip "leg_$n" "$WORK/leg_$n.txt" "$start" "" "$DIVE_DUR" "$WORK/leg_$n.mp4" || return 1
    [ -s "$WORK/last_leg_$n.png" ] || \
      ffmpeg -v error -y -sseof -0.15 -i "$WORK/leg_$n.mp4" -frames:v 1 "$WORK/last_leg_$n.png"
    prev="$n"
  done
}
gen_legs
```

It stops at the first failure; fix it (re-roll / reword) and re-run — finished legs are
cached. A leg whose last frame isn't a calm forward glide: delete that leg, its
`last_leg_*.png`, and every leg after it, then re-run.

## 3B. Architecture B — dives (parallel)

```bash
gen_dive() { gen_clip "dive_$1" "$WORK/dive_$1.txt" "$WORK/still_$1.png" "" "$DIVE_DUR" "$WORK/dive_$1.mp4"; }
for n in $NAMES; do gen_dive "$n" & done; wait
```

## 4. Boundary frames + connectors (architecture B)

Connector start = dive_i's LAST frame, end = dive_{i+1}'s FIRST frame — extracted from
the **rendered videos**, never the stills.

```bash
for n in $NAMES; do
  ffmpeg -v error -y -ss 0       -i "$WORK/dive_$n.mp4" -frames:v 1 "$WORK/first_$n.png"
  ffmpeg -v error -y -sseof -0.15 -i "$WORK/dive_$n.mp4" -frames:v 1 "$WORK/last_$n.png"
done

gen_conn() { gen_clip "conn_$1" "$WORK/conn_$1.txt" "$2" "$3" "$CONN_DUR" "$WORK/conn_$1.mp4"; }
set -- $NAMES; i=0; prev=""
for n in "$@"; do
  if [ -n "$prev" ]; then i=$((i+1)); gen_conn "$i" "$WORK/last_$prev.png" "$WORK/first_$n.png" & fi
  prev="$n"
done; wait
```

Re-extract frames whenever a dive is re-rolled (`-y` overwrites; also delete the stale
`*.png.url` upload caches for those frames), and delete the connectors on both sides.

## 5. Encode for scrubbing (Step 6)

Native resolution (never upscale — encode what ffprobe reports), crf 20, GOP 8, light
sharpen, no audio, faststart.

```bash
enc() { ffmpeg -v error -y -i "$1" -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart "$2"; echo "enc $2 $(du -h "$2"|cut -f1)"; }

# arch B
for n in $NAMES; do enc "$WORK/dive_$n.mp4" "$ASSETS/vid/$n.mp4"; done
set -- $NAMES; i=0; while [ $i -lt $(($# - 1)) ]; do i=$((i+1)); enc "$WORK/conn_$i.mp4" "$ASSETS/vid/conn$i.mp4"; done
# arch A
# for n in $NAMES; do enc "$WORK/leg_$n.mp4" "$ASSETS/vid/$n.mp4"; done
```

`sections[k].clip = assets/vid/<name>.mp4`; `connectors = [assets/vid/conn1.mp4, …]`
(length N-1; `[]` for arch A). A connector you gave up on → leave its file absent and
put `null` in that slot; the engine crossfades that seam directly.

## 5b. Posters — the encoded clip's own first frame

The poster shows until the clip paints; if it isn't the clip's own frame 0, the first
thing a visitor sees pops. Same doctrine as the seams — hand off actual frames:

```bash
for n in $NAMES; do
  ffmpeg -v error -y -ss 0 -i "$ASSETS/vid/$n.mp4" -frames:v 1 "$WORK/poster_$n.png"
  cwebp -quiet -q 84 "$WORK/poster_$n.png" -o "$ASSETS/$n-poster.webp"
done
```

Wire `sections[k].poster`. Keep `still` — it's the stills-mode artwork.

## 5c. Seam gate — automated first, then eyeball

SSIM of each seam's boundary frames, measured at **composition scale** (320 wide +
blur), so Seedance's frame-0 re-render shimmer doesn't read as a mismatch — upstream saw
correct seams score only 18–25 dB PSNR at full resolution. A real mismatch is different
composition or props, and that survives the downscale.

```bash
seam_ssim() { # fileA fileB
  ffmpeg -v error -y -sseof -0.05 -i "$1" -frames:v 1 -vf "scale=320:-2,gblur=sigma=1.5" "$WORK/_sa.png"
  ffmpeg -v error -y -ss 0       -i "$2" -frames:v 1 -vf "scale=320:-2,gblur=sigma=1.5" "$WORK/_sb.png"
  ffmpeg -v info -i "$WORK/_sa.png" -i "$WORK/_sb.png" -lavfi ssim -f null - 2>&1 \
    | grep -o 'All:[0-9.]*' | cut -d: -f2
}
check() { # fileA fileB label
  local s; s=$(seam_ssim "$1" "$2")
  case $(awk -v s="${s:-0}" 'BEGIN{print (s>=0.90)?"pass":(s>=0.75)?"warn":"fail"}') in
    pass) echo "PASS  $3  ssim=$s" ;;
    warn) echo "WARN  $3  ssim=$s — eyeball it (the crossfade usually covers a near-miss end frame)" ;;
    *)    echo "FAIL  $3  ssim=$s — endpoints aren't the neighbours' frames, or the wrong frame was extracted" ;;
  esac
}
# arch B
set -- $NAMES; i=0; prev=""
for n in "$@"; do
  if [ -n "$prev" ]; then i=$((i+1))
    check "$ASSETS/vid/$prev.mp4" "$ASSETS/vid/conn$i.mp4" "$prev>conn$i"
    check "$ASSETS/vid/conn$i.mp4" "$ASSETS/vid/$n.mp4"   "conn$i>$n"
  fi; prev="$n"
done
# arch A: check consecutive "$ASSETS/vid/<name>.mp4" pairs the same way.
```

**Thresholds are provisional** — not yet calibrated on Replicate output. On the first
build, write each seam's score next to your eyeball verdict and adjust them here. WARN
means look, not re-roll. Re-run after every re-roll: one replaced clip touches both of
its seams.

## 6. Mobile — native 9:16 portrait chain (Step 1.6 opt-in)

A **parallel 9:16 chain**, frame-locked against its OWN renders — never a crop, never
mixed with landscape frames (a clip can't change aspect mid-chain). Same functions,
different settings and names:

1. **Portrait starts.**
   - Floating-island dioramas: composite each knocked-out still onto a 1080×1920 canvas
     in the page bg colour (island ~94% width, visual centre ~45% height) — the
     identical island, re-framed:
     ```bash
     pcanvas() { # cutoutPng bgHex outPng
       python3 -I -c 'import sys
     from PIL import Image
     src, bg, out = sys.argv[1:4]
     im = Image.open(src).convert("RGBA"); W, H = 1080, 1920
     w = int(W * .94); h = int(im.height * w / im.width); im = im.resize((w, h), Image.LANCZOS)
     c = Image.new("RGBA", (W, H), bg); c.alpha_composite(im, ((W - w) // 2, int(H * .45) - h // 2))
     c.convert("RGB").save(out)' "$1" "$2" "$3"; }
     for n in $NAMES; do pcanvas "$WORK/still_$n.cut.png" "#F5EDE0" "$WORK/pstill_$n.png"; done
     ```
   - Full-bleed / photoreal: re-render each still with `SSIZE=1152x2048`, the same prompt
     plus a vertical-recomposition clause, style-locked to its landscape still.
2. **Clips**: `ASPECT=9:16`, the portrait clause first in every prompt (prompts.md), and
   `_p` names throughout — e.g.
   `gen_clip "dive_${n}_p" "$WORK/dive_${n}_p.txt" "$WORK/pstill_$n.png" "" "$DIVE_DUR" "$WORK/dive_${n}_p.mp4"`.
   Extract boundary frames from the portrait renders and build portrait connectors
   between them.
3. **Encode** 720 wide, GOP 4 (a phone's seek cost scales with distance to the keyframe):
   ```bash
   encm() { ffmpeg -v error -y -i "$1" -an -vf "scale=720:-2,unsharp=5:5:0.6:5:5:0.0" \
     -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p \
     -g 4 -keyint_min 4 -sc_threshold 0 -movflags +faststart "$2"; }
   ```
   → `assets/vid/<name>-m.mp4`, `assets/vid/conn<i>-m.mp4`.
4. **Posters** from the `-m.mp4` first frames (§5b recipe) → `assets/<name>-poster-m.webp`.
5. **Seam gate** (§5c) on the portrait chain on its own.

Wire `clipMobile`, `posterMobile`, `connectorsMobile`. Budget ≈ the desktop chain's video
spend again.

**Crop stopgap — only with the user's explicit OK:** `encm` the 16:9 masters with
`scale=-2:720` instead. Phones then see the centre ~26% of the frame. Say so; it is not
the mobile version.

## 7. Qualification probes — before the first build, and whenever a model changes

Replicate model versions move. Two cheap 480p clips prove the chain will hold
(≈ $0.80 + two stills). Needs two real stills, `still_A.png` and `still_B.png`, and two
short prompt files.

```bash
VRES=480p
# (1) start-frame lock: a leg from a real still
gen_clip probe_leg "$WORK/probe_leg.txt" "$WORK/still_A.png" "" 5 "$WORK/probe_leg.mp4"
ffmpeg -v error -y -ss 0 -i "$WORK/probe_leg.mp4" -frames:v 1 "$WORK/probe_f0.png"
ffmpeg -v info -i "$WORK/probe_f0.png" -i "$WORK/still_A.png" \
  -lavfi "[1][0]scale2ref[a][b];[b][a]psnr" -f null - 2>&1 | grep -o 'average:[0-9.inf]*'
# (2) end-frame landing: a connector from still A to a DIFFERENT still B
gen_clip probe_conn "$WORK/probe_conn.txt" "$WORK/still_A.png" "$WORK/still_B.png" 5 "$WORK/probe_conn.mp4"
ffmpeg -v error -y -sseof -0.05 -i "$WORK/probe_conn.mp4" -frames:v 1 "$WORK/probe_fN.png"
ffmpeg -v info -i "$WORK/probe_fN.png" -i "$WORK/still_B.png" \
  -lavfi "[1][0]scale2ref[a][b];[b][a]psnr" -f null - 2>&1 | grep -o 'average:[0-9.inf]*'
ffprobe -v error -show_entries stream=codec_type,width,height,duration -of csv=p=0 "$WORK/probe_conn.mp4"
```

Pass: (1) frame 0 matches the still to codec noise (PSNR ≳ 30 dB) at the requested
aspect; (2) the last frame lands on B's composition (a Seedance-style near-miss is fine —
the engine crossfade covers it); no audio stream; duration as requested. Record the
numbers and the date in SKILL Step 4.

## 8. Notes

- **Prices** in `rate()` are Replicate list prices read 2026-11-18 from each model page:
  seedance-2.0 $0.08 / $0.18 / $0.45 / $1.00 per output second at 480p / 720p / 1080p /
  4k (image inputs; video inputs cost more); seedance-2.0-fast $0.07 / $0.15 (480p/720p
  only); kling-v3-video pro ≈ $0.224/s with audio off; gpt-image-2 high ≈ $0.13/image.
  Re-check before quoting a budget. The ledger is an estimate; the Replicate dashboard is
  the bill.
- **Moderation failures** come back as `status: failed` with an error string, which
  `gen_clip` prints. Re-roll, then reword, then the Kling fallback for that one clip
  (gotchas.md): `VMODEL=kwaivgi/kling-v3-video; rm "$WORK/conn_3.mp4"; gen_conn 3 …;
  VMODEL=bytedance/seedance-2.0`. Kling's native resolution may differ — encode what
  ffprobe reports.
- **Draft → final.** Draft on Grok (`VMODEL=grok VRES=480p`, §1b) when a Grok session
  exists; otherwise previz on the same Replicate model at `VRES=480p` (the only option
  that also previews seam character). After approval: `VMODEL=bytedance/seedance-2.0
  VRES=1080p`, delete the draft clips, frames and their caches
  (`rm -f "$WORK"/{dive,conn,leg}_*.mp4 "$WORK"/{first,last,last_leg}_*.png*`), re-run.
  Approved stills are reused whichever backend made them. Keep the draft's encoded page
  (copy `$ASSETS` aside) if you want a side-by-side. Don't previz on
  `seedance-2.0-fast` — barely cheaper, and a different model.
- Output URLs expire after **1 hour** and uploaded files after ~24 h — the helpers
  download immediately and re-upload when a cache is stale.
