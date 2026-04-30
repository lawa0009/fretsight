# FretSight — Coding Rules & Documentation
> Living reference document. Update this file whenever a new capability tier is reached.
> Store in the root of the GitHub repo alongside `index.html`.

---

## 1. Version Nomenclature

Every deployment has a version string in the format:

```
DETECTION . AUDIO . GUIDANCE . PLATFORM . HARDWARE
```

**Reading the version — current example:**

```
1 . 0 . 2 . 0 . 0
│   │   │   │   │
│   │   │   │   └── HARDWARE  — 0 = no piezo clip yet
│   │   │   └────── PLATFORM  — 0 = no accounts/payments yet
│   │   └────────── GUIDANCE  — 2 = fretboard + finger animation
│   └────────────── AUDIO     — 0 = no audio interface yet
└────────────────── DETECTION — 1 = autocorrelation only (Phase 1)
```

Update this diagram each time a digit changes so it always reflects the current version.

Set this manually at the top of the `<script>` block in `index.html` before every push:

```javascript
// ── Deployment version — update this string each time you push ───────────
const APP_VERSION = '1.0.2.0.0';
```

The version appears in two places automatically:
- The **footer** of the app (subtle, small text)
- Every **detection log entry** (for cross-deployment analysis)

---

## 2. Version Digits Explained

### Position 1 — DETECTION
Tracks the quality and sophistication of the note recognition algorithm.

| Level | Description |
|-------|-------------|
| 1 | Autocorrelation only (baseline, Phase 1) |
| 2 | Autocorrelation + HPS consensus, stability gate, dynamic stability window per string |
| 3 | YIN algorithm added (planned) |
| 4 | ML-based pitch detection (future) |

**Increments when:** algorithm changes — new detection method, significant threshold tuning confirmed to improve accuracy across all 6 strings.

---

### Position 2 — AUDIO
Tracks audio input/output capability and device integration.

| Level | Description |
|-------|-------------|
| 0 | Built-in mic only, no device selection |
| 1 | Input device selector dropdown (choose mic vs audio interface) |
| 2 | Audio interface confirmed working (iRig HD 2 / Focusrite Scarlett) |
| 3 | FluidSynth server-side audio rendering (Phase 5c) |

**Increments when:** a new audio input source is supported or audio output quality meaningfully improves.

---

### Position 3 — GUIDANCE
Tracks the quality and sophistication of visual fretboard guidance.

| Level | Description |
|-------|-------------|
| 1 | Basic fretboard display, note position shown |
| 2 | Photorealistic SVG fretboard + headstock + cartoon finger animation (current) |
| 3 | Full 3D hand model via Three.js — all 4 fingers, naturalistic curl (Phase 5a) |
| 4 | MediaPipe live finger tracking + ghost hand guide overlay (Phase 5b) |

**Note:** The cartoon finger animation at level 2 is functional but not final — significant visual improvements are planned before level 3. Level 3 requires Three.js 3D hand, not just animation polish.

**Increments when:** a qualitatively new guidance capability is added, not just visual polish.

---

### Position 4 — PLATFORM
Tracks backend infrastructure, accounts, and distribution.

| Level | Description |
|-------|-------------|
| 0 | No backend — static HTML file only (current) |
| 1 | Supabase + FastAPI — user accounts and melody saving (Phase 2) |
| 2 | Stripe freemium — subscription payments live (Phase 3) |
| 3 | Mobile app via Capacitor — iOS and Android (Phase 4) |

**Increments when:** a new infrastructure layer goes live for real users.

---

### Position 5 — HARDWARE
Tracks physical accessory development and integration.

| Level | Description |
|-------|-------------|
| 0 | No hardware — software only (current) |
| 1 | ESP32 + piezo prototype — Bluetooth detection working on bench |
| 2 | Piezo clip integrated into app via Web Bluetooth API |
| 3 | Retail hardware — final enclosure, production firmware (Phase 6) |

**Increments when:** a hardware milestone is confirmed working end-to-end, not just ordered or assembled.

---

## 3. Version History

| Version | Date | Description |
|---------|------|-------------|
| `1.0.2.0.0` | Apr 2026 | HPS + autocorrelation consensus, stability gate, 8192 buffer, dynamic stability window, detection log with session tracking, live frequency display, deployment versioning |

> Add a row here every time you push a new version.

---

## 4. Version Examples — Future Milestones

```
1.0.2.0.0  →  current
1.1.2.0.0  →  input device selector added
1.2.2.0.0  →  audio interface confirmed working
2.2.2.0.0  →  YIN algorithm added and confirmed
2.2.3.0.0  →  Three.js 3D hand model (Phase 5a)
2.2.3.1.0  →  accounts + melody saving live (Phase 2)
2.2.3.2.0  →  Stripe payments live (Phase 3)
2.2.3.3.0  →  mobile app on App Store (Phase 4)
2.2.4.3.0  →  MediaPipe live tracking (Phase 5b)
2.3.4.3.1  →  ESP32 piezo prototype working
2.3.4.3.2  →  piezo clip integrated into app
2.3.4.3.3  →  retail hardware shipping
```

---

## 5. Detection Log Format

Every recorded note is appended to an in-memory log, downloadable as a `.txt` file.

**Log entry format:**
```
timestamp | version | session | note (midi) | AC=freq | HPS=freq | status
```

