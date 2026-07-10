---
name: video-edit-setup
description: Local setup for HyperFrames-based video editing projects on rolling-release Linux distros (Cachy/Arch). Use this skill whenever installing HyperFrames, scaffolding a new video project with `hyperframes init`, hitting sharp/node-gyp/node-addon-api install errors, or deciding directory layout. Also use when the user mentions Node.js version errors with HyperFrames, libvips compatibility, asks about separating skill code from project workspaces, or hits render-time issues — `Runtime.callFunctionOn timed out`, what `-q draft` vs `high` actually changes, `PRODUCER_*` env knobs, or machine hygiene before a render batch. The setup gotchas here are non-obvious and cost ~hour to discover from scratch — consult this skill before letting the user run `npm install hyperframes`.
---

# Video Edit Setup

Get HyperFrames installed cleanly on a rolling-release Linux distro and lay out directories so the skill repo stays separate from project workspaces.

> **📹 ACTIVE-SKILL MARKER:** Prefija tu reply con 📹 **solo en turnos donde el trabajo toca el dominio de `video-edit-setup`** — setup de edición de video — inicializar proyecto, file layout. La **capa/proyecto da igual** (frontend, backend, n8n, script local — todos valen): lo que importa es si *este turno* toca el dominio. En turnos que NO lo tocan (typecheck, build, deploy, git ops, edición o curl de otros dominios), **omite 📹** aunque la skill se haya cargado antes en la sesión. Si otras skills activas también aplican al mismo turno, **apila sus emojis** en el prefijo.

---

## Why this skill exists

HyperFrames depends on `sharp` for image processing. On distros with bleeding-edge glibc (Cachy/Arch rolling), `sharp` has no compatible prebuilt binary and falls back to building from source — which fails by default because `node-addon-api` and `node-gyp` aren't auto-pulled. Discovering this combination is ~1 hour of trial and error. This skill captures the working recipe so the next setup is 5 minutes.

A second, separate gotcha: people mix the skill repo with the project workspace, which contaminates the publishable skill with `node_modules/`, video assets, and renders. The directory convention here keeps them clean.

---

## Node.js version

**Use Node 22 LTS, not Node 25** (or whatever is bleeding edge). Sharp ships prebuilts for the active LTS line; on bleeding-edge Node it falls through to source builds.

```bash
nvm install 22
nvm use 22
node --version    # → v22.x.x
```

If `nvm` isn't loaded in fish/zsh/bash, source it explicitly or use the absolute binary path:
`~/.nvm/versions/node/v22.20.0/bin/node` (replace version with what `nvm` installed).

If a project must run with a specific Node version regardless of shell state, prefer absolute paths in scripts over relying on shell-side `nvm use`. Each Bash invocation may start in a fresh subshell that doesn't have nvm sourced.

---

## Sharp install — the working recipe

**Required system package** (one-time):

```bash
sudo pacman -S libvips           # provides libvips and headers (≥ 8.18)
pkg-config --modversion vips     # verify ≥ 8.18
```

**Per-project install** (always together, in this order):

```bash
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-addon-api
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-gyp
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install hyperframes
```

Why all three commands?

- `SHARP_FORCE_GLOBAL_LIBVIPS=1` tells sharp to use the system's `libvips` instead of fetching its own — required when no prebuilt matches the system.
- `node-addon-api` and `node-gyp` must be in the project's `package.json` before sharp's postinstall runs, otherwise sharp's source build aborts with "Please add node-addon-api/node-gyp to your dependencies".
- Doing it in three separate `npm install` calls keeps each step's failure isolated — easier to diagnose than one combined command.

**Verification**:

```bash
ls node_modules/sharp/build/Release/*.node    # the .node binary must exist
./node_modules/.bin/hyperframes --version     # should print 0.5.x
```

If `sharp/build/Release/*.node` is missing, the install was partial and `hyperframes` will be broken even if the bin symlink exists. Re-run the install.

---

## Directory layout convention

