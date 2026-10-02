# lets-scroll

> Turn any brand, product, or industry into a **scroll-driven cinematic landing page**.
> As the visitor scrolls, a pre-rendered camera travels through a series of AI-generated
> scenes as **one unbroken shot** — scroll down flies it forward, scroll up plays it back.

**lets-scroll** is an agent skill for **Claude Code**, **Codex**, and any other
`SKILL.md`-compatible agent. It doesn't just prompt a video model — it runs the whole
production: interview → art direction → scene stills → a frame-locked camera chain →
encode → a portable scroll-scrub engine wired into your page.

The scroll position never triggers cuts or transitions. It simply moves time along a
single continuous camera path, so the **seams between scenes have to be perfect**.
Making them perfect — each clip conditioned on its neighbour's *actual rendered frames*,
never on a fresh render of the same scene — is the core of what this skill does. This is
the same technique behind Apple's scroll-through product pages: the camera genuinely
moves, scroll only drives time.

- **MIT licensed** — see [LICENSE](LICENSE)
- **Plugin version:** 0.8.0 — see [`claude-plugin/`](claude-plugin/)
- **No framework assumptions** — the scrub engine is self-contained vanilla JS

---

## Table of contents

- [Why](#why)
- [Features](#features)
- [Install](#install)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [Camera styles](#camera-styles)
- [What's in the skill](#whats-in-the-skill)
- [Repository layout](#repository-layout)
- [Configuration reference](#configuration-reference)
- [Manual asset path](#manual-asset-path)
- [Costs and budget](#costs-and-budget)
- [QA checklist](#qa-checklist)
- [Troubleshooting / gotchas](#troubleshooting--gotchas)
- [Notes](#notes)
- [License](#license)

---

## Why

Scroll-jacked "cinematic" pages usually cheat: they cut between pre-rendered clips, or
they try to fake 3D with WebGL and pay for it in bundle size, load time, and device
support. lets-scroll does neither:

| Approach | What breaks |
|---|---|
| Cut between clips | The eye sees the edit. There is no journey, only a slideshow. |
| Real-time 3D (Three.js/WebGL) | Heavy bundle, shader/perf work per device, constant art-direction drift. |
| **lets-scroll** | Pre-rendered, frame-locked video chain + a few KB of JS. The camera moves; scroll scrubs time. Works in plain HTML, Next.js, Vue, or a server-rendered page. |

The value of this skill is the **pipeline, the prompts, and the seam method** — not the
framework. The engine ships as one dependency-free file.

---

## Features

- **One unbroken shot.** Every seam is *frame-identical*: connectors/legs are
  conditioned on the actual last/first frames of their neighbours, so there is no pop,
  no cut, no flash between scenes.
- **Three camera personalities** — pick once at the interview; the skill implements it
  and never re-decides:
  - **Fly through the world** — dive into each scene, pull up and out, hop to the next
    (the flagship diorama look).
  - **One continuous walkthrough** — a single forward flight gliding scene-to-scene,
    never pulling back (the seamless choice for grounded, photoreal directions).
  - **Locked isometric glide** — one fixed camera angle for the whole film; the world
    slides past beneath the same view (calmest and cheapest to re-roll).
- **Two backends, one pipeline.** Stills via `gpt_image_2` on Higgsfield (or Codex
  `image_gen` on a ChatGPT subscription); the video chain via **Monid by default**
  (Seedance 2.0, pay-per-clip USD) with **Higgsfield credits as the fallback biller**.
- **Manual asset path.** Don't want the skill to spend credits? It writes every prompt
  to a file plus a `HANDOFF.md` spec table (prompt file, conditioning frame(s), exact
  output filename, live status column). You render in any tool that accepts a start
  frame — and end frame for fly-through connectors — and drop the files back in. The
  skill validates what returns (count, dimensions, aspect, duration, seam frames)
  before anything chains.
- **Portable scrub engine** — blob-loaded video (always seekable), rAF-smoothed
  scroll→time mapping, lazy prefetch of nearby clips, frame-matched seam crossfades,
  pinned per-section copy, route rail, `prefers-reduced-motion` support, and phone
  hardening (seek coalescing, iOS first-touch priming, safe-area CSS) on by default.
- **Optional native mobile chain.** A second camera chain rendered natively in **9:16
  portrait** — composed for phones, not a centre-crop of the landscape film — served
  automatically on coarse-pointer / ≤860px viewports.
- **Budget-first.** Render tiers and estimated costs are shown and approved *before*
  anything generates.

---

## Install

Copy the skill folder into your agent's skills directory:

```bash
# Claude Code
cp -R lets-scroll/skills/lets-scroll ~/.claude/skills/

# Codex
cp -R lets-scroll/skills/lets-scroll ~/.codex/skills/
```

Then just ask for a scroll-through world landing page, or invoke `/lets-scroll`
(`$lets-scroll` in Codex).

**Plugin distribution:** the repo also ships a Claude Code plugin manifest in
[`claude-plugin/`](claude-plugin/) — `plugin.json` (name, version `0.8.0`, keywords)
and `marketplace.json` (registers `skills/lets-scroll` under the `design` category) —
so the skill can be packaged and distributed as a plugin instead of copied by hand.

**No build step, no `npm install`.** The repository contains only Markdown, one
vanilla-JS engine, one Python helper, and JSON manifests.

---

## Requirements

| Tool | Why | Required? |
|---|---|---|
| [Monid CLI](https://monid.ai) + API key/balance | **Default video-chain backend** (Seedance 2.0, billed per clip in USD) | Default path |
| [Higgsfield CLI](https://higgsfield.ai) + credits (`higgsfield auth login`) | Renders scene stills (`gpt_image_2`), the `kling3_0` fallback, and the whole chain when Monid is absent | Automatic path |
| `ffmpeg` / `ffprobe` | Frame extraction and encoding | **Always** |
| Python 3 + Pillow | Mobile portrait canvases; optional transparent-scene knockout | Optional |
| [Codex CLI](https://github.com/openai/codex) ≥ 0.125, ChatGPT login | Scene stills via built-in `image_gen` (same GPT Image model), billed to the subscription instead of Higgsfield credits | Optional |
| A start-frame-capable video tool | Manual asset path only | Manual path |

Notes:

- **Monid default pricing** (verified 2026-07-25): token-priced
  `width × height × 24 × seconds / 1024` at $7/1M (480p/720p) or $7.7/1M (1080p).
  Measured: 1080p 8s dive ≈ $2.99, 5s connector ≈ $1.87; a 6-scene 1080p chain ≈ **$27**
  (~$11 at 720p). Pay-per-use, no subscription, no monthly expiry. First/last-frame
  conditioning frame-locks, so it renders the full seamless chain; frames travel via
  Monid's free workspace file system. The skill re-checks the endpoint schema each
  build and keeps qualification probes in the pipeline for when the catalog changes.
- **Manual path:** if you render stills/clips yourself from the skill's prompt files,
  none of the AI CLIs are needed — only `ffmpeg` (and Python for the optional knockout).
  Your video tool **must** accept a start frame (and, for fly-through connectors, an
  end frame) or the seams can't lock; the walkthrough architecture needs no end-frame
  support.
- **Caveats baked into the skill:** macOS ships bash 3.2 (no `declare -A`); Higgsfield
  generations take 3–8 min each so they always run detached and polled; media flags take
  local file paths, not job UUIDs; video models differ in accepted params — confirm the
  schema with `higgsfield model get <job_type>` before batching.

---

## Quick start

1. **Install** the skill (above) and check the CLIs: `monid --version`,
   `monid balance`, `higgsfield workspace list`, `ffmpeg -version`.
2. **Invoke** `/lets-scroll` (or just ask for a scroll-driven world landing page).
3. **Answer the interview** — subject/industry + one-line pitch, brand kit (import from
   a URL, hand it over, or have one proposed), art direction, camera style, journey size
   (2 / 4 / 6 scenes), mobile yes/no, asset source (automatic vs manual), and budget
   approval.
4. **Let it render** — N stills, then the camera chain (backgrounded + polled), then
   encodes.
5. **Drop in the engine** — copy `scrub-engine.js` (and `index-template.html` if you
   want a standalone page) into your project and call `mountLetsScroll(...)`.
6. **QA the seams** — screenshot just before/after each seam; the frames must be
   near-identical.

**Cheapest full-pipeline test:** a **2-scene teaser on the walkthrough architecture** —
2 stills + 2 sequential legs, no connectors, no end-frame support needed. The right
first run, and a good live demo.

---

## How it works

Two pipelines, one page.

**Art pipeline.** Every scene still renders with `gpt_image_2` under a shared *style
preamble*, so the whole world reads as one place. The default look is a soft isometric
clay diorama; photoreal, papercraft, glossy-toy and neon-night are first-class
alternates. Aspect, background lock and palette live in the prompt text itself, because
on the manual path each still may be rendered in a different tool or session.

**Motion pipeline.** The camera chain renders with frame-locking video models —
Seedance 2.0 via Monid by default, Seedance or Kling on Higgsfield credits as fallback.
Any model that can't hold a seam (no `--start-image` / `--end-image`) is disqualified.

When invoked, the skill:

1. **Interviews you** — subject, brand kit, art direction, camera style, journey size
   and ordered scenes, asset source, mobile option, and budget (tiers + estimates shown
   before anything generates).
2. **Generates the assets** — one still per scene, then the camera chain.
   - *Fly-through (Architecture B):* one "dive-in" clip per scene **+** connector clips
     joining consecutive scenes (`N` dives + `N−1` connectors).
   - *Walkthrough / locked-iso (Architecture A):* one forward **leg** per scene, no
     connectors at all — each leg starts on the previous leg's actual last frame
     (`N` legs).
   - Either way every seam is frame-identical, because each clip is conditioned on its
     neighbour's **actual rendered frames**.
   - Mobile opt-in renders a parallel portrait chain the same way, frame-locked against
     its own 9:16 renders.
3. **Wires it up** — a config-driven scroll engine that plays the whole chain as one
   flight, serving portrait clips and posters automatically on phones.

**The one rule that makes or breaks it:** seams must be frame-identical.

```
For each connector between dive_i and dive_{i+1}:
  start-image = the LAST frame extracted from dive_i's rendered video
  end-image   = the FIRST frame extracted from dive_{i+1}'s rendered video
```

Rendered frames — **never** the original diorama still. Every generation renders
slightly differently; handing off exact pixels is what makes
`dive_i.end == connector.start` and `connector.end == dive_{i+1}.start`. Insurance:
a short (few-frame) crossfade at each seam covers the model's near-miss landing.

---

## Camera styles

Chosen at the interview as `CAMERA`; Step 4 then *implements* the choice.

| Style | Architecture | Best for | Trade-off |
|---|---|---|---|
| **Fly through the world** | B — dives + aerial connectors | Diorama / miniature / god's-eye worlds | Reverses direction at seams — charming in miniature, jarring in realism |
| **One continuous walkthrough** | A — forward legs, no connectors | Grounded / photoreal / walkthrough | Strictly sequential rendering (can't parallelize) |
| **Locked isometric glide** | A + locked-iso clause in every leg prompt | Emons-style calm loops | Seedance can drift the angle on long legs — eyeball each last frame |

"Forward only" is the *seam* rule, not the *leg* rule: inside a single leg the camera is
free (orbits, crane-ups, lateral tracking are safe — one render, no seam to break).
Across a seam, velocity must never reverse. Every leg ends by settling into a slow,
steady forward drift and every leg begins by continuing that drift — the **motion
handoff contract**.

**Camera grammar** — pick the mid-leg move from the scene's own logic:

| Concept / tone | Mid-leg move |
|---|---|
| Product / luxury retail | Slow half-orbit around the hero object, then continue past |
| Real estate / hospitality | Steadicam glide through doorways; gentle crane-up in atria |
| Industrial / process / logistics | Low lateral track alongside the line, foreground parallax |
| Travel / outdoors / campus | Drone-style rise-and-reveal, then a descending swoop |
| Food / craft / detail-driven | Push in close to the craft moment, ease back, carry on |
| Playful miniature (arch. B) | Dives + aerial hops — the connector *is* the grammar |

**Video model roster** (one model for the whole chain — mixing models shifts render
character and reads as a pop):

| Model | start/end image | Notes |
|---|---|---|
| `seedance_2_0` (default) | ✓ / ✓ | Full chain, 1080p, `--mode std`. Touchy NSFW filter. |
| `kling3_0` | ✓ / ✓ | Full chain, 720p native, no `--resolution` param, `--sound off`. Sanctioned NSFW fallback. |
| `seedance_2_0_mini` | ✓ / ✓ | Cheap draft/previz tier that keeps frame-locking (720p). |
| `minimax_hailuo` | ✓ / ✗ | Cheapest probe (~6 cr/clip), architecture A only — qualify a leg-to-leg handoff first. |

---

## What's in the skill

```
skills/lets-scroll/
├── SKILL.md                    the procedure + the seam rule + gotchas
└── references/
    ├── prompts.md              intake checklist + every prompt template
    ├── pipeline.md             copy-paste batch scripts (generate → frames → connectors → encode)
    ├── scrub-engine.js         portable, config-driven scrub engine (blob-seek, lazy load, seam crossfade)
    ├── index-template.html     a minimal standalone page that mounts the engine
    └── knockout.py             background knockout for floating scenes
```

---

## Repository layout

```
lets-scroll/
├── README.md                   this file
├── LICENSE                     MIT
├── .gitignore                  node_modules, build output, env files, generated media
├── claude-plugin/
│   ├── plugin.json             plugin manifest (name, version 0.8.0, keywords, author)
│   └── marketplace.json        marketplace entry → registers skills/lets-scroll (design)
└── skills/
    └── lets-scroll/
        ├── SKILL.md            the skill procedure (frontmatter + Steps 0–8 + gotchas)
        └── references/
            ├── prompts.md      intake checklist + prompt templates
            ├── pipeline.md     batch scripts for the whole render pipeline
            ├── scrub-engine.js the scroll-scrub engine (drop into any page)
            ├── index-template.html  standalone mount page
            └── knockout.py     transparent-background helper (border flood fill)
```

Generated `.mp4` / `.webp` assets are produced per project and are **not** shipped in
this repository (see [.gitignore](.gitignore)).

---

## Configuration reference

```js
mountLetsScroll(document.getElementById('world'), {
  brand: { name: 'Pearl & Co.' },
  diveScroll: 1.3,   // viewport-heights of scroll per dive/leg clip
  connScroll: 0.9,   // viewport-heights of scroll per connector clip
  sections: [
    {
      id: 'farm',
      label: 'The Farms',
      still: 'assets/farm.webp',
      clip: 'assets/vid/farm.mp4',
      // mobile opt-in only — native 9:16 render + its first frame as poster
      clipMobile: 'assets/vid/farm-m.mp4',
      stillMobile: 'assets/farm-m.webp',
      // optional pacing: longer dwell (scroll) + camera settles mid-scene (linger 0–1, ≤ 0.6)
      scroll: 1.6,
      linger: 0.45,
      accent: '#8FB98A',
      eyebrow: 'From leaf to last sip',
      title: 'It starts in the hills.',
      body: '…',
      tags: ['Single-origin', 'Hand-picked'],
      // cta: { ... }  // last section usually carries the CTA
    },
    // …one entry per section
  ],
  // length must be sections.length - 1; entries may be null (engine crossfades that seam)
  connectors:       ['assets/vid/conn1.mp4', 'assets/vid/conn2.mp4'],
  connectorsMobile: ['assets/vid/conn1-m.mp4', 'assets/vid/conn2-m.mp4'], // mobile opt-in only
});
```

The engine handles: the ordered dive/connector chain, scroll→`currentTime` with rAF
smoothing, blob loading, lazy prefetch of nearby clips, frame-matched crossfades, pinned
per-section copy (first section greets on landing, last holds its CTA), a route rail,
`prefers-reduced-motion`, and mobile.

**Theming** is CSS variables (`--accent`, `--sw-bg`, `--sw-ink`, `--sw-font-*`, …).
Engine tokens are wrapped in `@layer sw`, so a page-level `:root` / `.sw-root` block wins
cleanly — no specificity hacks. `--sw-ink` is your primary text/heading colour; the
accent fills the primary button and active nav. The visual identity comes from the
generated clips, so keep the chrome quiet.

**Encoding** (Step 6) — seekability, not keyframe density, is what makes scrubbing
work. The engine plays each clip from an in-memory `Blob`, so blobs are always fully
seekable even when the host serves no byte ranges. Don't shrink quality:

```bash
ffmpeg -i src.mp4 -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart out.mp4
```

Native resolution (1080p — don't downscale), `crf 20`, small GOP, no audio, faststart.
Mobile encodes: 720 wide (`scale=720:-2`), `-g 4` (more keyframes = cheaper seeks on
phone decoders), `crf 23`.

---

## Manual asset path

`ASSET_SOURCE = manual` swaps the render calls for a prompt + file handoff; everything
downstream (frame extraction, encode, engine, QA) is identical.

1. The skill writes `still_<name>.txt`, `dive_<name>.txt`, `conn_<i>.txt` plus the exact
   conditioning frames each clip must start/end on.
2. You render in your own tools (any image tool for stills; a start-frame-capable video
   tool — plus end-frame for architecture B connectors — for clips).
3. The skill **validates what comes back**: count, dimensions, aspect, duration, and
   that frame 0 of each clip matches the handed-over start frame. A clip whose tool
   ignored the start frame goes back for re-generation — never crossfaded over.

Every handoff is presented as a **spec table**, never prose:

| Prompt file | Start frame | End frame | Save as | Status |
|---|---|---|---|---|
| `conn_1.txt` | `last_<scene1>.png` | `first_<scene2>.png` | `$WORK/conn_1.mp4` | pending |
| `conn_2.txt` | `last_<scene2>.png` | `first_<scene3>.png` | `$WORK/conn_2.mp4` | pending |

Acceptance rule: the start frame must be obeyed exactly; the end frame only needs to
land on the same composition (a Seedance-style near-miss is fine — the crossfade covers
it).

---

## Costs and budget

On the automatic path the skill states the estimated total **before** generating:

```
N stills + (2N − 1) videos  [× 2 if mobile]  + ~15% re-roll headroom
```

- **Walkthrough / locked-iso** renders `N` forward legs (no connectors).
- **Fly-through** renders `N` dives + `N−1` connectors.
- The **mobile chain doubles the video gens** (it's a second, natively-portrait chain).
- Monid pricing is per-token and printed per run. Higgsfield pricing isn't exposed by
  its CLI, so the skill calibrates against your live balance: run ONE still and ONE
  video first, diff `higgsfield workspace list` before/after, extrapolate, and warn
  whenever the estimate exceeds ~70% of the balance.
- A real `not_enough_credits` mid-run is recoverable (finished clips survive; resume
  after top-up) — the point of the budget step is that the user decides *before* the
  spend.
- On the **manual** path nothing is billed by the skill; you sign up for the workload
  (`N stills + (2N−1) clips [×2 if mobile]` + re-roll headroom) and render yourself.

Render tiers (every option frame-locks seams):

| Tier | Model | Rough cost |
|---|---|---|
| Draft / previz | `seedance_2_0_mini` (720p) — or Monid at 480p | ~¼ of Standard |
| Standard (default) | `seedance_2_0` (1080p) | baseline |
| Alternate | `kling3_0` (720p native) | ≈ Standard; different look + content filter |

Draft doubles as a previz path: run the whole chain cheap, approve the journey, then
re-render final legs on Standard — still seamless, so it translates directly.

---

## QA checklist

Drive the page in a headless browser and verify **frame continuity at the seams** —
the thing most likely to be wrong:

- [ ] Screenshot just before and just after each seam — the two frames must be
      near-identical. Judge by *composition*, not raw PSNR (a correctly frame-locked
      seam can read ~18–25 dB from detail shimmer alone).
- [ ] Console is clean; `video.seekable.end(0) > 0` (blob loading working).
- [ ] `currentTime` tracks scroll across each clip's band; scrolling up plays back.
- [ ] **Reduced motion** falls back to stills — no video, no particles.
- [ ] **Mobile (only if opted in)** — on a real/emulated phone, portrait + landscape:
  - Emulate with **CPU throttled 4–6×** and flick fast — the clip tracks without freezing.
  - First scene shows its still immediately; video takes over on first scroll (iOS
    priming) — no blank/black scene on iOS Safari.
  - Network panel confirms `-m.mp4` on mobile, the 1080p master on desktop.
  - Mobile clips are **natively portrait** (`videoWidth < videoHeight`), and
    `stillMobile` posters match each portrait clip's first frame (no flash).
  - Collapsing the URL bar doesn't jump the page; rotation recomposes cleanly.
- [ ] Desktop-only build: one sanity phone-viewport pass — loads, posters show, nothing
      overlaps (the engine's hardening covers graceful degradation).

---

## Troubleshooting / gotchas

| Symptom | Cause | Fix |
|---|---|---|
| **Seam pop** | Connector endpoints were the diorama stills, not the neighbouring clips' actual frames | Re-extract real frames (Step 5) |
| **Seam stutter / camera "jumps backward"** | Velocity reverses across a seam (forward dive → backward pull-out) — inherent to architecture B | Use architecture A for grounded walkthroughs |
| **Frozen video, stuck at frame 0** | `seekable=[0,0]` — host serves no byte ranges | Play from a `Blob` URL (the engine does) |
| **Huge files** | All-intra encoding | Use `-g 8` + blob instead |
| **Soft / low quality** | Downscaled or over-compressed | Native 1080p, `crf ≤ 20`, add `unsharp` |
| **503s / `not_enough_credits` race** | Transient when many gens launch at once | Re-roll the individual failure; verify the balance |
| **NSFW false positives (Seedance)** | Filter flags bedroom/pool/spa scenes and words like "bed", "pool", "wine" | Re-roll → strip trigger words + "empty, unoccupied, no people" → re-render that clip on `kling3_0` → worst case set the connector to `null` (engine crossfades that seam) |
| **Dark/custom theme fights the defaults** | Specificity clash | Set `--sw-bg` / `--sw-ink` / `--sw-accent` at `:root` — engine tokens live in `@layer sw` |
| **Phone stutters on fast flick** | 1080p master too heavy; seeks pile up | Ship `-m.mp4` mobile encodes (720p, `-g 4`) + wire `clipMobile`; tighten GOP further if needed |
| **Blank/black scene on iOS** | Safari quirk: muted video paints nothing until primed | The engine primes each video on first touch — keep that code path |

---

## Notes

- Asset generation costs money and takes a while — the skill runs generations in the
  background and polls. The estimated total is always stated before spending.
- Expressive mid-leg moves raise re-roll odds (the model may not end a fancy move in a
  clean forward drift). Mitigations: keep the final-second settle clause verbatim,
  eyeball each leg's last frame before chaining the next, and budget ~1 extra re-roll
  per expressive leg. A plain forward glide is the zero-risk default.
- Scroll is a scrubber: visitors can scroll **up**, so every move also plays in reverse.
  That's free — but it's why seam velocity must be consistent in both directions.
- Per-section pacing knobs: `scroll` (more scroll = longer dwell) and `linger` (0–1,
  keep ≤ 0.6) remaps time so the camera settles mid-scene exactly while the copy
  peaks. Prefer expressive motion in the *clip* and restraint in the *scrub mapping*.
- The generated `.mp4`/`.webp` assets are produced per project; they're not shipped
  here.

---

## License

MIT — see [LICENSE](LICENSE).

---

**Built by Girish Lade** — [ladestack.in](https://ladestack.in)
