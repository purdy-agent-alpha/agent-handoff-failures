# Replay items: 36 steps where a working dispatcher agent went wrong

A household AI assistant ran for five weeks, mostly on a small model. Its job was to dispatch: read a task tracker, brief worker agents, report back to its owner. Its record held 153 failures of context or delegation, about 25 per 100 worker starts (and 40 other failures, counted apart). Three of them clearly came from context size, crowding or position in the prompt, and at most about twenty could have. Most of the rest happened at a handoff: the owner's words summarised into a brief and inverted, state lost at a reset, a stale read, a report that never landed, a rules file that never loaded. This folder holds 36 of those failing steps: what the assistant did at each, and gold checks that say what a correct answer does. The steps were replayed from frozen copies of the real sessions; those copies hold the real household's words and are not released (see the end of this page).

The real household is private, so every item is rewritten into a made-up one [Ash is the assistant and Sam its owner]. Task ids are `A-n`, project ids `Q-n`, the home folder `/home/ash`, the host `ash-host.home.arpa`. Where a failure touched private matters, the topic is replaced by an invented one of the same shape, so the failure and the shape of its checks carry over.

The checks have not been re-scored on the rewritten items.

## An item

`items.json` is a list. Each item has:

| Field | What |
|---|---|
| `id` | the case id in the original catalogue (`E041`) |
| `class` | the catalogue's letter: `B` a brief or summary changed the words, `T` a stale read, `S` state lost across a reset or between sibling tasks |
| `mechanism` | `M1` compression loss at a handoff, `M2` state on disk that the next session did not pull, `M3` stale read, `M4` a rule not in effect (the full taxonomy runs M1 to M8; these items use M1 to M4) |
| `prompt` | `heartbeat` (a timer woke the assistant) or `chat` (someone wrote to it) |
| `in_context` | `true` if the fact that decides the step was in front of the assistant; `false` if it never saw it (output cut short, a fact on a task it did not read). A `false` item tests whether an intervention puts the fact there |
| `arms` | the interventions it was scored in (below) |
| `step` | what the assistant did at this step and what it produced |
| `gold` | the checks (below) |
| `notes` | how the gold was derived |

## The answer and the checks

A replayed model answers with one JSON object:

```
{"acts": [any of create_task, comment, set_status, start_worker, reply, ask_owner, nothing],
 "task_detail": str|null, "brief": str|null, "comment": str|null,
 "status": str|null, "blocked_reason": str|null, "reply": str|null}
```

The gold sheet:

- `acts_any`: at least one of these acts is in the answer. `acts_none`: none of these is.
- `status_any`: the answer's `status` is one of these (`""` stands for no status); any other status fails.
- `must`: what a correct answer says. `must_not`: what the failure said (the inversion, the stale stage). Each check has an `id`, a `claim` (one plain sentence a grader can judge), `re` (Python regexes, case-insensitive, any one matching counts) and `in` (the answer fields to look in; all text fields if absent).
- `judge: true`: a grader model decides the claim, since a regex can hit on another task's text or on a negation. The regexes stand where the grader is unsure. For a `must_not` the grader is asked only when a regex hits, so it can clear a false alarm but never raise one.

An item passes a call when every check passes. Its score is the pass rate over repeats. A model's score is the mean over items: the item, not the call, is the unit.

## Arms: one changed input

- `fresh`: the wake's situation ends with the newest state ("the newest A-n report is round 21").
- `imperative`: the same, then what a rule concludes from it ("so the next round is 22").
- `verbatim`: the standing rules end with "a brief made from someone's words carries them verbatim".
- `pinned`: a list of the owner's standing decisions, rebuilt after each reset.

## What it found (on the original 42 items, six of which are not released here)

| Change | Result |
|---|---|
| a rule to quote the requester verbatim (small model, 13 items) | -0.01 (-0.12 to +0.11): no measurable effect |
| the bare newest fact (small model, 17 stale-read items) | 0.20 to 0.31, +0.11 (0.00 to +0.23) |
| the fact and the conclusion a rule draws (same items) | 0.20 to 0.62, +0.42 (+0.21 to +0.64) |
| "no worker is to start", written as that conclusion | ignored in 9 of 10 calls on two items; refusing the start in code removed the start in replay, though on one item the answer still misstated the next step |

## What is not here, and the limits

- **The frozen prompts are not released.** Each item was replayed from the real session: the standing files, the wake or chat, every tool call and result up to the step. Those hold the real household's words. To score your own agent, write the step's context in your agent's own shape from `step` and `notes`, ask for the answer object above, and check it against `gold`.
- One agent, one household, one model family. The failures were found by model sweeps and classified by models, and by the classifier's own estimate a second coder would move about one row in five between neighbouring classes. Several effects have intervals that touch zero.
- Closest published work: MAST (arXiv:2503.13657), a taxonomy over 1,600+ multi-agent framework traces, and Handoff Debt (arXiv:2606.02875), controlled takeovers of SWE-bench tasks. This set is a production dispatcher's own failures.
