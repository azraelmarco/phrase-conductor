# Phrase Conductor

A feasibility spike for **WAVE-FLOW**, an AI-conducted rhythm rail shooter.

It answers one question: *can a language model re-arrange music in real time from a player's performance without ever breaking sync?*

A local Llama model (Meta's Llama 3.2, served by Ollama) acts as an **AI Conductor**. Every phrase it reads the player's timing telemetry and returns a structured plan for an upcoming phrase: stem volumes, filter cutoff, tension, enemy density, color and section type. The browser applies that plan **sample-accurately on the bar**. If the model is late, the music never waits: a fallback plan is used and the event is logged.

> This is a standalone experiment in JavaScript and Web Audio. It is not the hackathon project and no code from it is reused in the Unreal Engine build.

## How it works

```
 player taps on the beat (SPACE)
            │
            ▼
 telemetry per phrase: hit rate, mean offset (early/late), jitter, extra taps
            │   at the start of phrase N
            ▼
 async request ──► Ollama /api/chat with a JSON-schema `format`
            │       (structured output)
            ▼
 validate + clamp every field (never trust the model)
            │
            ▼
 schedule plan on the AUDIO clock for the start of phrase N+1
            │
            ├── arrived in time ─► applied exactly on the downbeat
            └── late / error ───► previous plan held (fallback), logged
```

Key design decisions:

- **The audio clock is the single source of truth.** All stems start at the same `AudioContext` time and loop on an exact bar length (`loopEnd`), so they cannot drift.
- **The model never runs inside the timing loop.** It plans one phrase ahead. At 128 BPM an 8-bar phrase is 15 s, which is the model's latency budget.
- **Plans are scheduled, not applied "now".** Gain and filter changes use `AudioParam` automation at the exact phrase start time.
- **Structured output + validation.** The request carries a JSON schema; the response is still clamped to safe ranges before it touches audio.
- **Deadline with abort.** Each request has an `AbortController` that fires just before its phrase starts, so a slow model can only cause a logged fallback, never a glitch.
- **Session log export.** Every decision (telemetry in, plan out, latency, on time / late) is exportable as JSON: the same data a deterministic replay would need.

## Run it

Requirements: [Ollama](https://ollama.com) and any static web server (Python's built-in one is used below).

1. Make sure Ollama is running and has a model:
   ```
   ollama list
   ollama pull llama3.2:3b      # recommended: fast enough for 8-bar phrases
   ```
   `llama3.2-vision` (11B) also works but may be slower than a phrase on modest hardware. That is a useful test of the fallback path.

2. Serve this folder:
   ```
   cd wave-flow
   python -m http.server 8080
   ```

3. Open <http://localhost:8080> (not the file directly), click **Check Ollama**, then **Warm up model** (the first call loads it into memory).

4. Load stems (or click **Use built-in test stems**), press **Start**, and tap **SPACE** on every beat.

### If the browser cannot reach Ollama (CORS)

Ollama normally accepts requests from localhost pages. If you see a CORS error, allow the page's origin and restart Ollama:

- macOS / Linux: `OLLAMA_ORIGINS=http://localhost:8080 ollama serve`
- Windows (PowerShell): `setx OLLAMA_ORIGINS "http://localhost:8080"`, then quit and reopen Ollama.

## Using your own stems

- Export each stem as **WAV** (MP3 adds padding that breaks loops).
- All stems: **same BPM, same start point, exactly the "Loop bars" length** (e.g. 8 bars at 128 BPM = 15.000 s).
- Name files by role, for example `kick.wav`, `bass.wav`, `pad.wav`, `lead.wav`, `fx.wav`. The names are sent to the model, so clear names help it reason ("keep the kick loud as a timing reference").
- The stem list warns you if a file's length doesn't match the expected loop length.

## Tap calibration

Tap timing is compensated by the browser's reported output latency. If your hits consistently read early or late with Bluetooth or USB audio, adjust **Extra tap latency compensation (ms)**.

## Game layer (v3–v6)

To test the Conductor under real-time action, the prototype adds a minimal Rez-style game loop:

- **Enemies persist across phrases.** The Conductor's `enemy_density` (0–40) sets how many should be on screen; new ones arrive on the beat as mirrored pairs, faster at high tension.
- **Four enemy types** (cyan drone, magenta spiker, orange heavy, gold elite) with distinct shapes. The mix shifts with tension.
- **Phrase-end clear:** the player's accuracy decides the share of enemies destroyed, oldest first, as a particle chain reaction over about one beat. Survivors carry over.
- **Closed loop:** the number of enemies carried over is sent back to the model, which is told to ease off when pressure builds.
- **Visuals:** a two-hue gradient background mapped from section and tension (blue intro, teal breakdown, indigo→magenta build, red→orange drop), with 1.6 s eased transitions that start on the downbeat.
- **Opening at minimum tension:** kick and pad only, dark filter, no enemies.

## Results

| Version | Model | Plans on time | Median latency (15 s budget) | Notes |
|---|---|---|---|---|
| v1: model sees its previous plan | llama3.2:3b | 6/6 | ~4.0 s | Anchored on the previous plan and escalated even when the player stopped. |
| v2: player state + guardrails | llama3.2:3b | 10/10 | 6.5 s | Correct section in 10/10 phrases; guardrails adjusted 5/10 plans. |
| v6: full game loop, density 0–40 | llama3.2:3b | 10/10 | 5.5 s | 20–25 enemies cleared per strong phrase; guardrails adjusted 9/10 plans, including 3 section overrides. |

### Findings

- **Timing is solved by architecture, not model speed.** Planning one phrase ahead on the audio clock gave 26/26 on-time plans across the three logged sessions, with latencies of 3.7–7.4 s.
- **Splitting responsibilities fixed the decisions.** Code classifies the player; the model arranges; guardrails protect game feel. Without this split, the 3B model ignored the player.
- **The 3B model's mixing judgment is the limit.** As the parameter space grew (density 0–40, carried-over pressure), it picked breakdowns for steady players and sometimes muted the kick. Guardrails caught every case, but they are doing more of the work than they should.
- **Next steps:** run the same loop on Muse Spark; build a fine-tuning dataset from these logs (telemetry in, guardrail-corrected plan out) for a small on-device model such as Muse Glimmer; then port the Conductor into Unreal Engine (Quartz + MetaSounds).

`sample-session.json` is a full exported session from the v6 run.

## Demo video

[Watch the demo](https://drive.google.com/file/d/1MQc3wic6ioPfHDWM93pS7sWinMpCUyyx/view?usp=sharing)
