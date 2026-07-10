# Experimento 01 — Closed Captions: Whisper vs Scribe

> Este documento es el mapa para el siguiente Claude que aborde captions
> word-by-word style (TikTok/Reels) en HyperFrames. Si vas a hacer captions,
> léelo primero y vas directo a lo que funciona.

**Fecha:** 2026-05-08 · **Estado:** Whisper y Scribe ambos viables; **Scribe gana** (reñido).

## TL;DR — el camino corto

1. Setup local de HyperFrames es delicado en distros rolling-release. Sigue [Setup local](#setup-local) o vas a perder tiempo.
2. Para STT: **ElevenLabs Scribe es preferible** por timestamps más precisos. Whisper sirve si el costo importa más.
3. Hay 5 bugs/quirks de cada provider que **debes manejar en post-processing**. Listados abajo.
4. Estructura: skill repo en `~/projects/<skill-name>/` (solo skill); proyectos en `~/video-lab/<topic>/<video>/<provider>/`. **No mezclar.**

---

## Setup local (¡importante!)

### Node version
- Usar **Node 22 LTS** vía nvm. **No Node 25** (sharp build falla con `node-addon-api missing`).
- `nvm use 22` en cada shell, o llamar el binary absoluto: `~/.nvm/versions/node/v22.20.0/bin/...`.

### Sharp en Cachy/Arch rolling
HyperFrames depende de `sharp`. En distros con glibc bleeding-edge no hay prebuilt compatible — hace build-from-source. Combinación que funciona:

```bash
# 1. Sistema (una vez): pacman -S libvips (debe ser >= 8.18)
# 2. Por proyecto:
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-addon-api
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-gyp
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install hyperframes
```

Sin `SHARP_FORCE_GLOBAL_LIBVIPS` falla. Sin `node-addon-api` y `node-gyp` como deps falla. Verificado.

### Estructura de directorios
- **Skill repo**: `~/projects/<skill-name>/` — solo lo que se va a publicar (README, sub-skills, notes, docs). Sin `node_modules`. Sin proyectos hyperframes.
- **Workspace de proyectos**: `~/video-lab/<topic>/<video-name>/<provider-version>/` — donde viven assets, node_modules, renders. NO se publica.

Mezclar los dos es el error #1 que cometí. Sépáralos desde el inicio.

---

## Pipeline (alto nivel)

```
video.mp4
   │
   ▼
[1. STT call] ─── Whisper API   →  whisper-verbose.json
              └── Scribe API     →  scribe-response.json
   │
   ▼
[2. Build script (Node)] — lee JSON, normaliza, resuelve phantoms,
                            aplica shift, genera index.html con
                            <span class="word clip"> + GSAP timeline
   │
   ▼
[3. hyperframes lint] — debe quedar 0 errors. Warnings de "file too
                        large" / "track too dense" son tolerables.
   │
   ▼
[4. hyperframes render --quality draft] — iterar
[5. hyperframes render --quality high]  — final
```

---

## STT: Whisper vs Scribe (lo que importa)

### OpenAI Whisper API

```bash
curl https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F "file=@video.mp4" \
  -F "model=whisper-1" \
  -F "language=es" \
  -F "response_format=verbose_json" \
  -F "timestamp_granularities[]=word" \
  -F "prompt=Contexto del audio + nombres propios."
```

**Quirks que SÍ vas a encontrar:**

1. **Timestamps atrasados ~150ms** respecto al onset real del audio. Resultado: caption aparece tarde. Fix: shift global de **-0.15s**.
2. **Phantom words**: a veces emite `start == end` (zero duration). Fix: split-forward (la phantom toma la primera mitad de la duración de la siguiente palabra) o merge (concatena texto).
3. **Vocabulario propio**: nombres mexicanos como "Ahuja" salen como "Abuja". Fix: usar `prompt` parameter con contexto rico:
   `"El locutor es <nombre completo del locutor>, agente de seguros..."`
4. **El MCP `mcp__whisper__transcribe_audio`** del wrapper local NO pasa `response_format` ni `timestamp_granularities` — solo da texto plano. Para word-level usa **curl directo a la API**.
5. Costo: ~$0.006 USD/min.

### ElevenLabs Scribe

```bash
curl https://api.elevenlabs.io/v1/speech-to-text \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -F "file=@video.mp4" \
  -F "model_id=scribe_v1" \
  -F "language_code=spa" \
  -F "tag_audio_events=false"
```

**Diferencias de shape contra Whisper** (ojo al hacer adapter):

- `audio_duration_secs` (no `duration`)
- `language_code` (no `language`)
- `words[]` contiene tanto `type: "word"` como `type: "spacing"` — **filtrar `type === "word"`**
- Field por palabra es `text` (no `word`)
- Da `logprob` por palabra (confianza) — útil para flagging de errores

**Quirks que SÍ vas a encontrar:**

1. **Timestamps son más precisos** que Whisper-1. No necesitamos shift global.
2. **No tuvo phantom words** en este test (148 words limpios, vs 10 phantom en Whisper).
3. **Vocabulario propio**: "Ahuja" → "Aguja". El parámetro `biased_keywords=Ahuja:5` **no arregló** la transcripción en mi test. Solución: post-process con map `VOCAB_FIXES` en el script.
4. **Omite palabras esporádicamente** (en este test, omitió la "y" en "y disfrutar de una vejez").
5. Costo: ~$0.04 USD/min (~6.7× más caro que Whisper).

### Decisión

| Caso | Provider |
|---|---|
| Default (precisión > costo) | **Scribe** |
| Costo crítico, tolera shift heurístico | Whisper |
| Pre-producción / batch grande | Whisper, post-procesar |
| Cliente / final delivery | Scribe |

---

## Composition (HTML + hyperframes)

### Reglas non-negotiable

- Root `<div>` con `data-composition-id`, `data-width`, `data-height`
- Cada elemento timed: `data-start`, `data-duration`, `data-track-index`, `class="clip"`
- `<video>` debe ser `muted` + `<audio>` separado para que hyperframes mezcle el audio
- GSAP timeline `paused: true`, registrado en `window.__timelines["composition-id"]`
- Sin `Math.random()`, sin `Date.now()`, sin `fetch()` durante setup (rompe determinismo)

### TikTok-style word-pop (lo que funcionó)

```html
<span id="w42" class="word bottom clip"
      data-start="3.42" data-duration="0.18" data-track-index="1">palabra</span>
```

Y GSAP por palabra:

```js
tl.fromTo("#w42", { scale: 0.7, opacity: 0 },
                  { scale: 1.05, opacity: 1, duration: 0.04, ease: "back.out(2)" }, 3.42)
  .to("#w42", { scale: 1.0, duration: 0.04, ease: "power2.out" }, 3.46);
```

**Pop duration ≤ 80ms** (no 150ms). Más largo causa atropello visual cuando palabras adyacentes son cortas (típico en español hablado rápido).

### Non-overlap enforcement

Hyperframes lint **exige** que clips en el mismo `data-track-index` no se solapen. Whisper a veces reporta `B.start < A.end` (overlap real) y floating-point genera "ghost overlaps" tipo `30.001 vs 30.0`. Fix:

```js
const GAP = 0.01; // 10ms entre palabras
end = Math.min(end, nextStart - GAP);
const dur = Math.max(0.001, end - start);
```

La prevención de overlap manda sobre la duración mínima — acepta palabras de 1ms si Whisper las quiere.

### Posicionamiento dinámico (top vs bottom)

Si el source video tiene un lower-third overlay (nombre + cargo), los CC tapan ese texto. Fix: clase dinámica.

CSS:
```css
.word { /* shared style */ }
.word.bottom { bottom: 80px; }
.word.top    { top: 30px; }
```

Logic en build script:
```js
const TOP_WINDOW = { start: 0.7, end: 5.5 }; // anchored to transcript words
const isTopWord = (w) => w.start >= TOP_WINDOW.start && w.start <= TOP_WINDOW.end;
```

**Anclar la ventana a palabras del transcript** (no a segundos absolutos hardcoded). Ej: "desde antes de la palabra `nombre` hasta después de `profesión`". Más robusto si el video se reedita.

`top: 30px` en un canvas 576px de alto deja la cara del locutor (centro) intacta. Adapta para otra resolución.

### Dimensiones

**Mantén la composition a la resolución del source video.** Si source es 1024×576, composition es 1024×576. No upscale forzado a 1920×1080 — `object-fit: cover` escala el video con pérdida de calidad pero ganando nada.

---

## Source video preprocessing

Hyperframes warning durante render:
> *"Video has sparse keyframes (max interval: 8.33s). This causes seek failures and frame freezing."*

Fix antes de renderizar:

```bash
ffmpeg -i source.mp4 \
  -c:v libx264 -r 30 -g 30 -keyint_min 30 \
  -movflags +faststart -c:a copy \
  source.fixed.mp4
```

`-g 30 -keyint_min 30` fuerza un keyframe cada 30 frames (1s @ 30fps). El render seek-driven de hyperframes captura cada frame por separado — sin keyframes densos, hay frame freezing.

No fatal pero recomendado siempre. En el experimento no lo hicimos y el video se vio bien, pero pude ser suerte.

---

## Errores comunes y sus fixes

| Error de lint/render | Causa | Fix |
|---|---|---|
| `overlapping_clips_same_track` | Floating-point o real overlap | 10ms GAP entre clips, `toFixed(3)` |
| `media_missing_data_start` | `<video>` o `<audio>` sin `data-start` | Siempre `data-start="0"` en root media |
| Sharp install fails | Distro rolling-release | `SHARP_FORCE_GLOBAL_LIBVIPS=1` + node-addon-api + node-gyp |
| CC parecen atrasados | Whisper-1 timestamps tarde | `SHIFT = -0.15` global |
| CC tapan lower-third overlay | Posición fija | Clase dinámica `top` vs `bottom` con TOP_WINDOW |
| Pop atropellado entre palabras | Pop duration > word duration | Pop ≤ 80ms |
| Frame freezing en render | Source con keyframes sparse | Re-encode con `-g 30 -keyint_min 30` |

---

## Mejoras conocidas (no implementadas en exp-01)

- **Agrupar palabras consecutivas muy cortas** en un solo span de "palabra1 palabra2" cuando ambas duran < 150ms y están separadas por < 50ms. Mejora legibilidad en habla rápida.
- **Sub-compositions** para scenes (warning del lint sobre "file too large" / "track too dense"). Un `compositions/captions.html` montado vía `data-composition-src`.
- **Fix en MCP server `audio-transcriber`**: agregar parámetros opcionales `response_format` y `timestamp_granularities` para que el MCP también devuelva word-level. Vive en `~/.local/share/mcp/audio-transcriber/`.
- **VOCAB_FIXES por proyecto**: el map vive hardcoded en el build script. Mejor mover a un archivo `vocabulary.json` por proyecto.
- **Source video preprocessing automático**: detectar el warning de sparse keyframes en el primer render y re-encode automáticamente.

---

## Costos y tiempos del experimento (reference)

| Operación | Tiempo | Costo |
|---|---|---|
| Whisper API call (51s video) | 5-7s | ~$0.005 USD |
| Scribe API call (51s video) | 4-5s | ~$0.034 USD |
| `hyperframes init --video` (con video copy) | 2-3s | $0 |
| `hyperframes render --quality draft` (51s, 1543 frames, 3 workers) | ~2m 50s baseline | $0 (local) |
| `hyperframes render --quality draft` con `prime-run` + `--gpu` | ~2m 28s | $0 (local) |
| `hyperframes render --quality high` | ~4-5 min | $0 (local) |

## GPU acceleration (prime-run + --gpu)

En sistemas con PRIME offloading (laptops con dual GPU integrated + NVIDIA), los renders se aceleran prefijando `prime-run` y agregando el flag `--gpu` al render:

```bash
prime-run ./node_modules/.bin/hyperframes render --output out.mp4 --gpu
```

`prime-run` hace que Chrome y FFmpeg vean la NVIDIA discreta como GPU principal. `--gpu` activa NVENC para H.264 encoding hardware-accelerated.

### Benchmark medido (Quadro M1000M, video 51s 1024×576, 3 workers, draft)

| Config | Wall time | Speedup | Output size |
|---|---|---|---|
| Sin prime-run, sin --gpu | 2m 49.7s | 1.00× | 9.94 MB |
| `prime-run` + `--gpu` | 2m 28.6s | **1.14×** | 10.5 MB |

**12.5% más rápido en draft.** Mejora modesta porque ~88% del tiempo es captura serial de frames (Chrome seek), no encoding.

### Cuándo es relevante adoptarlo

| Escenario | Vale la pena? |
|---|---|
| Iteración con `--quality draft` | Marginal (~12%) — opcional |
| Render final con `--quality high` (x264 slow preset) | **Sí** — encoding es mayor parte del tiempo |
| Compositions con WebGL/Three.js (Fase 4) | **Sí, notable** — GPU rendering domina |
| Videos largos (5+ min) | **Sí** — el ahorro escala lineal |
| Render inside Docker | No aplica — Docker mode usa software-GL para determinismo |

**Patrón recomendado**: usar `prime-run` + `--gpu` siempre que el render no sea de iteración rápida. Para iterations en `draft` se puede omitir.

### File size con NVENC

NVENC produce archivos ~6% mayores que x264 ultrafast en mismo CRF nominal. No es pérdida de calidad — es cómo el encoder asigna bits. Si el size importa, usar `--video-bitrate` para forzar target.

---

## Próximas etapas (roadmap)

Esta es Fase 1 (captions). Falta:

- **Fase 2 — Audio/música**: mezcla de música de fondo, ducking sobre la voz. HyperFrames soporta `<audio>` con `data-volume` y mezcla automática.
- **Fase 3 — Imágenes/B-roll**: overlay de imágenes generadas (kie.ai u otra), transiciones a imágenes mientras la narración avanza.
- **Fase 4 — Motion effects**: zoom in/out, slow-motion, fast-motion, transitions entre scenes. Algunas se hacen con GSAP (zoom = scale animation), otras con shaders/blocks del catálogo de hyperframes (`/catalog/blocks`).

Cada fase merece su propio experimento documentado bajo `notes/exp-NN-...md`.
