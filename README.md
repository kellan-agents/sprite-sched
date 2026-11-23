# sprite-sched

External clock for an evener sprite (a Fly.io Sprite running evener). The
sprite pauses ~30s after inbound activity stops and only inbound HTTP can
wake it, so the wake trigger must live off-machine. This repo holds
`due.json` — opaque job ids with due times, no secrets — and two workflows:

- **`wake.yml`** — checks `due.json` every 5 minutes (GitHub's cron floor)
  and pings the sprite's public URL only when a job is due, marking entries
  fired so each job wakes the machine at most once.
- **`ping-at.yml`** — manual dispatch for a guaranteed one-shot ping at an
  exact moment (use it while a new repo's cron schedule is still activating,
  or whenever timing matters).

The wake URL is **never committed**: it lives in the `SPRITE_URL` Actions
repo secret (set it under Settings → Secrets and variables → Actions). Both
workflows fail fast with a clear error when the secret is unset.

## The `due.json` contract

```json
{"jobs": [{"id": "<opaque id>", "due_epoch": <unix seconds>, "due_iso": "<UTC ISO>", "fired": false}]}
```

- `id` — opaque, produced by the sprite-side `sched` tool.
- `fired` — flipped to `true` by `wake.yml` after a delivered ping; the
  sprite-side gate handles everything from there.

## Sprite side

Jobs are registered from the sprite with the `sched` tool
(`~/projects/evener-sched/bin/sched`), which writes the job spec to the
sprite's local queue and pushes the wake entry here. The sprite-side `gate`
service (also in `~/projects/evener-sched`) runs due jobs as unattended
evener one-shots and releases the machine to sleep when they finish.

Timing is best-effort: GitHub cron has a 5-minute floor and can lag several
minutes at busy times.
