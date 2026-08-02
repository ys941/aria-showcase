<div align="center">

# 🎙️ Aria

### An **AI-to-AI podcast**, hosted by little robots you design yourself — written, cast and voiced live, in your browser.

<p>
  <img src="https://img.shields.io/badge/three.js-WebGL-000000?style=for-the-badge&logo=three.js&logoColor=white" alt="three.js" />
  <img src="https://img.shields.io/badge/FastAPI-backend-009688?style=for-the-badge&logo=fastapi&logoColor=white" alt="FastAPI" />
  <img src="https://img.shields.io/badge/WebSocket-streaming-4353FF?style=for-the-badge" alt="WebSocket" />
  <img src="https://img.shields.io/badge/Vite-frontend-646CFF?style=for-the-badge&logo=vite&logoColor=white" alt="Vite" />
</p>
<p>
  <img src="https://img.shields.io/badge/Groq%20%C2%B7%20Gemini%20%C2%B7%20Modal-brain%20chain-FF6F00?style=for-the-badge" alt="LLM chain" />
  <img src="https://img.shields.io/badge/TTS-multi--engine%20fallback-8E44AD?style=for-the-badge" alt="TTS" />
  <img src="https://img.shields.io/badge/themes-15-2ea44f?style=for-the-badge" alt="15 themes" />
</p>

<em>Pick a topic. Two robots take the mic. Neither of them knew what they were going to say.</em>

</div>

> **About this repo.** This is a **public overview** of a private project. It showcases the architecture and the engineering behind it — the source code is kept private.

---

## What it is

**Aria** is a live show generator. You open a room, give it a topic, and a language model **writes the whole episode in one pass** — the cast, their personalities, and the script. Each line is then voiced by a text-to-speech engine and streamed to the browser, where two 3D robot hosts speak it with subtitles and a live level meter.

It is the kind of project that looks like a toy for about ninety seconds, until you realise how many things have to not break for a robot to say a sentence out loud on time.

---

## ✨ Highlights

### 🧠 A brain that refuses to be a single point of failure
- The script writer runs an **ordered provider chain** — a fast hosted model first, then a second hosted provider, then a self-hosted GPU endpoint, then a fourth — each tried in turn when the one above it errors, rate-limits or times out.
- A **runtime model picker** in the top bar lets you choose the brain for the next show; the choice persists and simply **reorders the chain**, putting your pick first and keeping the rest as fallback. No restart.
- The whole episode — around two dozen lines — is written in a **single call**, rather than dribbling out segment by segment. Rolling continuation exists but is gated off by default, because an unattended show that keeps calling a model forever is a bill, not a feature.

### 🗣️ A voice pipeline built around real quota limits
- Text-to-speech is a **fallback chain**, not a single provider. A hosted TTS with a generous-looking free tier turned out to cap at a few thousand tokens **per day** — barely a couple of lines — which was the real cause of a recurring "no voice" symptom. The self-hosted GPU engine is now the default, with the hosted one as an alternate, and a rate-limit response rolls over transparently so audio keeps flowing mid-episode.
- Provider quirks are handled rather than worked around: voice names are normalised, and long lines are **chunked under the input character cap and re-concatenated** at the audio level so a long sentence is still one continuous utterance.
- A local media toolchain is resolved robustly at runtime rather than assumed to be on the system path — a silent-room bug that came down to nothing more than a package manager not installing a shim.

### 🎭 Two modes
- **Podcast** — an improvised comedy show on any topic you type, with a cast the model invents to fit it.
- **🎓 English practice** — a completely separate writer (none of the comedy prompting is reused) with three fixed roles: a **coach**, a **learner** and a **guide**. The learner makes a small, level-appropriate mistake; the coach corrects it and explains why; the guide offers a natural alternative phrasing; the learner tries again. Scenario and difficulty are selectable.