Two distinct concepts. Don't mix them.

### The skill repo (this directory)

`~/projects/ai-video-skills/` — what gets published. Contains `SKILL.md` files, sub-skills, references, prompts, examples. **Never** install `hyperframes` here. Never put project assets here.

### The project workspaces

`~/video-lab/<topic>/<video-name>/<provider-or-version>/` — where actual video projects live. Contains `node_modules/`, source `.mp4`, transcripts, renders. Each `<topic>` is a logical grouping (e.g., `captioning-experiments`, `client-marketing-2026`). Each `<video-name>` is one source video. Each `<provider-or-version>` is one variant (e.g., `A-whisper`, `B-scribe`, `final-v3`).

Example real layout:

```
~/projects/ai-video-skills/        ← skill (publishable)
└── skills/, notes/, README.md

~/video-lab/                          ← project workspace (private)
└── captioning-experiments/
    └── especialista-segurosdevida/
        ├── node_modules/             ← shared by all variants below
        ├── package.json
        ├── A-whisper/                ← one HyperFrames project
        │   ├── index.html, build-captions.mjs, video.mp4, transcript.json
        └── B-scribe/                 ← another variant of same source
            └── ... same shape ...
```

`node_modules/` lives **above** the variants when they share dependencies — saves disk and keeps installs in one place. The `hyperframes` bin is invoked via relative path: `../node_modules/.bin/hyperframes render --output ...`.

---

## Common setup errors

### `sharp: Please add node-addon-api to your dependencies`
The recipe above wasn't followed. Run the three `npm install` commands in order with `SHARP_FORCE_GLOBAL_LIBVIPS=1`.

### `sharp: Please add node-gyp to your dependencies`
Same root cause. The full recipe pulls both deps before sharp's postinstall.

### `sharp build from source` then silent failure, leaving no `.node` binary
Means libvips system package is missing or too old. Verify `pkg-config --modversion vips` returns ≥ 8.18.

### `node: undefined symbol: ...`
Node version mismatch — sharp built against one Node ABI, you're running another. Re-run install on the target Node version.

### `hyperframes` bin missing after npm install
`npm` cleanup deleted the partial install when sharp failed. Re-run install with the full recipe.

---

## Why directory separation matters

If the skill repo contains `node_modules/` and project files:

1. `git status` is noisy with project-local changes
2. The `.gitignore` becomes a tangle of "these node_modules but not those"
3. Publishing the skill includes garbage no consumer needs
4. Multiple projects can't share the skill cleanly

If the project workspace contains skill files:

1. The same skill gets duplicated per project
2. Updates to the skill must be replicated everywhere
3. There's no "source of truth" for the skill itself

The convention is symmetrical: skill content stays in `~/projects/<skill-name>/`, all working artifacts go to `~/video-lab/<topic>/<video>/<variant>/`. The `~/video-lab/` tree is **never** committed to the skill repo.

---

## Render-time knobs & machine hygiene

What `-q draft` actually changes (hint: not output bitrate), which flag combinations trigger `Runtime.callFunctionOn timed out`, the undocumented `PRODUCER_*` env knobs, and the process hygiene that prevents intermittent render timeouts: [RENDER-KNOBS.md](RENDER-KNOBS.md).

---

## Quick start for a new project

```bash
# 1. ensure setup is done (one-time per machine)
nvm use 22
sudo pacman -S libvips             # if not already installed

# 2. create the project workspace
mkdir -p ~/video-lab/<topic>/<video-name>
cd ~/video-lab/<topic>/<video-name>
npm init -y

# 3. install hyperframes (every new workspace)
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-addon-api
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-gyp
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install hyperframes

# 4. scaffold the first variant
./node_modules/.bin/hyperframes init <variant-name> \
    --video /path/to/source.mp4 \
    --example blank --non-interactive

# 5. variant scaffolds with index.html, hyperframes.json, package.json,
#    and a copy of the source video. Edit, lint, render from inside.
```
