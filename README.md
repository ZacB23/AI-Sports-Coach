# AI Sports Coach (working name: RowCoach)

Film a rowing crew side-on with a phone and get back:

- stroke rate over the piece
- each seat's catch and finish timing relative to stroke ("seat 3 catches 45 ms late")
- drive:recovery ratio and how steady it is
- the coach's spoken comments, pinned to the moment in the video
- plain-English notes per seat, with the clip that backs them up

Every number comes with a measured error bar. Everything runs locally on our own Macs: no video leaves the machine.

**Status:** planning. Nothing to run yet. See [`ROADMAP.md`](ROADMAP.md) for the plan and [`BUILD_LOG.md`](BUILD_LOG.md) for progress.

## Who builds what

| Folder | Owner | Branch |
|---|---|---|
| `web/` — the web app | Zac | `zac/web` |
| `server/` — video pipeline + API | Aluinn | `aluinn/server` |
| `shared/` — the `analysis.json` contract | Both | PR approved by both |
| `eval/` — test labels and accuracy checks | Both | PR approved by both |
| `docs/` — metric definitions, filming guide | Zac | `zac/web` |

Work on your own branch and open a pull request into `main`.

## Footage and privacy

This repo is **public**. Rowing video never goes in it: `footage/` and `data/` are git-ignored, and so are video files anywhere in the repo. People are identified by seat number only, with no face recognition. See ROADMAP section 10.

## Licences

Every library and model we use is listed in [`THIRD_PARTY.md`](THIRD_PARTY.md) with its licence.
