# Movies — Sessions + RPI Experiment

A small Kubernetes-native HTTP API. The interesting thing isn't the API — it's *how you build it*.

You are participating in an experiment to test whether a deliberate session-based AI workflow produces faster, more honest engineering than ad-hoc prompting. Your run is one data point.

---

## Read these in order

| # | File | What it gives you | Time |
|---|---|---|---|
| 1 | [EXPERIMENT.md](docs/EXPERIMENT.md) | The hypothesis, ground rules, what we measure, how to submit | 5 min |
| 2 | [METHODOLOGY.md](docs/METHODOLOGY.md) | Sessions + RPI + the fit check, with a worked example | 10 min |
| 3 | [spec.md](docs/spec.md) | The Movies API spec — your build target | 10 min |
| 4 | [session-log.md](session-log.md) | The log template you'll fill in as you go | skim |

If you skip step 1 or 2, you'll do the experiment wrong and your data won't count. The whole point is the methodology.

---

## What "done" looks like

A `1.0.0` tag on your fork where every checkbox in [spec.md §14](docs/spec.md#14-acceptance-criteria) is green on a freshly-wiped local k3s cluster.

Expect this to take **6–10 sessions of 90–120 minutes each**, ~12–20 hours of focused time over 2–4 weeks. If your first session goes 5 hours, you skipped the fit check. Re-read [METHODOLOGY.md](docs/METHODOLOGY.md).

---

## Step-by-step: getting started

These steps get you to the *start of Session 1*. They do not build anything for you.

### 0. Use the template

On the GitHub repo page, click **Use this template → Create a new repository**. Do not fork — a template gives you a clean history, which is part of the experiment.

Clone your new repo locally.

### 1. Verify your toolchain

You will need, at minimum:
- `git`, `gh` (GitHub CLI), `make` (or your platform's equivalent)
- Docker or Podman
- A local Kubernetes — `k3d`, `kind`, or `minikube` are all fine
- `kubectl`, `kustomize`
- An AI assistant of your choice (Copilot, Claude, Cursor, Aider, plain ChatGPT — RPI is a workflow, not a tool)

Do not install language toolchains yet. Picking the language is part of Session 1.

### 2. Read the four files above, in order

Don't skim. The 25 minutes you spend here saves hours in Session 1.

### 3. Plan your first session in the log — *before* you open an AI prompt

Open [session-log.md](session-log.md). Fill in the **Frame** for Session 1:
- **Goal:** what does done look like for this session?
- **Out of scope:** what are you explicitly not doing?
- **Failure condition:** what would make this session a failure?

Suggested Session 1 goal: **"Choose stack + ship `/version` and `/healthz` end-to-end on local k3s, tagged `0.1.0`."** That is a real session — research the stack, plan the smallest end-to-end slice, fit-check it, implement, review, tag. Do not start broader.

You may pick a different Session 1 — but write it in the log first and defend the frame.

### 4. Run the RPI cycle for Session 1

For the prompts and the 10-step inner loop, see [METHODOLOGY.md → Your Inner Loop, Step by Step](docs/METHODOLOGY.md#your-inner-loop-step-by-step).

**Critical**: do not skip the **fit check** between Plan and Implement. It is the only moment with enough evidence to cut scope honestly. If you skip it once, note it in the session log — that is data.

### 5. Close the session

Tests green → FF-merge → tag → fill in the close fields in [session-log.md](session-log.md) → write your one-paragraph summary. Do not start Session 2 in the same sitting.

### 6. Repeat until §14 is green and you tag `1.0.0`

---

## Submitting your run

See [EXPERIMENT.md → How to Submit Your Run](docs/EXPERIMENT.md#how-to-submit-your-run). Short version: comment on the [submissions tracking issue](https://github.com/context-first/core/issues/6) with your repo link, session count, time-to-1.0.0, and a link to your `RETRO.md`.

---

## Honest run vs. polished run

This experiment values **honest** runs over polished ones. We learn nothing from a participant who:
- Pre-built scaffolding off-record and started the clock at "session 1"
- Skipped the fit check and quietly cut scope inside Implement
- Backfilled the session log from memory at the end
- Edited timestamps to look more disciplined than they were

Drift, blown fit checks, abandoned sessions, and re-frames are **valuable signal**. Log them honestly. The retro is where they pay off.

---

## License

[LICENSE](LICENSE). Your fork is yours; we ask only that you make it public so others can read your session log.
