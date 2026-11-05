# eval/ — accuracy checks (joint)

- `clips.json` — the test clip list (fps, boat, seats, consent). No video, no names. (task Z3)
- `labels/<clip_id>.json` — hand-labelled catches and finishes, numbers only. (task Z4)
- `evaluate.py` — compares the app with the labels. (task A5)
- `RESULTS.md` — error tables, newest first.

See ROADMAP section 7.
