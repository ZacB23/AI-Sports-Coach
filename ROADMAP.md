# RowCoach — Roadmap and Build Guide

*Draft v0.1 · 5 Nov 2026 · Zac & Aluinn · working name, rename freely*

**Our constraints (decided):** ~5 hours a week each for 10 weeks (about 45–50 hours each) · small boats first, no erg stage · existing footage only to start · portfolio project · both on Apple Silicon Macs · everything runs locally, free, no API keys needed.

**How to read this:** sections 1–3 are the "what", 4–6 are the "how and when", 7–10 are "how we know it works and stay out of trouble", 11–13 are "how we work". The first Claude Code prompts are at the very end.

---

## 1. The idea and who it's for

**One paragraph.** Most club, school and university crews can't afford boat telemetry, so "are we together at the catch?" is judged by eye from a launch. RowCoach takes a phone video of a crew filmed side-on and gives back the stroke rate, each seat's catch and finish timing relative to stroke ("seat 3 catches 45 ms late, on 14 of 18 strokes"), drive:recovery ratio and how steady it is, the coach's spoken comments pinned to the moment they were said, and short plain-English notes per seat that link to the clip that proves them. Every number comes with an honest error bar.

### Who needs what

| Group | What they need | Why they're hard or easy |
|---|---|---|
| **University and club coaches** | Fast answer after an outing, on a phone; one or two points per seat they can act on next session; numbers they trust | **Easiest first target.** Adults, so consent is simple. You have access through the uni club. Small boats are common in training. |
| School coaches | The same, plus simple sharing with athletes | Junior crews mean parental consent and safeguarding rules. Do later. |
| Masters crews | Objective feedback when they have no full-time coach | Adults, keen, often self-coached. Good second target. |
| Coxes and rowers | Reviewing their own pieces; "was it me?" | Same tool, read-only view. Comes for free once coaches have it. |
| Individuals on the erg (later) | Technique sequencing (legs, body, arms) | Big group, but a different product: no crew timing. Park it. |

**Build for first:** a university or club coach reviewing an **adult pair or four** from existing footage. Everything in the MVP is judged by whether that coach would say "yes, that's what I saw".

---

## 2. The MVP, cut hard

**The MVP is one sentence:** on a real 30–60 s clip of an adult pair or four filmed side-on, the app plays the video with skeletons and seat numbers drawn on, and shows a chart of each seat's catch timing versus stroke for every stroke, with the measurement error we have checked against hand labels.

That's it. Stroke rate and ratio come almost free from the same catch and finish times, so they're in. Transcription and LLM feedback are **after** the MVP (weeks 6–8). Blade tracking is a stretch goal we probably won't reach in 10 weeks, and that's fine.

### Erg first or boats first?

You chose small boats, and I'm happy with that, with three small changes:

1. **Start with a pair or double (2 rowers), then a four.** Two people side-on is the simplest real crew. An eight is week 10+ at the earliest.
2. **Keep one erg clip as a sanity test.** Film yourself on an erg for 60 s, side-on, at 60 fps. No consent needed, and if the boat clips misbehave you can check whether the catch detector or the boat footage is the problem in ten minutes.
3. **Audit the existing footage before building anything (task Z3).** Old coaching video is often filmed at an angle, at 30 fps, panning, with the crew half out of frame. We need at least 6 usable clips. If we can't find them, film new ones (adults, with consent).

### What's in and out

| In the MVP (weeks 1–5) | After the MVP (weeks 6–9) | Not in this 10-week plan |
|---|---|---|
| Upload/run on a clip | Upload from phone, job status | Eights |
| Pose + seat numbers | Coach audio transcript pinned to time | Blade tracking (SAM 2) |
| Catch/finish per seat | Rule-based feedback per seat | Erg technique mode |
| Rate, offsets, ratio | LLM rewrite of feedback, number-checked | Accounts, sharing, hosting |
| Player with overlays + timing chart | Clips (jump-to and loop) | Junior crews |
| Evaluation against hand labels | Coach check of the verdicts | |

---

## 3. Architecture

### The flow

```
 phone video (.mov/.mp4)
        │
        ▼
┌──────────────────────── server/ (Aluinn, Python + FastAPI) ────────────────────────┐
│ 1 ingest      read every frame's real timestamp (PTS), size, rotation, audio       │
│ 2 pose        person boxes + 17 body keypoints per person per frame                │
│ 3 seats       follow each person across frames, order them along the boat,        │
│               drop the cox, number bow=1 … stroke=N                                │
│ 4 phases      per seat: "handle trace" → catches (minimum) and finishes (maximum)  │
│ 5 metrics     rate, catch/finish offsets vs stroke, drive, recovery, ratio         │
│ 6 transcript  Whisper on the audio → timed segments, seat mentions ("3 is …")      │
│ 7 feedback    rules first → optional LLM rewrite → every number checked            │
│                                                                                     │
│ writes:  data/<id>/analysis.json   data/<id>/landmarks.json   data/<id>/video.mp4  │
└──────────────────────────────────┬──────────────────────────────────────────────────┘
                                   │  HTTP (same address: server also serves the built web app)
                                   ▼
┌──────────────────────── web/ (Zac, TypeScript + Vite) ─────────────────────────────┐
│ upload · <video> + <canvas> overlay (skeletons, seat labels) · timing charts ·     │
│ transcript pins on the timeline · feedback per seat with "show me" clips ·         │
│ label mode (for hand-labelling test clips)                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘
        ▲
        └── until the server is ready: web loads shared/examples/*.json (fake data)
```

Two things that differ from last time and matter a lot:

- **Time comes from each frame's own timestamp, never from `frame_number / fps`.** iPhones record at a variable frame rate. If we assume a fixed rate, timings drift by tens of milliseconds over a minute, which is the same size as the effect we're measuring.
- **Every measurement is relative inside one rower's body** (hands relative to feet), so it doesn't matter if the camera pans, the boat moves across the frame or the launch zooms.

