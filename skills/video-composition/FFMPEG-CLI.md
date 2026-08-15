# FFmpeg CLI — long filter graphs, and the option-from-file syntax

HyperFrames drives ffmpeg internally, but anything outside a composition — ping-pong loops, A/B comparisons, side-by-side cuts, audio mixes — is a hand-written ffmpeg call. Once the graph passes two or three stages, the shell becomes the hard part.

## `-filter_complex_script` was REMOVED in ffmpeg 9 — use `-/filter_complex`

Measured on `ffmpeg version n9.0.1`:

```
$ ffmpeg ... -filter_complex_script graph.filter ...
Unrecognized option 'filter_complex_script'.
Error splitting the argument list: Option not found
```

The replacement is the generic "read this option's value from a file" syntax — a slash before the option name:

```bash
ffmpeg -i a.mp4 -i b.mp4 -/filter_complex graph.filter -map "[v]" out.mp4
```

It is not filter-specific: `-/<option>` works for any option whose value you would rather keep in a file. Copy-pasted recipes from before ffmpeg 9 will carry the old spelling and fail with the message above — which names the option, so it is at least self-diagnosing.

## Why moving the graph to a file is worth doing anyway

A long `-filter_complex` written inline is a quoting trap: the graph contains `:`, `@`, `'`, `[`, `]` and commas, and a `drawtext=...:text='...'` broken by quoting does **not** fail with a quoting error. It fails with

```
Either text, a valid file, a timecode or text source must be provided
```

— i.e. it accuses the filter of having no text when the text was right there. That is hours of looking in the wrong place. Putting the graph in a file removes the entire failure class at once, and makes it reviewable in git.

The file also accepts newlines between stages — one `...[l];` stage per line — which is the only way a 3+ stage graph stays readable:

```
[0:v]scale=1080:1920[a];
[1:v]scale=1080:1920[b];
[a][b]hstack=inputs=2[v]
```

## Walls

- **`-filter_complex_script` no longer exists (ffmpeg 9+)** — `-/filter_complex <file>`.
- **A quoting break in `drawtext` reports "no text provided"** — the error names the wrong cause; suspect the shell first.
- **Inline graphs beyond two stages are a false economy** — the file version is versionable, diffable and line-wrapped.
