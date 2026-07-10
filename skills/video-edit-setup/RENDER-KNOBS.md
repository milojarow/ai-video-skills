# Render Knobs, Timeouts & Machine Hygiene

HyperFrames 0.6.x render-time tuning: what the quality flag actually does, which flag combinations time out, undocumented env knobs, and the machine hygiene that prevents intermittent timeouts.

---

## `-q draft` is not "low quality output"

The `--quality` / `-q` flag tunes capture/encode presets, **not** the output bitrate target. Measured on the same 1080×1920 / 33s project: `draft` produced 2.51 Mbps, `high` produced 2.33 Mbps — draft came out *higher*.

For deliverables that get re-compressed hard downstream (WhatsApp, Reels), `draft` is perfectly deliverable. Don't chase `high` for the name alone — verify the actual output content instead of trusting the flag.

---

## Flag combinations that time out

Symptom: `Runtime.callFunctionOn timed out` (the CDP protocol timeout is 300s).

- `-c <file>` + `-q standard` or `-q high` → consistent timeout. `-c` + `-q draft` always passed.
- A composition with **no `<audio>`** on the main path, in `standard`/`high` → the calibration turns on `forceScreenshot:true` (capture ~8× slower) and blows the timeout.
- Heavy page + workers ≥ 2 + audio → timeout; the auto-calibration drops to 1 worker for a reason ("Chrome compositor starvation").

---

## Undocumented env knobs (`PRODUCER_*` family)

Found by grepping `dist/cli.js`:

- `PRODUCER_ENABLE_STREAMING_ENCODE=0` — disables the streaming-encode path that engages with 1 worker.
- `PRODUCER_CHUNK_SIZE_FRAMES`, `PRODUCER_ENABLE_CHUNKED_ENCODE` — exist, untested.

---

## Machine hygiene before a render batch

Intermittent timeouts are often resource contention, not a HyperFrames bug. Before an important render:

- Kill running studios: `pkill -f "hyperframes preview"` — their file watchers recompile on every change and steal CPU during renders.
- Kill headless browsers left over from any chrome-devtools MCP debugging session (they restore dozens of tabs and eat GB of RAM).
- Clean orphaned chromes from dead renders: `pkill -f chrome-headless-shell`.
- Verify before launching: `pgrep -af hyperframes` and `free -h`. Debugging leaves processes behind.

---

## `--docker` fallback

`--docker` expects a `docker` binary on PATH. On a podman-only machine, a shim makes it resolvable:

```bash
ln -s "$(command -v podman)" ~/.local/bin/docker
```

(Bridge documented for completeness; untested end-to-end.)