### How a catch is found (the core idea, in plain English)

For each rower, each frame, measure how far the hands are from the feet along the boat, divided by that rower's torso length (so it doesn't depend on how big they look on screen). Call this the **handle trace**. It goes down as the hands reach towards the stern and up as they draw in. The catch is the bottom of each dip (hands furthest forward, legs compressed); the finish is the top of each peak (hands into the body, legs flat). Smooth the trace a little, find each low point, then fit a small curve through the three frames around it to estimate the moment *between* frames. Zac writes the exact definition (task Z2); Aluinn implements it (task A4).

This is a body-based catch, a few tens of ms before the blade locks on. That's fine: we report each seat **relative to stroke**, and if every seat has the same small bias it cancels out. We check that in evaluation.

### Folders

```
rowcoach/
├── CLAUDE.md  ROADMAP.md  BUILD_LOG.md  THIRD_PARTY.md  README.md
├── .gitignore                  # .env, footage/, data/, web/public/dev/, node_modules, .venv
├── shared/                     # JOINT — changes need a PR both of you approve
│   ├── CONTRACT.md             # conventions in plain English (copy of the list below)
│   ├── analysis.schema.json    # the source of truth for analysis.json
│   ├── landmarks.schema.json
│   └── examples/
│       ├── analysis.example.json
│       └── landmarks.example.json
├── server/                     # Aluinn — branch aluinn/server
│   ├── pyproject.toml
│   ├── rowcoach/
│   │   ├── main.py             # FastAPI app, endpoints
│   │   ├── cli.py              # `python -m rowcoach run footage/clip.mov`
│   │   ├── ingest.py  pose.py  seats.py  phases.py  metrics.py
│   │   ├── transcribe.py  feedback.py  store.py
│   └── tests/
├── web/                        # Zac — branch zac/web
│   ├── src/  (main.ts, api.ts, config.ts, player/, charts/, feedback/, label/)
│   ├── scripts/                # make-fake.ts, type generation
│   └── public/dev/             # git-ignored: drop a test clip here for local work
├── eval/                       # JOINT
│   ├── clips.json              # clip list: id, fps, boat, seats, consent status (no video)
│   ├── labels/<clip_id>.json   # hand-labelled catches and finishes (numbers only)
│   ├── evaluate.py
│   └── RESULTS.md              # newest first
├── docs/
│   ├── metric-definitions.md   # Zac (task Z2)
│   └── filming-guide.md        # Zac (copy of section 8, kept up to date)
├── footage/                    # git-ignored. Raw video lives here and only here.
└── data/                       # git-ignored. Server outputs.
```

### The shared contract: `analysis.json` (draft v0.1.0)

**Conventions (these go in `shared/CONTRACT.md` word for word):**

| Thing | Rule |
|---|---|
| Time | Milliseconds (`_ms`) from the start of the video file, as floats rounded to 0.1. Taken from each frame's presentation timestamp. |
| Image coordinates | Normalised 0–1 on the upright frame (rotation already applied). Origin top-left, x to the right, y **down**. |
| Seats | `1` = bow … `N` = stroke. The cox is never a seat. Stroke seat = `num_rowers`. |
| Offsets | `seat time − stroke seat time` for the same stroke. **Positive = late**, negative = early. Stroke seat is always 0. |
| Stroke rate | `60000 / (next stroke-seat catch − this stroke-seat catch)`, in strokes per minute. |
| Drive / recovery | Drive = catch → finish. Recovery = finish → next catch. Both in ms. |
| Ratio | `recovery_ms / drive_ms`. So 2.0 means the usual "1 : 2". |
| Consistency | Standard deviation in ms across valid strokes (`_sd_ms`). |
| Confidence | 0–1 numbers from the models; `"high" / "medium" / "low"` words only in feedback. |
| Missing | `null`, never `0`. A stroke with any seat missing has `valid: false` and is left out of averages. |
| Keypoints | COCO-17 names and order (listed in `landmarks.json`). Both MediaPipe and RTMPose output can be mapped to this. |
| Versions | `contract_version` follows major.minor.patch. Any change = PR touching `shared/`, approved by both, version bumped. |

**Example (a pair, shortened with `…`):**

