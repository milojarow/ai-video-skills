# ai-video-skills

Skills for producing short videos for social platforms (WhatsApp Stories, IG Reels, FB Stories) end-to-end:
**kie.ai** for image generation + Wan 2.6 image-to-video animation, **ElevenLabs** for voice/music/SFX/transcription, **Luna CDN** for hosting, **HyperFrames** for composition.

Built so future Claude sessions go straight to what works — without re-discovering pricing, voice quirks, anti-patterns, or composition gotchas.

Modeled after [`n8n-mcp-skills`](https://github.com/czlonkowski/n8n-skills) by Romuald Członkowski: one plugin, many small focused sub-skills that load on demand based on what the user is asking.

---

## Sub-skills (Phase 1-2)

| Sub-skill | Triggers when... |
|---|---|
| **`video-edit-setup`** | Installing HyperFrames, scaffolding a new video project, hitting sharp/node-gyp/libvips errors, or deciding directory layout. |
| **`video-captions`** | Adding closed captions / subtitles (TikTok-style word-by-word, karaoke, or block style) to a video. Comparing Whisper vs Scribe. Hitting `overlapping_clips_same_track`, phantom words, or frame freezing. |
| **`video-asset-generation`** | Generating any video asset programmatically — images (kie.ai Nano Banana 2 with papercut or cinematic-realism styles), voiceovers (ElevenLabs TTS — Antonio/Camila/Marto/Jose), music + SFX (ElevenLabs), image-to-video animation (kie.ai Wan 2.6), and Luna CDN hosting. |
| **`video-composition`** | Assembling pre-generated assets into a final HyperFrames composition — multi-track img+video+audio, watermark with 4-corner rotation, URL typewriter CTA, fade-to-black ending, platform-specific safe-zones (WhatsApp Stories / IG Reels / FB Stories). |

---

## Repo layout

```
ai-video-skills/
├── .claude-plugin/                    # plugin metadata
│   ├── plugin.json
│   └── marketplace.json
├── skills/                             # the actual skills
│   ├── video-edit-setup/
│   │   └── SKILL.md
│   ├── video-captions/                 # captions only
│   │   ├── SKILL.md                    # overview + when-to-use
│   │   ├── PIPELINE.md                 # TikTok-pop full build script
│   │   ├── KARAOKE.md                  # Karaoke style — final tuned config
│   │   ├── PROVIDERS.md                # Whisper vs Scribe deep dive
│   │   └── TROUBLESHOOTING.md          # error catalog
│   ├── video-asset-generation/         # images, voice, music, SFX, animation
│   │   ├── SKILL.md                    # overview + decision matrix
│   │   ├── IMAGES.md                   # kie.ai Nano Banana 2 + 2 styles + anti-patterns
│   │   ├── VOICES.md                   # ElevenLabs TTS — 4 Mexican voices, pre-test pattern
│   │   ├── MUSIC-SFX.md                # ElevenLabs Music + Sound Generation
│   │   ├── ANIMATION.md                # kie.ai Wan 2.6 image-to-video full guide
│   │   └── LUNA-CDN.md                 # Luna CDN — 2 keys + upload patterns
│   └── video-composition/              # HyperFrames assembly + branding
│       ├── SKILL.md                    # workflow integrator
│       ├── COMPOSITION.md              # img+video+audio multi-track patterns
│       ├── WATERMARK.md                # 4-corner rotation w/ compactar/descompactar
│       ├── ENDINGS.md                  # URL typewriter + fade-to-black
│       └── PLATFORMS.md                # WhatsApp Stories / IG Reels / FB safe-zones
├── agents/                             # project-scoped subagents (travel with the plugin)
│   ├── scene-prompt-smith.md
│   └── asset-qa-validator.md
├── notes/                              # research notes (not loaded as skill)
│   ├── exp-01-captions-whisper-vs-scribe.md
│   └── eval-01-openmontage-vendored-hyperframes.md   # framework audit: is HyperFrames already vendored inside it?
└── README.md
```

The `notes/` directory holds raw findings from the experiments that produced these skills. The skills themselves are the distilled, prescriptive form. If something is in `notes/` but not in a `SKILL.md`, it's because it didn't make the cut for the prescriptive guide.

---

## Phases

| # | Topic | Status |
|---|---|---|
| 1 | Closed captions — word-by-word TikTok-style + karaoke | ✅ **Done** (v0.1, `video-captions`) |
| 2 | Asset generation — images, voice, music, SFX, animation | ✅ **Done** (v0.2, `video-asset-generation`) |
| 2 | Composition — HyperFrames assembly + branding | ✅ **Done** (v0.2, `video-composition`) |
| 3 | Motion effects — zoom in/out, slow/fast-mo, scene transitions beyond what Wan does | Backlog |
| 4 | Multi-language adaptation — voice + caption variants for non-Spanish markets | Backlog |

Each phase produces sub-skills under `skills/`. The convention from `video-captions` (SKILL.md overview + auxiliary references) is the template.

---

## Project workspaces

This repo is the skill — what gets published. **It does not contain `node_modules` or video assets.**

Actual video projects (where HyperFrames is installed and renders happen) live separately:
```
~/video-lab/<topic>/<video-name>/<variant>/
```

See `skills/video-edit-setup/SKILL.md` for the full directory convention and rationale.

Reference production projects (one papercut pilot without Wan animation, one cinematic-realism short with 3 Wan clips) live in the operator's local `~/video-lab/` tree, following the convention above.

---

## Repo rules

- Public GitHub repo (`milojarow/ai-video-skills`), distributed via marketplace + auto-update.
- No client names, no secret values, no operator-local absolute paths — env-var names only.
- Credentials and large binaries in `.gitignore`.
- Every experiment commits its findings to `notes/` before being distilled into skill content.

---

## Status

**v0.5.0** — Phases 1 + 2 complete and production-tested (Phases 3-4 remain backlog). Public repo, distributed as a Claude Code plugin marketplace with auto-update: improvements reach every install on the next Claude Code start.

Install:
```
/plugin → Marketplaces → Add Marketplace → milojarow/ai-video-skills
```
