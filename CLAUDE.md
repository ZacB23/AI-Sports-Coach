# CLAUDE.md — RowCoach

## The project in 3 lines
RowCoach turns a side-on phone video of a rowing crew into stroke rate, each seat's catch and finish timing relative to stroke, and drive:recovery ratio, with an honest error bar on every number.
It adds the coach's spoken comments pinned to the video and short plain-English feedback per seat, backed only by the computed numbers.
Portfolio project by Zac and Aluinn, running fully on our own Macs (no cloud, no API keys needed). Full plan: `ROADMAP.md`.

## Who builds what
| Folder | Owner | Branch | What's in it |
|---|---|---|---|
| `web/` | Zac | `zac/web` | Vite + TypeScript app: upload, video player with overlays, timing charts, feedback, label mode |
| `server/` | Aluinn | `aluinn/server` | Python + FastAPI: ingest, pose, seats, phases, metrics, transcription, feedback, endpoints |
| `shared/` | Joint | PR approved by both | The contract: `analysis.schema.json`, `landmarks.schema.json`, `examples/`, `CONTRACT.md` |
| `eval/` | Joint | PR approved by both | Clip list, hand labels, `evaluate.py`, `RESULTS.md` |
| `docs/` | Zac | `zac/web` | `metric-definitions.md` (how catch, finish, "late" etc. are defined), `filming-guide.md` |

Only work in the folder your task is for. If something in another folder or in `shared/` looks wrong, **stop and say so**; don't change it.

## The contract (`analysis.json` + `landmarks.json`)
`shared/analysis.schema.json` is the source of truth. The server validates its output against it in tests; the web app generates its types from it (`npm run types`). Never hand-write contract types.

Conventions (full list in `shared/CONTRACT.md`):
- **Time:** milliseconds from the start of the video, floats rounded to 0.1, taken from each frame's real timestamp. **Never `frame_number / fps`** (phone video has a variable frame rate).
- **Image coordinates:** normalised 0–1 on the upright frame, origin top-left, x right, y **down**.
- **Seats:** `1` = bow … `N` = stroke. The cox is never a seat.
- **Offsets:** seat time − stroke seat time, same stroke. **Positive = late.**
- **Rate:** `60000 / stroke-seat catch-to-catch ms` (spm). **Ratio:** `recovery_ms / drive_ms` (2.0 = "1:2").
- **Missing values:** `null`, never `0`. Strokes with any seat missing are `valid: false` and excluded from averages.
- **Keypoints:** COCO-17 names and order; each person's `kp` is 51 numbers (`x, y, confidence` × 17).
- **Changes:** PR touching `shared/`, approved by both, bump `contract_version`.

Endpoints: `POST /analyses`, `GET /analyses/{id}`, `GET /analyses/{id}/analysis.json`, `GET /analyses/{id}/landmarks.json`, `GET /analyses/{id}/video`, `DELETE /analyses/{id}`, `GET /health`. The web app always calls **relative** paths; Vite proxies to `localhost:8000` in dev.

## How to run
**Server** (from `server/`):
```
uv sync
uv run python -m rowcoach probe ../footage/<clip>     # inspect a clip
uv run python -m rowcoach run ../footage/<clip>       # full pipeline → ../data/<id>/
uv run uvicorn rowcoach.main:app --reload --port 8000
uv run pytest
```
**Web** (from `web/`):
```
npm install
npm run types            # regenerate types from shared/ schemas
npm run dev              # laptop
npm run dev -- --host    # phone on the same Wi-Fi
npx tsc --noEmit         # must be clean
```
Fake data: `USE_FAKE_SERVER` in `web/src/config.ts`; all server calls go through `web/src/api.ts`. Put a local test clip in `web/public/dev/` (git-ignored).

**Evaluation** (from repo root): `python eval/evaluate.py` → appends to `eval/RESULTS.md`.

## Coding conventions
- **TypeScript:** strict mode, no `any`, no framework. Small modules; keep maths (pure functions) separate from drawing code.
- **Python:** 3.11+, type hints on every function, Pydantic for data shapes, pure functions for maths in `phases.py` / `metrics.py` with unit tests.
- Name units in variable names (`catch_ms`, `drive_ms`, `rate_spm`).
- The LLM never invents numbers: every number in feedback must appear in that item's `evidence` and match `metrics`.

## Rules
- **Explain in plain English. Prefer simple over clever.** Zac is a beginner coder.
- Small commits on your own branch; PRs into `main`. **Never push to `main`.** Don't push at all unless asked; the human pushes.
- Add a dated entry at the **top** of `BUILD_LOG.md` each session (newest first).
- Before adding any library or model, **state its licence and ask**. Then add it to `THIRD_PARTY.md` (name, version, licence, link). No AGPL (so no Ultralytics YOLO).
- **Never commit keys.** `.env` is git-ignored; `.env.example` has names only. Never read or edit `.env`.
- **Never commit footage.** `footage/`, `data/`, `web/public/dev/` are git-ignored. No real video in tests. Check `git status` before committing.
- **Privacy:** no face recognition; people are identified by seat number only. No video or frames go to any online service.
- Test web changes on a phone as well as the laptop.
- Stop at the end of the task you were given. Don't start the next one.