```json
{
  "contract_version": "0.1.0",
  "analysis_id": "a_20261116_pair01",
  "created_at": "2026-11-16T18:04:00Z",
  "status": "complete",
  "video": {
    "filename": "pair01.mov",
    "playback_url": "/analyses/a_20261116_pair01/video",
    "duration_ms": 41233.3,
    "width_px": 1920,
    "height_px": 1080,
    "rotation_deg": 0,
    "nominal_fps": 30.0,
    "measured_fps": 29.97,
    "variable_frame_rate": true,
    "frame_count": 1236,
    "has_audio": true
  },
  "boat": {
    "type": "2-",
    "num_rowers": 2,
    "has_cox": false,
    "travel_direction": "right",
    "direction_source": "auto"
  },
  "seats": [
    { "seat": 1, "label": "Bow",    "track_id": 3, "coverage": 0.97, "mean_confidence": 0.84 },
    { "seat": 2, "label": "Stroke", "track_id": 1, "coverage": 0.99, "mean_confidence": 0.88 }
  ],
  "landmarks": {
    "file": "landmarks.json",
    "keypoint_set": "coco17",
    "model": "rtmpose-m via rtmlib"
  },
  "traces": {
    "what": "handle trace: hands-to-feet distance along the boat, in torso lengths",
    "t_ms": [0.0, 33.4, 66.7, "…"],
    "by_seat": { "1": [1.42, 1.38, 1.31, "…"], "2": [1.45, 1.40, 1.33, "…"] }
  },
  "strokes": [
    {
      "index": 0,
      "valid": true,
      "stroke_catch_ms": 1203.4,
      "rate_spm": 31.2,
      "seats": [
        { "seat": 1, "catch_ms": 1250.1, "finish_ms": 2001.7, "catch_offset_ms": 46.7,
          "finish_offset_ms": 14.3, "drive_ms": 751.6, "recovery_ms": 1171.0, "confidence": 0.86 },
        { "seat": 2, "catch_ms": 1203.4, "finish_ms": 1987.4, "catch_offset_ms": 0.0,
          "finish_offset_ms": 0.0, "drive_ms": 784.0, "recovery_ms": 1138.6, "confidence": 0.91 }
      ]
    }
  ],
  "metrics": {
    "piece": {
      "start_ms": 1203.4, "end_ms": 39870.0, "strokes_total": 20, "strokes_valid": 18,
      "rate_spm_mean": 31.0, "rate_spm_min": 30.1, "rate_spm_max": 32.4,
      "timing_error_ms": 18.0
    },
    "by_seat": [
      { "seat": 1, "strokes_used": 18,
        "catch_offset_ms_median": 44.0, "catch_offset_sd_ms": 12.5, "late_stroke_share": 0.78,
        "finish_offset_ms_median": 11.0, "drive_ms_mean": 760.2, "drive_sd_ms": 31.0,
        "ratio_mean": 1.52, "ratio_sd": 0.08, "catch_verdict": "late" }
    ]
  },
  "transcript": {
    "model": "faster-whisper large-v3-turbo",
    "language": "en",
    "segments": [
      { "start_ms": 8120.0, "end_ms": 9600.0, "text": "Bow, you're late in, wait for stroke", "seats_mentioned": [1] }
    ]
  },
  "feedback": {
    "generator": "rules-v1",
    "items": [
      {
        "id": "f1",
        "seat": 1,
        "text": "Bow catches about 44 ms after stroke on 14 of 18 strokes. That's bigger than our measurement error (about 18 ms), so it's real.",
        "evidence": [
          { "metric": "catch_offset_ms_median", "value": 44.0 },
          { "metric": "late_stroke_share", "value": 0.78 },
          { "metric": "timing_error_ms", "value": 18.0 }
        ],
        "confidence": "high",
        "clips": [ { "start_ms": 7800.0, "end_ms": 11200.0, "label": "Strokes 4–5" } ]
      }
    ]
  },
  "warnings": [
    { "code": "low_fps", "message": "Filmed at 30 fps: each frame is 33 ms, so small timing differences are uncertain." }
  ]
}
```

**`landmarks.json`** is separate because it's big (every person, every frame). Each person's 17 keypoints are a flat list of `x, y, confidence` × 17 = 51 numbers, rounded to 3 decimals:

```json
{
  "contract_version": "0.1.0",
  "keypoint_set": "coco17",
  "keypoint_names": ["nose","left_eye","right_eye","left_ear","right_ear","left_shoulder","right_shoulder",
    "left_elbow","right_elbow","left_wrist","right_wrist","left_hip","right_hip","left_knee","right_knee",
    "left_ankle","right_ankle"],
  "frames": [
    { "i": 0, "t_ms": 0.0, "people": [ { "seat": 1, "kp": [0.412, 0.533, 0.91, "…51 numbers"] } ] }
  ]
}
```

**Endpoints (task A6):** `POST /analyses` (form fields: `video`, `boat_type`, optional `travel_direction`) → `{analysis_id}` · `GET /analyses/{id}` → `{status: queued|running|complete|failed, step, progress 0–1, error?}` · `GET /analyses/{id}/analysis.json` · `GET /analyses/{id}/landmarks.json` · `GET /analyses/{id}/video` · `DELETE /analyses/{id}` (deletes everything for that clip) · `GET /health`. In production the server also serves the built web app, so the web app always calls **relative** paths (`fetch('/analyses')`), with a Vite dev proxy to `localhost:8000`, same as last project.

**One source of truth for types:** `shared/analysis.schema.json` is the contract. The server checks its output against it in tests; the web app generates its TypeScript types from it (`npm run types`), so the two halves can't quietly drift apart.

---

## 4. Tech stack

Versions and licences checked 5 Nov 2026. Re-check before adding anything, and record it in `THIRD_PARTY.md`.

| Part | Choice | Why | Licence | Alternative |
|---|---|---|---|---|
| Video reading | **PyAV** | Gives each frame's real timestamp and rotation; no separate install | BSD-3 (bundles FFmpeg, LGPL) | `ffmpeg`/`ffprobe` via subprocess |
| Body pose | **RTMPose-m via `rtmlib`** | Top-down (finds each person, then their joints), so it copes better with several small people than MediaPipe; runs on ONNX on a Mac without the heavy MMPose stack | Apache-2.0 (code and checkpoints; some training datasets have their own terms) | **MediaPipe Pose Landmarker** (Apache-2.0), easiest, supports several people via `num_poses`; benchmark both in A2 |
| Pose (not using) | MMPose | Same models, but the project looks inactive (no PyPI release in the last year) and is heavy to install | Apache-2.0 | `rtmlib` above |
| Who is who across frames | **Order along the boat + simple matching** | Rowers don't overtake each other; sorting by position along the boat is enough most of the time | — | **ByteTrack** via Roboflow `trackers` (Apache-2.0; original ByteTrack is MIT) if IDs swap |
| Smoothing and peak finding | **SciPy** | Savitzky–Golay smoothing and `find_peaks` are exactly what phase detection needs | BSD-3 | NumPy by hand |
| Transcription | **faster-whisper**, model `large-v3-turbo` | Word timestamps, simple Python API, fine speed for 60 s clips (runs on CPU on a Mac) | MIT | `mlx-whisper` (MIT, faster on Apple Silicon) or `whisper.cpp` (MIT) |
| Feedback (step 1) | **Rules in Python** | Deterministic, testable, can't make numbers up | — | — |
| Feedback (step 2, optional) | **Local LLM via Ollama**: `qwen3.8:27b` if your Mac has 24 GB+ RAM, otherwise **Ministral 3** 8B or 14B | Free, offline, footage-adjacent data never leaves the Mac | Qwen3.8-27B Apache-2.0; Ministral 3 Apache-2.0; Ollama MIT | Hosted API (e.g. Claude Haiku 5.5): better wording, small cost, needs a key and sends transcript text off the Mac |
| Blades (stretch) | SAM 2 | Promptable video segmentation | Apache-2.0 | SAM 3: SAM License — royalty-free and commercial use allowed, but custom terms Meta can change; read it before relying on it |
| Server | **FastAPI + Uvicorn + Pydantic** | You used it last time; Pydantic validates the contract | MIT / BSD-3 / MIT | Flask |
| Python tooling | **uv** | One command to create the environment and install | MIT or Apache-2.0 | `venv` + `pip` |
| Contract checking | `jsonschema` (Python), `json-schema-to-typescript` (web) | Same schema drives both halves | MIT / MIT | Hand-written types |
| Web app | **Vite + TypeScript (strict), no framework** | Same as last time; small app | MIT / Apache-2.0 | Preact if the UI grows |
| Video + overlays | **`<video>` + `<canvas>`**, drawn with `requestVideoFrameCallback` | Draws on exactly the frame being shown; supported in Safari/iOS 15.4+ | Built into browsers | `requestAnimationFrame` + `currentTime` (less exact) |
| Charts | **Chart.js** | Easy, good docs, works on phones | MIT | uPlot (MIT), smaller and faster for long traces |