### 🤖 The 3D front end
- A hand-built **three.js** single-page app — no UI framework — with a designable robot twin, a wizard for name, persona and voice, hologram glow, animated subtitles colour-coded per speaker, and a live audio meter.
- **15 themes** with a swatch picker showing real colour chips, plus ambient particle systems (petals, embers, drifting stars) that respect `prefers-reduced-motion`.
- A built-in **radio station** with a full track list that works standalone, with no cloud accounts configured at all.

### 🔌 Transport, rebuilt
The project originally streamed audio over a WebRTC SFU, which meant a container, a coturn-shaped port range, and a class of Windows-specific networking failures. That was **removed** in favour of a plain **WebSocket hub**: the server broadcasts a compact message protocol — cast, status, line (with audio inline), end — and the client queues, decodes and plays each line in order. No Docker required to run a show.

### 🐛 Three bugs worth keeping
- **The 24-hour sleep.** Some TTS engines emit WAV headers whose frame count is the int32 placeholder value. Reading it literally gave a clip duration of roughly 89,000 seconds, so the show scheduler dutifully slept for a day after line zero. Duration is now derived from the actual PCM byte length, with the sleep clamped.
- **Playing, but silent.** Every audio element was routed through a Web Audio analyser for the level meter. If the audio context was suspended, playback resolved successfully and produced **no sound** — and because playback "succeeded", the browser's unlock gate never appeared either. The analyser tap is now conditional on a running context, and the element otherwise plays natively.
- **The room you couldn't rejoin.** The show task outlived the last viewer and blocked reconnection. Disconnecting the final client now cancels the room task so the next join starts clean.

---

## 🧱 Tech stack

| Layer | Tech |
|---|---|
| Server | **FastAPI** (via a Gradio server host) · `uvicorn` · Python 3.13, `uv`-managed |
| Transport | **WebSocket** hub — cast / status / line / end protocol, audio inline |
| Frontend | **three.js** + **Vite**, hand-rolled SPA · 15 themes · particle systems |
| Brain | ordered fallback chain across hosted LLMs and a self-hosted GPU endpoint |
| Voice | multi-engine TTS chain with quota-aware rollover, chunked long-line synthesis |
| Audio | ffmpeg-based PCM decode with runtime binary resolution |
| Safety | profanity filtering on generated dialogue |
| Ops | one-click start/stop scripts · admin-token room control · settings persisted to disk |

---

## 🗺️ How an episode happens

```
   topic + template
         │
         ▼
   ┌───────────────────────────────┐
   │  write_script()  ── ONE call  │   brain chain:
   │  cast · personas · ~24 lines  │   hosted ▸ hosted ▸ self-hosted ▸ hosted
   └───────────────┬───────────────┘   (first failure falls through)
                   │
                   ▼            per line
   ┌───────────────────────────────┐
   │  tts_wav()                    │   engine chain, quota-aware rollover
   │  chunk ▸ synthesise ▸ concat  │   long lines split under the input cap
   └───────────────┬───────────────┘
                   │  decode → PCM → duration from real byte length
                   ▼
   ┌───────────────────────────────┐
   │  WebSocket hub                 │  {cast} {status} {line + audio} {end}
   └───────────────┬───────────────┘
                   ▼
   ┌───────────────────────────────┐
   │  three.js client               │  queue ▸ decode ▸ play ▸ subtitles
   │  robot twins · meter · themes  │  native playback if audio ctx suspended
   └───────────────────────────────┘
```

---

## 📌 Status

Verified end to end: a live generated show — model writes cast and script, TTS voices both hosts, audio streams and plays in the browser with subtitles. This public README is a portfolio overview — the implementation is private.

---

## 📬 Contact the developer

Interested in this or **any of Bhardwaj's projects** — a demo, a walkthrough, collaboration, or licensing?

### 📧 **[ys9410017064@gmail.com](mailto:ys9410017064@gmail.com)**

---

<div align="center">

<img src="assets/heart.gif" width="26" height="26" alt="beating heart" />

**Made with love by Yati Bhardwaj**

<sub><a href="https://github.com/ys941">github.com/ys941</a> · <a href="mailto:ys9410017064@gmail.com">ys9410017064@gmail.com</a></sub>

</div>
