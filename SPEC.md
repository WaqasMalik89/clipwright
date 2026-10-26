# Clipwright: Agentic Social Video Generator

## What this project is
A CLI + local pipeline that turns a raw video into ready-to-post Instagram content
(Reels, feed posts, carousel stills, captions) with branded, animated overlays.
An LLM "creative director" plans the edit as structured JSON; deterministic code
renders it; an LLM "critic" reviews rendered previews and revises the plan.

**In scope:** analysis, clip selection, reframing, animated captions/titles, copywriting, variants, rendering.
**Out of scope:** posting/scheduling to social platforms (user uploads manually).

## Core design principles (follow these strictly)
1. **The LLM chooses, templates execute.** The agent never writes animation code at runtime.
   It selects templates and fills parameters in a validated JSON edit plan.
2. **Every LLM output is schema-validated** (Zod in TS, Pydantic in Python). On validation
   failure, retry once with the error message appended; then fail loudly.
3. **Deterministic stages are cached** by content hash (input file hash + stage params) in `.cache/`.
   Re-running the pipeline must not re-transcribe or re-analyze unchanged inputs.
4. **Quality comes from the critic loop.** Never skip it except with `--fast`.
5. Keep stages independently runnable for debugging (`clipwright analyze`, `plan`, `render`, `critique`).

## Repo layout
```
clipwright/
  SPEC.md
  package.json              # pnpm workspace root
  apps/
    cli/                    # Node CLI entry (commander), orchestrates stages
  packages/
    agent/                  # Anthropic SDK calls: director, critic, copywriter
    schemas/                # Zod schemas shared by agent + render (single source of truth)
    render/                 # Remotion project: compositions + templates
  services/
    analysis/               # Python: transcription, scenes, audio, reframe tracks
  brand/
    example.brand.json
  fixtures/                 # short test videos (<30s) for tests
  .cache/                   # stage outputs (gitignored)
  out/                      # final deliverables (gitignored)
```

## Tech stack
- **Node 20+, TypeScript strict, pnpm workspaces**
- **Anthropic TypeScript SDK** (`@anthropic-ai/sdk`). Model from env `CLIPWRIGHT_MODEL`,
  default `claude-sonnet-5-5`. Use `claude-opus-5-5` for the director when `--quality max`.
  Use tool use / JSON output with schema-validated parsing.
- **Remotion** for rendering (React compositions, props = edit plan). Note: Remotion requires a
  company license above a certain team size — check its license before commercial use.
- **Python 3.11 service** (`uv` for deps): `whisperx` (word-level timestamps), `scenedetect`,
  `librosa` (audio energy/onsets), `mediapipe` (face/pose tracking), `opencv-python`.
- **FFmpeg** for probing, frame extraction, proxy creation, final muxing.
- Node ↔ Python: Node spawns `uv run python -m analysis <cmd> --in X --out Y.json`. No HTTP server needed for v1.

## Pipeline
```
input.mp4
  └─▶ [1] ingest        → proxy.mp4 (720p), probe.json
  └─▶ [2] analyze       → analysis.json   (Python)
  └─▶ [3] plan          → plan.v1.json    (Director agent)
  └─▶ [4] reframe       → crops.json      (Python, per selected clip)
  └─▶ [5] preview       → preview_*.mp4 + frames/*.jpg (Remotion, low-res)
  └─▶ [6] critique      → critique.json → plan.v2.json ... (max 3 iterations)
  └─▶ [7] final render  → out/<clip-id>/{reel.mp4, feed.mp4, cover.jpg, carousel_*.jpg, caption.md}
```

### [1] Ingest
- `ffprobe` → duration, fps, resolution, rotation, audio presence.
- Create 720p proxy for analysis and previews. Final render always uses the original.