**LLM recommendation:** don't let the LLM be load-bearing. Ship rule-based feedback first (it's honest by construction). Then add a local model to rewrite the rule-based notes into friendlier coach-speak, with every number checked against `analysis.json` (task A8). Compare local vs a hosted model on 5 clips; for a free, offline portfolio project the local model is the right default, and the hosted one is a one-line config swap if the wording is clearly better.

**Mac memory check:** Qwen3.8-27B in Ollama is about an 18 GB download and needs a Mac with 24 GB or more. On 16 GB, use Ministral 3 8B.

---

## 5. Work split and tasks

Each person works in their own folder and branch and opens PRs into `main`. Never push to `main`. `shared/` and `eval/` change only through a PR the other person approves.

**Zac can build the whole web app against fake data.** `shared/examples/*.json` plus a script that generates fake landmarks for any test clip means nothing in `web/` waits on the server until Z6.

### Joint

| ID | Task | Done looks like |
|---|---|---|
| **J1** | Contract v0.1.0: copy section 3 into `shared/` as schema + examples + `CONTRACT.md` (one 60-min call, week 1) | Both example files validate against the schemas; merged to `main` |
| **J2** | Consent and footage permission (section 10), led by Zac | Written OK from the club for the clips we use; every clip in `eval/clips.json` has `consent: "yes"` |
| **J3** | Human-agreement check: both label the same clip independently | Report in `eval/RESULTS.md`: how far apart two people are, in ms. This is our floor |

### Zac — web app + rowing domain (`web/`, `docs/`, labels)

| ID | Task | Done looks like | Blocked by | Model | Est. |
|---|---|---|---|---|---|
| **Z1** | Web scaffold + player: plays a local clip, draws fake skeletons + seat labels in sync, phone layout | Fake stick figures move with the video on laptop and iPhone; scrubbing and pause stay in sync; `npx tsc --noEmit` clean | J1 | Sonnet 5.5 | 6 h |
| **Z2** | Metric definitions (`docs/metric-definitions.md`): catch, finish, the handle trace, which wrist/ankle to use, seat numbering, "late", ratio, consistency, thresholds, when to say "can't tell" | Aluinn has read it and can code it without asking you anything; one worked example with numbers | — | Opus 5.5 | 3 h |
| **Z3** | Footage audit: go through existing videos, pick 6–10 usable clips (checklist in section 8), record fps, angle, boat, seats, consent status in `eval/clips.json` | ≥ 6 clips marked usable, including ≥ 3 pairs/doubles; clips copied into `footage/` | — | — | 3 h |
| **Z4** | Label mode in the player: step frame by frame, pick a seat, press C (catch) / F (finish), export `eval/labels/<clip>.json`; then label 3 clips × all seats × 10 strokes | 3 label files committed; labelling one clip takes < 30 min | Z1, Z3 | Sonnet 5.5 | 6 h + 4 h labelling |
| **Z5** | Timing charts: per-seat catch offsets per stroke (dots, with a shaded "can't tell" band = ± timing error), rate over the piece, drive/recovery; click a stroke → video jumps there | Charts render from the example file and from a real `analysis.json`; usable on a phone | Z1 | Sonnet 5.5 | 6 h |
| **Z6** | Upload + real server: `api.ts` with `USE_FAKE_SERVER` switch, upload form (boat type, direction), progress polling, error messages | Upload a clip from your phone → see real results; switch back to fake with one line | A6 | Sonnet 5.5 | 5 h |
| **Z7** | Feedback view: one card per seat, evidence numbers shown, "show me" plays the clip range on loop; transcript pins on the timeline | Every card's numbers visibly match the charts; tapping a pin seeks to that comment | Z5 (real data: A7, A8) | Sonnet 5.5 | 6 h |
| **Z8** | Coach check: go through 3 analysed clips with a coach (or yourself as a coach), note agree/disagree for each verdict | Write-up in `eval/RESULTS.md`: "coach agreed with X of Y calls", plus what to fix | A4, Z7 | — | 4 h |
| Z9 | Polish (cut first if short of time): loading states, empty states, blurred-face demo clip, README screenshots | Demo you'd show an interviewer | Z8 | Sonnet 5.5 | 3 h |

### Aluinn — server + vision pipeline (`server/`, `eval/evaluate.py`)

