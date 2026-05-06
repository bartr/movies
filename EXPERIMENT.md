# Experiment — Movies

> The repo you just templated is the **experiment harness**. You are participant N of many. Your run is comparable evidence.

## What We Are Measuring

The hypothesis:

> A small Kubernetes-native service (the **movies bar**: API · structured logs · Prometheus metrics · Grafana dashboard · load-test client) can be shipped by **one engineer in a small number of focused sessions** when the engineer uses [sessions](METHODOLOGY.md) + [RPI](METHODOLOGY.md) together.

A comparable bar has been shipped before in two very different ways: a 4-person team over many months, and a single senior engineer in a few days using a year of pre-built tooling. Both results were confounded — by team size in the first case, by senior experience plus reusable tooling in the second.

**This experiment strips both confounds:**
- **Greenfield.** The repo template gives you a spec and data files. No reusable tooling.
- **Stack agnostic.** The spec ([spec.md](spec.md) §1) deliberately does not specify a language. You pick.
- **Wide engineer pool.** Multiple participants run independently. We compare.

If most participants ship the movies bar in roughly the budget the methodology predicts, the multiplier is methodology-driven, not seniority-driven. If they don't, the methodology needs to change — or be honest about who it works for.

## What "Done" Means

You have shipped successfully when **all** §14 acceptance criteria in [spec.md](spec.md#14-acceptance-criteria) are green on a freshly-wiped local k3s cluster, and you have tagged `1.0.0`. Specifically:

- [ ] §9.1 dev-loop steps bring up movies-api + Prometheus + Grafana on a fresh cluster.
- [ ] All §6 endpoints respond per the contract; baseline + benchmark Web Validate suites pass.
- [ ] `/metrics` exposes the §7.1 metrics with the specified names and labels.
- [ ] Logs are valid JSON (one object per line) with §7.2 fields.
- [ ] Grafana dashboard auto-provisions and shows live data.
- [ ] Container image runs as non-root with read-only root FS.
- [ ] §12 inner-loop dev process runs end-to-end on a clean machine.

## What We Track

You track these by simply **using the repo as designed.** No extra reporting tools.

| Signal | Where it lives | Why we care |
|---|---|---|
| **Session count** | git tags (`0.1.0`, `0.2.0`, …, `1.0.0`) | The headline number. How many sessions to ship the bar? |
| **Session duration** | timestamps in [`session-log.md`](session-log.md) (frame time → close time) | Did sessions stay in the 90–120 minute bound? |
| **Fit-check decisions** | recorded in each session block of [`session-log.md`](session-log.md) | Were plans realistic? Where did frames over-promise? |
| **Drift incidents** | "drift moments" field per session in [`session-log.md`](session-log.md) | Was scope held? |
| **RPI artifacts** | `.copilot-tracking/` | Evidence that Research happened before Plan; Plan before Implement. |
| **Stack chosen** | first session's research artifact | For cross-run comparison. |
| **Time-to-1.0.0** | git: tag date of `1.0.0` minus first commit | The summary metric. |
| **§14 checklist** | checked in your final session log paragraph | Does the bar actually pass? |

You do not need to add anything to track these. **Run the methodology as written and the evidence accumulates as a side effect.**

## How to Submit Your Run

When you tag `1.0.0`:

1. Make your repo public (if you used "Use this template" it's already a real repo of yours).
2. Comment on the [submissions tracking issue](https://github.com/context-first/core/issues/6) on `context-first/core` using the template in that issue. At minimum:
   - Link to your repo
   - Stack you chose (one line)
   - Total session count
   - Total wall-clock time-to-1.0.0
   - AI assistant used
   - Link to your `RETRO.md`
   - One-liner: where the methodology helped, where it got in the way
3. Write a short retro in your own repo's `RETRO.md`. The honest one. What broke, what surprised you, what you'd do differently. Even a half-page is valuable.

## Ground Rules

These exist so the runs are comparable evidence, not folklore.

1. **One participant, one session at a time.** Pair-programming changes the unit of analysis. Solo only.
2. **No starting from existing code.** No copy-paste from your other projects. The repo template is the starting line.
3. **Real sessions, not theatrical ones.** A session ends when *you* ran out of focus, not when the clock said 120 minutes. If a session was 75 minutes, log 75. If it was 140, log 140.
4. **The fit check is mandatory.** If you skipped it, log that you skipped it — that is itself useful evidence.
5. **Tag every session.** No untagged work on `main`. Even `0.0.1` ("scaffold builds") is a valid tag.
6. **Honest retros.** A run where the methodology *didn't* help is more valuable evidence than a run that did. Write it down anyway.

## What Is *Not* Being Tested

Naming these explicitly so we don't accidentally measure the wrong thing.

- **Not raw AI assistant quality.** You can use any AI: Copilot, Claude Code, Cursor, Cody, plain Claude.ai chat, ChatGPT. Note which one in your retro.
- **Not which language is fastest.** The spec is stack-agnostic on purpose. We expect Go, Rust, Python, TypeScript, .NET runs all to land at the bar.
- **Not your individual speed.** The unit is *sessions to ship the bar*, not *minutes to ship the bar*. A run that took 8 sessions over 2 weeks calendar-time is the same evidence as one that took 8 sessions over 2 days.

## What to Do If You Get Stuck

- **Re-read the frame.** The most common failure mode is drift. Are you working on what the frame said?
- **Run the fit check again.** Mid-session, it can save the session.
- **Close the session early.** A short session that closes cleanly beats a long one that doesn't. The next session starts fast.
- **Ask the AI to do less.** If a phase is producing slop, the prompt is too broad. Constrain it.
- **Read [METHODOLOGY.md §Failure Modes](METHODOLOGY.md#failure-modes-to-watch-for).** It's a short table; the symptom you're hitting is probably there.

## After You Submit

- The aggregate of submissions is the experiment's evidence.
- The methodology will evolve based on patterns across runs. (For example, the **fit check** itself was added to the methodology mid-experiment after the first dry-run surfaced the gap.)
- Your retro might be the one that produces the next methodology change. The provenance pattern is: **rules earn their keep by surviving real sessions, not by being designed.**

Thanks for running this. The methodology is only as good as the evidence behind it.
