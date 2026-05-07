# Methodology — How to Run This Experiment

> A self-contained primer. You do **not** need to read any other repo to run this experiment. Links to source material are provided for depth.

This experiment uses two methodologies together:

- **Sessions** — the [context-first](https://context-first.ai) outer loop. The unit of work is one ~90–120 minute focused session, ending in a tagged dot-release.
- **RPI** — Microsoft's [HVE Core](https://github.com/microsoft/hve-core) Research → Plan → Implement → Review inner loop. One RPI cycle runs *inside* one session.

Total reading time before you start: about 10 minutes.

## The Session Model in 60 Seconds

A session has three beats. Plan them. Don't skip any.

**Before — the frame** (2 minutes, written down)
- What does *done* look like for this session?
- What is explicitly *out of scope*?
- What would make this session a *failure*?

If you can't answer all three, the session is not ready.

**During — one rule**
When you feel the urge to pull a thread outside your frame, write it in the *parking lot* and stay in scope. Context drift is the enemy.

**After — the close ritual**
- Green tests
- FF-merge: `gh pr merge --rebase --delete-branch`
- Tag: `git tag X.Y.Z && git push origin X.Y.Z`
- Update repo memory (`CLAUDE.md`, `.github/copilot-instructions.md`, etc. — whatever your AI reads)
- One paragraph in [`session-log.md`](session-log.md): what you built, what you decided, where the next session starts.
- Append one bullet to [`RETRO.md`](../RETRO.md) — what surprised you this session, what you'd do differently. One line is fine.

That paragraph is the compounding mechanism. Without it, each session starts cold.

**Time-logging rule.** Write **Start time** into [`session-log.md`](session-log.md) the moment you finish the frame, and write **End time** the moment you finish the close ritual — not later, not from memory. As a cross-check *during* the close ritual (not at release time), compare against git: the first commit on your session branch ≈ start, the merge/tag timestamp ≈ end. If they disagree by more than a few minutes, fix the log now while you can still remember.

For depth: [sessions-not-stories.md](https://github.com/context-first/core/blob/main/methodology/sessions-not-stories.md).

## The RPI Workflow in 60 Seconds

The core insight: **AI assistants cannot tell investigating from implementing.** Asked for code, they write code — including code for APIs that do not exist. RPI fixes this by running each phase under a constraint that *prevents* the next one. When the AI knows it cannot implement, it stops optimizing for plausible code and starts optimizing for verified truth.

Four phases. Each ends with a written artifact in `.copilot-tracking/`. **Start a new chat between every phase** (or `/clear` if your tool supports it). Findings live in the files, not in chat history.

| Phase | Constraint | Output file |
|---|---|---|
| **Research** | Cannot plan or write code. Must investigate, cite findings, recommend one approach. | `.copilot-tracking/YYYY-MM-DD-<topic>-research.md` |
| **Plan** | Cannot research or write code. Must turn research into an ordered, checkbox-able task list. | `.copilot-tracking/YYYY-MM-DD-<topic>-plan.md` |
| **Implement** | Cannot replan. Executes the plan task by task. | Working code + `.copilot-tracking/YYYY-MM-DD-<topic>-changes.md` |
| **Review** | Cannot modify code. Validates implementation against research and plan. | `.copilot-tracking/YYYY-MM-DD-<topic>-review.md` |

**You do not need any specific extension or tool.** The HVE Core VS Code extension provides `/task-research`, `/task-plan`, etc. as sugar — but the workflow is a *convention*, not a tool. A short prompt at the start of each phase substitutes for the agent: *"Research only. Do not plan or implement. Output a file at the path below."*

For depth: [HVE Core RPI guide](https://github.com/microsoft/hve-core/blob/main/docs/rpi/README.md).

## The Fit Check (30 seconds)

Between **Plan** and **Implement**, take two minutes:

1. Will this plan fit in 90–120 minutes? Honest gut, not optimistic.
2. If no, what is the smallest cut that keeps the session release-able? Name it.
3. Proceed, cut, or re-frame? Record the decision in [`session-log.md`](session-log.md).

The fit check is the cheapest moment to cut scope. A deleted bullet costs nothing; abandoned implementation work costs the rest of the session. **Never skip it.**

## How One Session Maps to One RPI Cycle

```
┌─ Session (outer loop) ──────────────────────────────────────────┐
│  Frame                                                          │
│  ┌─ RPI cycle ─────────────────────────────────────────────┐    │
│  │  Research → Plan → fit check → Implement → Review       │    │
│  └─────────────────────────────────────────────────────────┘    │
│  Close (green tests · FF-merge · tag · repo memory · starter)   │
└─────────────────────────────────────────────────────────────────┘
```

One session ≈ one RPI cycle. If you need more than one cycle, your session is probably two sessions.

## A Worked Example (One Session)

Concrete shape from a real planning session. Your stack/timing will differ; the *structure* is what matters.

**Frame** (written into [`session-log.md`](session-log.md) before starting)
- Goal: Service end-to-end on localhost — data loaded, all `GET` endpoints with validation, integration tests, ≥80% coverage.
- Out of scope: `/healthz`, `/metrics`, container, k8s manifests.
- Failure condition: schema invented rather than inferred from data files; any validation rule missing a negative test.

**Research** → `.copilot-tracking/2026-05-06-stack-research.md`

A short doc comparing candidate stacks (Go/Rust/Python/TS) for a small read-only API. Each option scored on image size, cold start, Prometheus client maturity, OpenAPI story. One recommendation. Explicit "what we are NOT taking" list.

**Plan** → `.copilot-tracking/2026-05-06-service-mvp-plan.md`

Numbered task list: skeleton → inferred types → store → router → handlers → tests → smoke. Each task has a checkbox, file targets, and exit criteria. Parking lot for decisions deferred to Implement.

**Fit check**

Plan walked through honestly. Tests + coverage gate alone are 25–30 min; ambiguous response shapes need a 2-min "freeze" beat at the top. Stretch item is 45–60 min on its own. Decision recorded: **proceed without stretch**; defer to next session. This decision saves the session.

**Implement**

Ten focused minutes of "freeze ambiguities," then mechanical execution against the plan. The AI writes most of the code from the plan + data files. Negative tests are table-driven, one row per validation rule.

**Review**

Coverage report green. One follow-up logged. Closes cleanly.

**Close ritual**

`go test -race ./...` green · PR opened, self-reviewed, `gh pr merge --rebase --delete-branch` · `git tag 0.1.0 && git push origin 0.1.0` · repo memory updated with the inferred schema decisions · one paragraph in [`session-log.md`](session-log.md) + next-session starter.

## Your Inner Loop, Step by Step

For each session:

1. **Frame** — fill in the next blank session block in [`session-log.md`](session-log.md). Two minutes.
2. **Research** — new chat. Prompt: *"Research only — investigate X, cite findings with file paths and line numbers, recommend one approach. Do not plan or implement. Save your output to `.copilot-tracking/<today>-<topic>-research.md`."*
3. **New chat** (or `/clear`). Open the research file in your editor.
4. **Plan** — *"Plan only — read the research file, produce an ordered checkbox task list with file/line refs and exit criteria. Do not implement."*
5. **Fit check** — two minutes. Decision into the session log.
6. **New chat.** Open the plan file.
7. **Implement** — *"Implement only — execute the plan task by task. Verify after each. Record changes in `.copilot-tracking/<today>-<topic>-changes.md`."*
8. **New chat.** Open plan + changes log.
9. **Review** — *"Review only — validate the implementation against the research and plan. Run lint/build/test. Identify follow-ups. Save to `.copilot-tracking/<today>-<topic>-review.md`."*
10. **Close ritual** — green tests, FF-merge, tag, repo memory, paragraph + next-session starter in [`session-log.md`](session-log.md), one-bullet append to [`RETRO.md`](../RETRO.md), Start/End times written in the moment, and a git-timestamp cross-check before you walk away.

## When to Skip

- **Skip RPI for trivial tasks** — typo fixes, log statements, refactors under ~50 lines. RPI overhead exceeds the work.
- **Skip the formal frame for spike work** where the goal is genuinely *"see what's possible in 30 minutes."* Spikes are not sessions; they are exploration.
- **Never skip the fit check.** It costs two minutes and is the highest-leverage beat in the workflow.

## Failure Modes to Watch For

| Smell | Probable cause | Fix |
|---|---|---|
| Session ran past 120 min | Frame was wrong, fit check was skipped | Close short next time. Re-frame. |
| Session has no tag at the end | Goal was too big, no release-able cut existed | Smaller frame next time. |
| Next session starts cold | Close ritual paragraph was skipped or vague | Be specific in the next-session starter. |
| AI generates code that uses APIs that don't exist | Research phase was skipped or rushed | Force a Research artifact before Plan. |
| Plan keeps drifting during Implement | Plan wasn't specific enough; or fit check said "proceed" when it should have said "cut" | Cut harder next session. |

## Source Material (Optional Deeper Reading)

- [context-first methodology repo](https://github.com/context-first/core) — the canonical home of the session model.
- [sessions-not-stories.md](https://github.com/context-first/core/blob/main/methodology/sessions-not-stories.md) — full session primitive write-up.
- [sessions-and-rpi.md](https://github.com/context-first/core/blob/main/methodology/sessions-and-rpi.md) — long-form version of this primer.
- [HVE Core](https://github.com/microsoft/hve-core) — the upstream RPI source.
- [HVE Core RPI guide](https://github.com/microsoft/hve-core/blob/main/docs/rpi/README.md) — full RPI write-up with per-phase agents.

You should not need any of these to run this experiment. They are here for the curious.