### [2] Analyze (`services/analysis`)
Output `analysis.json`:
- `transcript.words[]`: `{ text, start, end, confidence, speaker? }`
- `transcript.sentences[]`: grouped words with start/end
- `scenes[]`: `{ start, end }` from PySceneDetect (ContentDetector)
- `audio.energy[]`: RMS per 0.5s window, normalized 0–1; `audio.onsets[]`
- `frames[]`: one sampled frame per scene + every 3s, saved as JPGs (512px wide), with timestamps
- `faces[]`: per sampled frame, bounding boxes (normalized)

Then the agent package adds `moments[]` via a vision pass: send batched frames + matching
transcript windows to Claude and score each candidate window (5–60s) on:
`hook_strength`, `clarity_standalone`, `emotion`, `visual_interest` (0–10), plus a one-line `why`.

### [3] Plan — Director agent
Inputs: `analysis.json` (with moments), `brand.json`, CLI options (`--clips N`, `--max-len`, `--goal`).
Output must validate against `EditPlanSchema` (below). Prompt rules:
- Select clips that stand alone without prior context; trim filler words and dead air at edges.
- The first 2 seconds must contain a hook: either a spoken line or an on-screen hook title.
- Produce exactly 3 hook variants per clip (different angles: curiosity, benefit, contrarian).
- Pick `emphasis` words (max 1 per sentence) for color highlight + punch-in zoom.
- Only use template IDs listed in the provided template registry.