**Example:**
```
2026-04-29T14:32:05.123Z | v1.0.2.0.0 | session 1 | E2 (midi 40) | AC=82.41Hz | HPS=82.38Hz | AGREE
2026-04-29T14:32:06.441Z | v1.0.2.0.0 | session 1 | G#2 (midi 44) | AC=207.65Hz | HPS=103.83Hz | DISAGREE
2026-04-29T14:35:14.001Z | v1.0.2.0.0 | session 2 | A2 (midi 45) | AC=— | HPS=110.02Hz | SINGLE
```

**Status values:**
| Status | Meaning |
|--------|---------|
| `AGREE` | Both autocorrelation and HPS agree within 1 semitone — highest confidence |
| `DISAGREE` | Both fired but disagree — algorithm applied fallback rule |
| `SINGLE` | Only one algorithm detected a frequency — lower confidence |

**Sessions:** Pressing **Clear** increments the session counter. The log keeps accumulating across sessions and deployments until the page is refreshed.

---

## 6. Algorithm — Current Detection Pipeline

```
Mic input (8192 sample buffer, 44100Hz)
    │
    ├── Autocorrelation (autoCorrelate)
    │     RMS threshold: 0.02
    │     Parabolic interpolation for sub-sample accuracy
    │     Returns frequency or -1
    │
    ├── HPS — Harmonic Product Spectrum (hpsDetect)
    │     Uses browser AnalyserNode FFT (free, no compute cost)
    │     Multiplies magnitude spectrum at 1x–5x harmonics
    │     Best for low strings where harmonics confuse autocorrelation
    │     Returns frequency or -1
    │
    └── Consensus (detectPitch)
          Both agree (≤1 semitone) → average frequency, status=AGREE
          Both fire, disagree → HPS for midi<45, AC for midi≥45, status=DISAGREE
          Only one fires → accept it, status=SINGLE
          Neither fires → null, no note recorded

Stability gate (recLoop)
    Candidate must hold same MIDI note for:
      60ms  — low strings (midi < 45, E2/F2/F#2/G2/G#2/A2)
      35ms  — high E string (midi > 64)
      50ms  — all other strings
    Debounce: 400ms minimum between accepted notes
```

---

## 7. Known Issues

| Issue | Status | Notes |
|-------|--------|-------|
| High E string missed notes | Partial | Stability window at 35ms helps; HPS consensus improves accuracy |
| Low E G#/A/A# octave errors | Partial | HPS targets this specifically; monitor via DISAGREE log entries |
| Electric guitar via built-in mic | Known | Needs audio interface (iRig HD 2) or piezo clip (Phase 6) |
| Note detection lag in Safari | Partial | 8192 buffer + browser FFT (no manual DFT) — lag eliminated |

---

## 8. Tech Stack Reference

| Layer | Technology | Status |
|-------|-----------|--------|
| Frontend | HTML / JS / CSS (single file) | Live |
| Audio | Tone.js + Web Audio API | Live |
| Pitch detection | Autocorrelation + HPS | Live |
| Hosting | GitHub Pages | Live |
| Backend | FastAPI (Python) | Phase 2 |
| Database | Supabase | Phase 2 |
| Payments | Stripe | Phase 3 |
| Mobile | Capacitor | Phase 4 |
| 3D hand | Three.js | Phase 5a |
| Hand tracking | MediaPipe | Phase 5b |
| Server audio | FluidSynth | Phase 5c |
| Hardware | ESP32 + piezo | Phase 6 |

---

## 9. Version Update Rules — How & When Claude Updates the Version

### Trigger phrase
Say either of these in the chat to trigger a version update:
> *"confirmed working"*
> *"confirmed, update the version"*

Claude will then automatically generate updated `index.html` and `FRETSIGHT_CODING_RULES.md` files ready to push — no manual editing needed.

### What Claude does on confirmation
1. Determines which digit to increment based on what was just built and confirmed
2. Updates `APP_VERSION` in `index.html`
3. Updates the version anatomy diagram in this document to match the new version
4. Adds a row to the Version History table (Section 3) with date and description
5. Updates the Known Issues table — marks fixed issues, adds any new ones introduced
6. Delivers both files ready to push in one response

### Rules for incrementing
- **Increment the digit that matches what changed** — Detection for algorithm work, Audio for I/O work, Guidance for visual work, Platform for backend/payments/mobile, Hardware for piezo/ESP32
- **Leave all other digits untouched** — digits are independent, not cascading
- **Never reset a digit to 0** unless a capability is explicitly retired (rare)
- **Only increment after you confirm** — not when we build it, not when we push it, only when you test it and say it works
- **One increment per confirmation** — if multiple things were built in one session, increment each relevant digit once

### Example progression
```
1.0.2.0.0  →  you confirm HPS detection working    →  2.0.2.0.0
2.0.2.0.0  →  you confirm input device selector    →  2.1.2.0.0
2.1.2.0.0  →  you confirm 3D hand model working    →  2.1.3.0.0
2.1.3.0.0  →  you confirm accounts live            →  2.1.3.1.0
```

---

## 10. Deployment Checklist

Before every `git push`:

- [ ] Say "confirmed working" in chat — Claude updates `APP_VERSION` and docs automatically
- [ ] Drop the updated `index.html` and `FRETSIGHT_CODING_RULES.md` into `Documents/fretsight`
- [ ] Test all 6 strings with mic before pushing
- [ ] Commit message describes what changed (e.g. `feat: YIN algorithm`)

```bash
cd ~/Documents/fretsight
git add index.html FRETSIGHT_CODING_RULES.md
git commit -m "your message here"
git push
```

---

*FretSight · See it · Hear it · Learn it · github.com/lawa0009/fretsight*
