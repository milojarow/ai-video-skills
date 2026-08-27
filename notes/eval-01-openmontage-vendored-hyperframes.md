# Eval 01 — OpenMontage (calesthio/OpenMontage): what it actually is, before installing

Audited via the GitHub API + reading files directly, without cloning or running `make setup`.
A real, active, permissively-starred repo, AGPL-3.0 licensed.

## The finding that saves the whole audit

**Its composition engine IS the HyperFrames stack, vendored.** Its own bridge doc for the
engine does not teach HyperFrames — it explicitly says "for the raw knowledge read the layer-3
skills" and points at the same HyperFrames skill pack this repo already wraps (core, creative,
animation, cli, registry, media, plus `media-use`, `motion-graphics`, `music-to-video`,
`website-to-video`, `remotion-to-hyperframes`).

Its skills directory ships **over a thousand files across ~85 packs**. Before adopting a repo
like this, check the overlap first: pull its file tree recursively and grep for packs you
already have installed. High overlap means the new thing is not the render engine — it's a
governance layer on top of an engine you already run.

⚠️ **A vendored copy can be stale.** In this audit it still shipped a HyperFrames pack that
upstream had already retired in favor of a newer one (it carried both, old and new side by
side). A vendored pack is a snapshot, not a live link — don't assume it tracks upstream.

## What is actually original (the governance layer)

Multiple pipeline YAMLs (explainer, hybrid, documentary-montage, clip-factory, talking-head,
cinematic, podcast-repurpose, screen-demo, localization-dub, animation, avatar-spokesperson,
character-animation) · a large set of stage-director skills · many Python tools · JSON schemas
· a live dashboard · budget tracking with reserve + reconciliation · resumable checkpoints.

Each pipeline runs idea → script → scene_plan → assets → edit → compose → publish, every stage
carrying `checkpoint_required`, `human_approval_default`, `review_focus`, and
`success_criteria` — plus a large reviewer skill doing automated post-render self-review
(frame extraction + audio analysis + video-probe metadata).

## What's worth reading even if you never install it

| File (relative to `skills/`) | Why |
|---|---|
| `meta/reviewer.md` | formalized automated post-render self-review — exactly the class of defect that passes lint with zero errors |
| `creative/short-form.md` | per-platform safe zones (TikTok/Reels/Shorts/FB) at 1080×1920, duration↔completion tables, upload specs |
| `creative/broll-planning.md` | B-roll planning |
| `creative/video-editing.md`, `sound-design.md`, `storytelling.md`, `typography.md` | editing craft, written as prose |
| `meta/taste-direction.md` | a "design read" contract + three dials (visual_variance / motion_intensity / information_density) |

⚠️ The retention tables in `short-form.md` mix sources — some official (platform creator docs),
some from marketing blogs. Treat them as direction, not as measurement.

⚠️ AGPL-3.0: using it as a tool to produce videos does not make the videos a derivative work.
The network clause bites if you wrap it inside a service you offer over the network. Don't copy
its prose/tables verbatim into a repo under a different license.

## The cost of adopting it

`make setup` installs a Python venv + `requirements.txt`, runs `npm install` in a Remotion
composer directory, installs a TTS package, and warms an npx cache. In short: it installs
packages, unprompted — worth surfacing explicitly before running it, under the standing rule of
never installing without approval.

## The rule this produces

**Before adopting an agentic video framework, measure whether its composition layer is already
yours.** If you already own and know the engine, the only thing a wrapper like this can add is
governance (staged approval, drift detectors, automated self-review) or craft written in prose
— and both are harvested by reading a handful of files, not by installing thousands.