### [4] Reframe
For each clip and each target aspect (9:16, 4:5):
- Track faces (MediaPipe) on proxy at 10fps; fallback to pose, then to saliency/center.
- Compute crop center per frame, then smooth with a critically-damped spring or
  One-Euro filter; add dead-zone (don't move unless subject leaves central 30%).
- On scene cuts, snap (no smoothing across cut boundaries).
- Output `crops.json`: keyframes `{ t, cx, cy, scale }` normalized to source.

### [5] Preview render
- Remotion renders at 540×960, `crf` high, to keep each iteration fast.
- Extract frames for the critic at: 0.0s, 0.5s, 1.0s, 2.0s, each emphasis word, and every 3s.

### [6] Critic loop
Critic receives: preview frames (images), the plan JSON, transcript of the clip, brand.json,
and the checklist below. Returns `CritiqueSchema`: `{ pass: boolean, score: 0-100, issues[]: { severity, check_id, timestamp?, fix } }`.
Director receives the critique and outputs a revised plan (full plan, validated).
Stop when `pass && score >= 80`, or after 3 iterations (keep the best-scoring plan).

**Critic checklist (check_id):**
- `HOOK_2S` — a clear hook is visible/audible within 2s
- `SAFE_ZONE` — no text/logo inside platform UI zones (see specs)
- `READABILITY` — caption text contrast and size readable on a phone; ≤ ~6 words on screen at once
- `SUBJECT_FRAMED` — speaker/subject not cut off in 9:16 and 4:5 crops
- `BRAND` — fonts/colors match brand.json; logo used per brand rules
- `PACING` — no dead air > 0.7s; a visual change (cut, zoom, text) at least every ~3s
- `ENDING` — clip ends on a complete thought or CTA, not mid-sentence
- `OVERLAP` — text layers don't overlap each other or faces

### [7] Final render + copy
- Reel: 1080×1920, 30fps (or source fps if 24/25/30), H.264 high profile, AAC 48kHz, yuv420p.
- Feed: 1080×1350 (4:5).
- Cover: 1080×1920 JPG from the best hook frame with hook title applied (also 4:5 crop for grid).
- Carousel: 3–7 stills (1080×1350) of key quotes over frames, using the `quote-card` template.
- `caption.md`: caption (first line = hook, ≤125 chars before "more"), body, CTA, 3–8 hashtags,
  alt text. Plus the 3 hook variants for A/B.

## Platform specs (verify periodically — Instagram UI changes)
- Reels safe zone (1080×1920): keep text/logo out of roughly the top ~250px, bottom ~450px,
  and right ~140px (action buttons). Encode these as constants in `packages/schemas/platform.ts`.
- Feed 4:5 is cropped to 3:4 in the profile grid; keep cover text centered.

## Schemas (`packages/schemas`) — source of truth
```ts
BrandSchema = {
  name: string,
  fonts: { heading: string, body: string },        // local font files in brand/fonts
  colors: { primary: hex, accent: hex, text: hex, background: hex },
  logo?: { path: string, placement: "top-left"|"top-right"|"end-card"|"none" },
  tone: string,                                      // e.g. "confident, playful, no jargon"
  banned_words?: string[],
  cta_default?: string
}

EditPlanSchema = {
  version: number,
  clips: Array<{
    id: string,
    source: { start: number, end: number },           // seconds in original
    cuts?: Array<{ start: number, end: number }>,     // removed ranges (filler/dead air)
    hooks: [Hook, Hook, Hook],                        // Hook = { text: string (≤8 words), angle }
    active_hook: 0|1|2,
    title: { template: TemplateId, text: string, in: number, out: number } | null,
    captions: { template: TemplateId, position: "center"|"lower", max_words: number },
    emphasis: Array<{ word_index: number, effect: "highlight"|"punch_in"|"both" }>,
    broll?: Array<{ at: number, duration: number, frame_from: number }>, // reuse own footage
    end_card: { template: TemplateId, cta: string } | null,
    music?: { mood: string, duck_db: number } ,
    copy: { caption: string, hashtags: string[], alt_text: string }
  }>
}
```

## Template registry (`packages/render/templates`)
Each template = a Remotion component + a JSON metadata file:
`{ id, kind: "caption"|"title"|"end-card"|"quote-card"|"lower-third", description, params schema, safe_area_aware: true }`.
The registry metadata (not code) is what the director sees in its prompt.

**v1 templates (build these first, polish before adding more):**
1. `caption-pop` — word-by-word, current word scales 1.0→1.12 with spring, emphasis words in accent color.
2. `title-slam` — hook title slides/scales in over first 0.4s, holds, exits by 2.5s.

**v2:** `caption-karaoke` (highlight sweep), `lower-third`, `end-card-cta`, `quote-card`, `progress-bar`.
All animations use Remotion `spring()`/`interpolate()`; no CSS transitions. All text respects safe zones.

## CLI
```
clipwright generate input.mp4 --brand brand/example.brand.json --clips 3 [--fast] [--quality max]
clipwright analyze input.mp4
clipwright plan input.mp4 --brand ...
clipwright render <plan.json> [--preview]
clipwright critique <plan.json>
```

## Conventions
- TypeScript strict; no `any` in schemas/agent. Prefer pure functions in stages.
- All prompts live in `packages/agent/prompts/*.md` with variables `{{like_this}}`; never inline long prompts in code.
- Log every LLM call (prompt hash, model, tokens, latency, validation result) to `.cache/llm-log.jsonl`.
- Time units: seconds (float) in schemas; convert to frames only inside Remotion.
- Coordinates: normalized 0–1 relative to source frame.

## Testing
- Unit: schema validation, crop smoothing, cut-list → timeline mapping, safe-zone math.
- Golden tests: render 2 fixtures with a fixed plan; compare sampled frames via perceptual hash.
- Agent tests: mock SDK responses from recorded fixtures; one live smoke test behind `LIVE=1`.

## Milestones
1. Ingest + analyze + schemas + CLI skeleton.
2. Director → plan.json on a fixture; manual plan rendering with `caption-pop` + `title-slam`.
3. Reframe (9:16, 4:5) with smoothing.
4. Critic loop end to end; measure score improvement across iterations.
5. Copywriting, cover, carousel, variants.
6. v2 templates, music ducking, batch mode for multiple input videos.

## Environment
- `ANTHROPIC_API_KEY` (required), `CLIPWRIGHT_MODEL` (optional)
- Requires `ffmpeg` on PATH, `uv`, Node 20+, pnpm.
