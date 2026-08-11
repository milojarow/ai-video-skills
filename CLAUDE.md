# CLAUDE.md — ai-video-skills repo

Guidance for Claude when editing THIS repo (not when using the skills inside a video project).

## What this repo is

A **public** Claude Code marketplace plugin shipping four sub-skills for producing short social-media videos end-to-end:

- `video-edit-setup` — local HyperFrames setup on rolling-release Linux (sharp / node-gyp / libvips gotchas).
- `video-asset-generation` — images (kie.ai Nano Banana 2), voice/music/SFX (ElevenLabs), image-to-video (Wan 2.6), hosting (Luna CDN).
- `video-captions` — word-synced captions (TikTok-pop + karaoke).
- `video-composition` — assembling assets into a final HyperFrames composition (watermark, CTA, safe-zones).

Plus `agents/` — two project-scoped subagents (scene-prompt-smith, asset-qa-validator) that travel with the plugin, and `notes/` — research provenance (not loaded as skills).

## Editing & release

1. Edit `skills/<sub-skill>/SKILL.md` (lean; `description:` frontmatter = WHEN-to-use triggers only) + `reference/*.md` (the depth).
2. Bump `version` in BOTH `.claude-plugin/marketplace.json` and `plugin.json` — they must match (autoUpdate keys off it). Keep the README status line in sync (treat it as a fourth place to bump).
3. Commit (English, imperative) + push. The marketplace auto-updates on Claude Code startup, not on `/reload-plugins`.

## Privacy posture (public repo)

- `.claude/memory/` is gitignored; **CLAUDE.md is committed** (skill repos are the one exception to the global gitignore-CLAUDE.md rule).
- No secret VALUES anywhere — only env-var names / the secrets-file path.
- Use `~` / `$HOME`, never a literal `/home/<user>`.
- **Teach-the-system, not inventory:** keep ONE illustrative voice and ONE illustrative project as examples; point at the source of truth for the rest (the ElevenLabs voice library filtered by es-MX; `ls ~/video-lab/` for past projects). A hardcoded roster of voices/clients goes stale on every change. The repo is public: no client names, no real campaign content, no operator-local absolute paths.

## Known gap: asset-qa-validator cannot judge an asset at its use size

`IMAGES.md` documents the use-size referee (downscale to the destination size, upscale with
`-filter point`, compare candidates in a montage) and lists "does not read at its use size"
as a REGEN criterion. **The `asset-qa-validator` subagent cannot apply it today**, for two
reasons:

1. Its dispatch contract has no **destination size** input — it receives only the image path,
   the prompt, and the protagonist clause. It has no way to know the asset is headed for a
   48 px button.
2. Its `tools:` list is `["Read"]`. The referee needs `magick`, i.e. `Bash`.

Designs considered, none built:

- Add an optional `destination size` field to the agent's input contract and a kill criterion
  that judges legibility at that size **by eye** from the full-resolution image. Cheap, no new
  tool, but it is a judgement call the agent will make badly — the whole point of the referee
  is that eyeballing the big version is the wrong ruler.
- Give the agent `Bash` and have it run the two `magick` commands itself, then Read the
  montage. Correct, but it widens a read-only QA agent to arbitrary shell for one metric, and
  it needs `magick` present on the machine (currently an undeclared dependency of the repo).
- Keep it out of the agent and make the use-size check a step the parent performs before
  dispatching QA. No agent change; costs a manual step per batch.

Unresolved — the count criterion (#8) shipped because it is purely visual and needs no new
input or tool; the use-size one is not the same shape. **Do not document a destination-size
parameter, a `magick` step, or a use-size criterion in the agent's spec until one of the
above is actually built** — an agent contract that promises an input the dispatcher never
sends teaches every future session to pass a field into a void.
