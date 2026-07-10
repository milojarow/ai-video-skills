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
