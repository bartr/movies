# Session Log

> One entry per session. Frame before, ritual after. The log itself is the experiment evidence.
>
> Methodology: [METHODOLOGY.md](docs/METHODOLOGY.md) · Experiment: [EXPERIMENT.md](docs/EXPERIMENT.md) · Spec: [spec.md](docs/spec.md)

Copy the **Session Template** block below for each new session. Fill in the frame *before* you start, the close fields *after* you tag.

**Time-logging rule:** write **Start time** the moment the frame is done, and **End time** the moment the close ritual is done. Do **not** estimate after the fact. During the close ritual itself, cross-check against git (first session commit ≈ start; merge/tag timestamp ≈ end) and reconcile any drift now — not at release time.

---

## Session Template

### Session N — [date]

**Frame** (fill in *before* starting — 2 minutes)
- Goal: what does done look like for this session?
- Out of scope: what am I explicitly not doing today?
- Failure condition: what would make this session a failure?

**Start time:** HH:MM  *(write this the instant the frame is done — not later)*

**RPI cycle**
- Research: `.copilot-tracking/YYYY-MM-DD-<topic>-research.md`
- Plan: `.copilot-tracking/YYYY-MM-DD-<topic>-plan.md`
- Changes: `.copilot-tracking/YYYY-MM-DD-<topic>-changes.md`
- Review: `.copilot-tracking/YYYY-MM-DD-<topic>-review.md`

**Fit check** (after Plan, before Implement — 2 minutes)
- Will this plan fit in 90–120 min? (yes/no)
- Smallest cut if no:
- Decision: (proceed / cut: <what> / re-frame)

**During**
- Drift moments (threads I wanted to pull but didn't):
- Parking lot (revisit between sessions):

**Close ritual**
- [ ] Tests green
- [ ] FF-merge (`gh pr merge --rebase --delete-branch`)
- [ ] Tag (`git tag X.Y.Z && git push origin X.Y.Z`)
- [ ] Repo memory updated
- [ ] **End time written the moment the ritual completes** (not estimated later)
- [ ] **Git timestamp cross-check done now** (first session commit vs. Start; merge/tag vs. End — reconcile any drift here, not at release time)
- [ ] One-bullet entry appended to [`RETRO.md`](RETRO.md) for this session
- [ ] Next session starter (one sentence — where does the next session begin?):

**End time:** HH:MM  *(write this the instant the ritual is complete — not later)*
**Total focus minutes:**
**Tag shipped:** X.Y.Z

**One-paragraph summary**
What I built · what I decided · what matters for next time.

**Health signal**
- Framing quality (1–5): did the frame hold?
- Drift (yes/no): did I leave scope?
- Fit check honest (yes/no): did I record a real decision, not a vibe?
- Close complete (yes/no): tests · merge · tag · memory · paragraph?

---

## Session 1 — [date]

**Frame**
- Goal:
- Out of scope:
- Failure condition:

**Start time:**

**RPI cycle**
- Research:
- Plan:
- Changes:
- Review:

**Fit check**
- Will this plan fit in 90–120 min?
- Smallest cut if no:
- Decision:

**During**
- Drift moments:
- Parking lot:

**Close ritual**
- [ ] Tests green
- [ ] FF-merge
- [ ] Tag
- [ ] Repo memory updated
- [ ] End time written in the moment
- [ ] Git timestamp cross-check done now
- [ ] One-bullet entry appended to [`RETRO.md`](RETRO.md)
- [ ] Next session starter:

**End time:**
**Total focus minutes:**
**Tag shipped:**

**One-paragraph summary**


**Health signal**
- Framing quality (1–5):
- Drift (yes/no):
- Fit check honest (yes/no):
- Close complete (yes/no):

---

<!-- Copy the Session Template block above for each new session. -->