| ID | Task | Done looks like | Blocked by | Model | Est. |
|---|---|---|---|---|---|
| **A1** | Server scaffold + ingest + CLI: FastAPI `/health`; `python -m rowcoach probe <clip>` prints duration, size, rotation, nominal vs measured fps, variable frame rate yes/no, audio yes/no, and saves every frame timestamp | Works on 3 real clips; correctly reports variable frame rate; `pytest` passes | — | Sonnet 5.5 | 5 h |
| **A2** | Pose: run RTMPose (rtmlib) and MediaPipe on 2 clips, pick one, write `landmarks.json` to the contract | Comparison note in BUILD_LOG (speed, missed people); `landmarks.json` validates; Zac's player draws it in the right place | A1, J1, Z3 | Opus 5.5 for the comparison, Sonnet for the rest | 6 h |
| **A3** | Seats: keep identities across frames, order rowers along the boat, ignore the cox and bystanders, guess bow direction (motion, and which way faces point), allow override | Right seat number on ≥ 90 % of frames on 3 clips, checked by eye in Zac's player | A2 | Opus 5.5 | 6 h |
| **A4** | Phases + metrics exactly as in Z2: handle trace, smoothing, catches/finishes with between-frame estimate, stroke matching, offsets, rate, ratio, consistency, verdicts | `analysis.json` validates; unit tests on a synthetic sine-wave "rower" with a known 50 ms delay recover 50 ± 2 ms | A3, Z2 | Opus 5.5 | 8 h |
| **A5** | `eval/evaluate.py` (section 7): compare detections to Zac's labels, write a table to `eval/RESULTS.md` | One command prints errors for all labelled clips | A4, Z4 | Sonnet 5.5 | 3 h |
| **A6** | Endpoints + jobs: upload, background processing with status/progress, serve files and video (transcode to H.264 MP4 if the browser can't play the original), `DELETE`, serve built web app | Zac's Z6 works against it from a phone on the same Wi-Fi | A4 | Sonnet 5.5 | 5 h |
| **A7** | Transcription: faster-whisper with word timestamps → segments; tag seat mentions ("bow", "two", "3 seat", "stroke") | Segments line up within ~0.5 s by ear on 3 clips; seat tags right on obvious cases | A1 | Sonnet 5.5 | 3 h |
| **A8** | Feedback: rules first (from Z2 thresholds), then optional LLM rewrite via Ollama; checker rejects any text whose numbers aren't in its `evidence` or don't match `metrics` | Checker passes 100 % on 5 clips; a short local-vs-hosted comparison in BUILD_LOG | A4, A7, Z2 | Opus 5.5 for the prompt + checker | 6 h |
| A9 | Stretch: blade entry with SAM 2 on one clip | Only if every gate up to week 6 passed | A4 | Opus 5.5 | — |

### What blocks what

```
J1 ──► Z1 ──► Z4 (label mode) ──► labels ──► A5 (evaluate)
 │      └──► Z5 (charts) ──► Z7 (feedback view) ──► Z8 (coach check)
 │
 └──► A2 (pose) ◄── A1 (ingest)      Z3 (footage audit) ──► A2, Z4
        └──► A3 (seats) ──► A4 (phases+metrics) ◄── Z2 (definitions)
                               ├──► A5
                               ├──► A6 (endpoints) ──► Z6 (upload)
                               └──► A8 (feedback) ◄── A7 (transcript)
```

**Critical path:** J1 → A1 → A2 → A3 → A4 → A5. Zac's Z2 and Z3 must land in week 1 so they don't hold it up.

---

## 6. Milestones (week by week, with gates)

Weeks start on Mondays. A gate is a check you both look at together on the Sunday. If a gate fails, fix it before moving on; if it fails twice, cut scope rather than slip (see the fallback in brackets).

| Week | Dates | Zac | Aluinn | Gate |
|---|---|---|---|---|
| 1 | 9 Nov | J1 (joint), **Z2**, **Z3**, start Z1 | J1, **A1** | ≥ 6 usable clips listed with fps and consent status; contract merged; `probe` works on one clip |
| 2 | 16 Nov | **Z1** done | **A2** | Real skeletons drawn on a real clip in Zac's player, no visible lag on laptop or iPhone |
| 3 | 23 Nov | **Z4**, label 2 clips | **A3** | Correct seat numbers on ≥ 90 % of frames on 3 clips *(fallback: you tap to set bow direction in the UI)* |
| 4 | 30 Nov | Label 3rd clip, start **Z5**; J3 | **A4**, **A5** | Stroke rate within ±1 spm of a hand count on 3 clips; first error table exists |
| 5 | 7 Dec | **Z5** done | Tune A4 against labels | **MVP gate:** median error in seat-vs-stroke catch offset ≤ 1 frame (≤ 33 ms at 30 fps, ≤ 17 ms at 60 fps) on the labelled clips, shown in the chart *(fallback: report only rate + ratio, which need less precision, and flag offsets as "rough")* |
| 6 | 14 Dec | **Z6** | **A6**, **A7** | Upload from phone → results back → charts, end to end |
| 7 | 21 Dec | Christmas: buffer, catch up | Buffer | Nothing new. Fix what's red |
| 8 | 28 Dec | **Z7** | **A8** | Feedback for every seat; number checker passes 100 % |
| 9 | 4 Jan | **Z8** coach check | Fix what the coach check finds | Coach agrees with most "late"/"together" calls; disagreements are explained |
| 10 | 11 Jan | Z9, README, demo | README, tidy | Demo video (faces blurred, consent given); decide what's next: fours → eights, erg mode, or blades |

---

## 7. Testing and accuracy

**The question:** when the app says "seat 1 is 44 ms late", how wrong could that be?

### Hand labels (Zac, task Z4)

1. Pick 3 clips (at least 2 pairs). For each seat, label 10 consecutive strokes.
2. Use the **same body-based definition** as the app (from Z2): the catch frame is the frame where the hands stop moving towards the stern and start coming back. Label finishes the same way. Optionally also note the frame where the blade first touches the water, for later.
3. Step one frame at a time in label mode. Don't look at the app's answer first.
4. **Both of you label one clip independently (J3).** The difference between you is the best accuracy we can hope to prove.

Label file `eval/labels/<clip_id>.json`:

```json
{ "clip_id": "pair01", "labeller": "zac", "fps": 30, "boat": "2-",
  "events": [ { "seat": 2, "type": "catch", "t_ms": 1201.2 }, { "seat": 1, "type": "catch", "t_ms": 1234.6 } ] }
```

### What we measure

| Measure | Meaning | Target by week 5 |
|---|---|---|
| Stroke rate error | App vs hand count, spm | ≤ 1 spm |
| Catch detection | % of labelled catches the app found within ±150 ms; extra catches it invented | ≥ 95 % found, ≤ 1 extra per clip |
| Absolute catch error | median and 90th-percentile \|app − label\| in ms | median ≤ 1 frame |
| **Relative offset error** | \|app's (seat − stroke) − labels' (seat − stroke)\| in ms. This is what we actually report to coaches | **median ≤ 1 frame** |
| Bias | mean of (app − label). A steady bias is fine if it's the same for every seat | report it |

The app's own `timing_error_ms` in `analysis.json` is set from this table (90th-percentile relative offset error), and the charts shade that band as "can't tell".

### The evaluation script (A5) — sketch

```python
# eval/evaluate.py — sketch, Aluinn fills it in
# usage: python eval/evaluate.py            (all clips with labels and a result)
import json, statistics
from pathlib import Path

WINDOW_MS = 150

def match(app_times, label_times):
    """Pair each label with the nearest app time within WINDOW_MS."""
    pairs, used = [], set()
    for lt in label_times:
        best = min((a for a in app_times if a not in used), key=lambda a: abs(a - lt), default=None)
        if best is not None and abs(best - lt) <= WINDOW_MS:
            pairs.append((best, lt)); used.add(best)
    return pairs

for label_file in sorted(Path("eval/labels").glob("*.json")):
    labels = json.loads(label_file.read_text())
    analysis = json.loads(Path(f"data/{labels['clip_id']}/analysis.json").read_text())
    # 1. per seat: catch times from labels and from analysis["strokes"]
    # 2. match(), then absolute errors: abs(app - label)
    # 3. relative: for each stroke, (seat - stroke seat) in app vs in labels
    # 4. found %, extras, stroke-rate error
    # 5. append a row per clip to eval/RESULTS.md (newest first)
```

### Tests that don't need footage

A4 includes a **synthetic rower**: a smooth wave for each seat with a known delay (e.g. seat 1 is 50 ms behind seat 2, rate 30 spm, sampled at a slightly uneven 30 fps). The pipeline must recover 50 ± 2 ms. This catches maths and timestamp bugs before real footage muddies things.

---

## 8. Filming guide (Zac)

### Auditing existing footage (do this first)

A clip is **usable** if all of these are true:

- [ ] The whole crew is in frame, heads to feet, for at least 10 strokes in a row
- [ ] Filmed roughly side-on (within about 30° of square to the boat)
- [ ] One continuous shot (no edits or cuts in the section we use)
- [ ] At least 30 fps (check: `ffprobe` or the Mac's Get Info). 60 fps is much better
- [ ] Rowers are adults, and the club/crew is OK with us using it (section 10)

Write down for each clip: boat type, crew size, which way the boat is moving, fps, camera position (bank or launch), and the start/end time of the usable section.

### Filming new clips

| Setting | Do this | Why |
|---|---|---|
| Angle | Square to the boat, side-on | Hands and feet move along the boat in the picture, which is what we measure |
| Height | A bit above the water: bank steps, a bridge, standing in the launch | Rowers hide each other less |
| Distance | So the whole boat fills about 70–80 % of the frame width. Roughly 15–25 m for a pair, 30–40 m for an eight | Bigger people in frame → better joints |
| Frame rate | **1080p at 60 fps** (iPhone: Settings → Camera → Record Video) | At 30 fps a frame is 33 ms; at 60 fps it's 17 ms. The effects we measure are 20–80 ms |
| Steadiness | Brace the phone or use a small tripod on the bank. From a launch: run parallel at the crew's speed, keep the crew centred, don't zoom mid-piece | Panning is handled, but blur and zoom jumps cost accuracy |
| Exposure | Tap and hold to lock focus and exposure on the boat | Stops the picture pumping as the background changes |
| HDR | Turn off HDR video | HDR files often look washed out in browsers and some tools |
| Light | Overcast or sun behind you. Avoid filming into the sun (rowers become silhouettes) | Pose models need to see limbs |
| Length | 30–60 s, at least 10 full strokes at steady pressure | Enough strokes to say "consistently late" |
| Audio | Phone near the coach; say "go" or call the start clearly | Better transcript; an obvious marker for the start |
| Orientation | Landscape | Boats are long |
| Notes | Say the boat type and direction out loud at the start ("pair, going left to right"). Seat numbers, not names | Saves guesswork; keeps names out of transcripts |

**Avoid:** filming from bow-on or stern-on, crews passing behind posts or other boats, Slo-mo mode (the file plays back slowed down, so timings come out wrong unless the pipeline knows), Cinematic mode, digital zoom.

---

## 9. Risks and fallbacks

| Risk | What happens | What we do | Fallback |
|---|---|---|---|
| **Occlusion** (rowers hide each other side-on) | Joints of the far-side arm or a rower behind another go missing or jump | Use the near-side wrist and ankle; check keypoint confidence; mark low-confidence strokes `valid: false` instead of guessing | Use the hips-to-feet distance (legs compression) for that seat; it's more often visible |
| **Camera panning / moving launch** | Everything slides across the frame | Our measure is inside each rower's body (hands vs feet), so panning cancels out; normalise by torso length for zoom | Reject clips with cuts or big zooms; detect sudden jumps and split the clip |
| **Splash and spray** | Blades and hands blurred | We time from the body, not the blade | — |
| **Bad light / backlit** | Missed joints | Filming guide; confidence checks | Ask for better footage; say "can't tell" |
| **Seat assignment wrong** (direction flipped, cox counted, bystander in the launch counted) | Every seat number is wrong | User picks boat type at upload (so we know N and whether there's a cox); pick the N people with the biggest regular rowing motion; guess direction from travel and face direction | One tap in the UI to flip bow/stroke; a "reassign seats" view |
| **30 fps footage** | ±17 ms rounding per event | Between-frame estimate; error band shown in charts | Only call "late" when it's bigger than twice the measured error and on most strokes |
| **Variable frame rate** | Timing drifts | Use frame timestamps everywhere (A1) | — |
| **Not enough usable footage** | Can't test | Audit in week 1 | Film your own erg clip + a new adult pair with consent |
| **Time budget** (5 h/week is tight) | Tasks slip | Gates each Sunday; Z9, A9 and the LLM rewrite are the first cuts | Ship the MVP (week 5) as the portfolio piece; the rest is a bonus |
| **LLM states things that aren't true** | Bad advice | Rules first; checker rejects any number not in the evidence | Show rule-based feedback only |

---

## 10. Privacy and consent (practical)

We're filming identifiable people, so the video is personal data. This is a practical checklist, not legal advice.

**During the 10 weeks: adults only.** Leave out any clip with under-18s, even if it's great footage. That avoids the parental-consent and safeguarding questions until the tool is worth it. Add juniors later only with written parental consent arranged through the club's welfare officer, following your club's and British Rowing's safeguarding guidance on filming.

**Consent for existing footage.** That footage was filmed for coaching, and we're using it for something new. So:

1. Ask the club (captain or head coach) in writing: what the project is, that video stays on our laptops, that it's deleted when we're done, no faces in anything public.
2. Tell the rowers in each clip and give them a real chance to say no. A short message and a "reply if you'd rather not" is fine for adults; record who said yes in `eval/clips.json` (by seat and clip, not by name).
3. Anything public (portfolio demo, LinkedIn, README) needs a separate, explicit yes from everyone in it, and faces blurred anyway.

**How the app handles video:**

- Everything runs on our own Macs. No video or frames go to any online service.
- `footage/` and `data/` are git-ignored and never pushed, uploaded to shared drives, or put in chats.
- Turn on FileVault (disk encryption) on both Macs.
- `DELETE /analyses/{id}` removes the video and all outputs. The server deletes uploaded video automatically after 7 days by default (setting in `.env`); `analysis.json` (numbers only) can be kept.
- **No face recognition, ever.** People are identified only by seat number. We use face direction only to tell which way the boat faces.
- Label files and `eval/clips.json` hold only numbers and seat numbers, so they can live in the repo.
- If a hosted LLM is ever used, only numbers and transcript text are sent, with names in the transcript replaced by seat numbers first.
- If this becomes a university project or study, it will need ethics approval first.

---

## 11. Which Claude model to use in Claude Code

| Use **Opus 5.5** for | Use **Sonnet 5.5** for |
|---|---|
| Planning and changing this roadmap | UI: player, charts, cards, layout, CSS, phone fixes |
| Anything in `shared/` (the contract) | FastAPI endpoints, file handling, jobs |
| Z2 metric definitions and A4 phase detection / metrics maths | Scripts, tests that follow an existing pattern |
| A3 seat assignment logic | README, BUILD_LOG, docs tidying |
| A8 feedback prompt and number checker | Small refactors |
| Hard debugging: sync drift, timestamps, results that are "nearly right" | |

**When to switch:** start on Sonnet unless the task is in the left column. Switch to Opus if Sonnet has tried the same bug twice, if the problem involves timing or the contract, or if you can't tell why a number is wrong. Switch back once the hard part is done. Use `/model` in Claude Code to change. Each prompt in section 13 names its model.

---

## 12. Working rules

1. **Small commits**, each one doing one thing, with a clear message.
2. **Branches:** Zac on `zac/web`, Aluinn on `aluinn/server`. PRs into `main`; the other person glances at it before merge. Never push to `main`.
3. **`shared/` changes** need a PR approved by both and a contract version bump.
4. **`BUILD_LOG.md`:** one dated entry per session, **newest at the top**: what changed, what's broken, what's next.
5. **`THIRD_PARTY.md`:** every library and model, its version, licence and link. **Nothing new gets installed without stating its licence first and asking.** No AGPL (that rules out Ultralytics YOLO).
6. **Never commit keys.** `.env` is git-ignored; `.env.example` lists the names with no values.
7. **Never commit footage.** `footage/`, `data/` and `web/public/dev/` are git-ignored. Check `git status` before each commit.
8. **Test the web app on a phone** every session (`npm run dev -- --host`, open the address on your iPhone on the same Wi-Fi).
9. **Claude Code stops at the end of each task** and doesn't start the next one.
10. **Plain English, simple over clever.** If you can't explain a function to the other person, simplify it.

---

## 13. First Claude Code prompts

Both prompts assume J1 is done (the contract files are in `shared/` on `main`) and `CLAUDE.md` is in the repo root.

### Z1 — Zac: web scaffold and video player with fake overlays

**Model: Sonnet 5.5** (switch to Opus 5.5 if the skeletons drift out of sync and two fixes haven't solved it).

```
Task Z1 for RowCoach: set up the web app and build a video player that draws
skeletons and seat labels over the video, using fake data.

Read CLAUDE.md, ROADMAP.md (sections 3, 5 and 12) and shared/CONTRACT.md first.

What to build:
1. A Vite + TypeScript app in /web. TypeScript strict, no `any`, no framework.
2. `npm run types`: generate TypeScript types from shared/analysis.schema.json and
   shared/landmarks.schema.json into web/src/types/ using json-schema-to-typescript
   (MIT). Don't hand-write the contract types.
3. web/scripts/make-fake.ts: given a duration in ms, fps and number of rowers,
   write a fake landmarks.json and analysis.json (into web/src/fake/) that match
   the contract and validate against the schemas. Fake rowers should move like a
   simple rowing stroke at about 30 strokes per minute (a smooth back-and-forth of
   the hands and knees), with seat 1 a little behind stroke. Use slightly uneven
   frame times, like a real phone video.
4. A player page:
   - plays a clip from web/public/dev/ (git-ignored; I'll drop a clip there);
   - a <canvas> exactly on top of the <video>, matching the visible picture even
     when the video is letterboxed (object-fit) or the window resizes or rotates;
   - draws on every shown frame using requestVideoFrameCallback: find the frame in
     landmarks.json nearest to the frame's mediaTime (binary search on t_ms) and draw
     each person's skeleton (COCO-17 lines) and a seat label ("1 Bow", "2 Stroke");
   - skip keypoints with confidence below 0.3;
   - play/pause, scrub bar, and step one frame back/forward buttons;
   - works on iPhone Safari (playsinline, no autoplay, touch-friendly buttons at
     least 44 px). Note: iOS Safari's mediaTime can be slightly off, so snap to
     the nearest frame in landmarks.json rather than trusting it exactly.
5. All data loading goes through web/src/api.ts, with USE_FAKE_SERVER in
   web/src/config.ts set to true. Later we'll point it at the real server.
6. Add web/public/dev/ to .gitignore.

What done looks like:
- With a test clip in web/public/dev/, fake skeletons move with the video.
- Pausing, scrubbing and stepping keep the skeletons on the right frame.
- It works on my laptop (npm run dev) and my iPhone (npm run dev -- --host).
- npx tsc --noEmit has no errors.
- Fake files validate against the schemas (a small check script is fine).

How to test: run it, tell me exactly what to click and what I should see, on
laptop and on iPhone.

Rules:
- Work only in /web (and the one .gitignore line). Never change /shared or
  /server; if the contract seems wrong, stop and tell me.
- Explain what you did in plain English. I'm a beginner. Prefer simple over clever.
- Before adding any library, tell me its licence and ask. Add each one to
  THIRD_PARTY.md (name, version, licence, link). No AGPL.
- Small commits on branch zac/web. Don't push; I'll push myself.
- Never touch .env or keys. Never commit video files.
- Add a dated entry at the TOP of BUILD_LOG.md.
- Stop when Z1 is done. Don't start Z2 or anything else.
```

### A1 — Aluinn: server scaffold, video ingest and CLI

**Model: Sonnet 5.5** (switch to Opus 5.5 if frame timestamps look odd and it's not obvious why).

```
Task A1 for RowCoach: set up the Python server and a command that reads a rowing
video and reports exactly how its frames are timed.

Read CLAUDE.md, ROADMAP.md (sections 3, 4, 5 and 12) and shared/CONTRACT.md first.

What to build:
1. A Python 3.11+ project in /server using uv, package name `rowcoach`, type hints
   everywhere, FastAPI app with GET /health returning {"ok": true, "version": ...}.
2. rowcoach/ingest.py using PyAV:
   - open a video (.mov or .mp4, as recorded on an iPhone);
   - read every frame's presentation timestamp and convert to ms from the start
     (never frame_number / fps);
   - read width, height and rotation (phones store rotation as metadata; report the
     upright size);
   - work out nominal fps, measured fps, whether the frame rate is variable
     (frame gaps differ by more than 1 ms), frame count, duration, whether there's
     audio.
   Return a typed object whose fields match the "video" block of analysis.json.
3. A CLI: `uv run python -m rowcoach probe footage/<clip>` prints a short plain
   summary and writes data/<clip_id>/video_info.json and
   data/<clip_id>/frame_times.json (list of t_ms). Never modifies the original.
4. Tests (pytest): generate a tiny synthetic video in the test itself (a few
   frames, including one with uneven frame timing), check timestamps, size and
   the variable-frame-rate flag. No real footage in tests or in the repo.
5. .gitignore covers .env, .venv, footage/, data/. Add .env.example (empty values).

What done looks like:
- `probe` works on 3 real clips from footage/ and the numbers look right (compare
  with ffprobe if installed).
- Variable frame rate is reported correctly on an iPhone clip.
- `uv run pytest` passes; the type checker (pyright or mypy, your choice: say
  which and its licence) is clean.
- GET /health works at http://localhost:8000/health.

How to test: tell me the exact commands to run and what output to expect.

Rules:
- Work only in /server (and .gitignore / .env.example). Never change /shared or
  /web; if the contract seems wrong, stop and tell me.
- Explain what you did in plain English. Prefer simple over clever.
- Before adding any library, tell me its licence and ask. Add each one to
  THIRD_PARTY.md (name, version, licence, link). No AGPL.
- Small commits on branch aluinn/server. Don't push; I'll push myself.
- Never touch .env or keys. Never commit video or anything in footage/ or data/.
- Add a dated entry at the TOP of BUILD_LOG.md.
- Stop when A1 is done. Don't start A2 or anything else.
```

---

*Sources checked 5 Nov 2026 for section 4: MediaPipe Pose Landmarker docs (multi-person `num_poses`); rtmlib (Apache-2.0); MMPose maintenance status; Roboflow `trackers` (Apache-2.0) and `supervision` (MIT); SAM 3 "SAM License"; Qwen3.8-27B (Apache-2.0, ~18 GB in Ollama); Mistral 3 family (Apache-2.0); faster-whisper releases; `requestVideoFrameCallback` browser support. Re-check before relying on any of them.*
