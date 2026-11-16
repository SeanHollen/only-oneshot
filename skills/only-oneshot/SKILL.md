---
name: only-oneshot
description: Build an app from a single spec prompt that acts as its source code. Every later correction or follow-up is written into that spec (never patched into the code). The app folder is then wiped and a fresh subagent rebuilds the whole app from the amended spec alone, and the parent checks the result. Use when the user says "only-oneshot", "/only-oneshot", "only one shot", "build this one-shot style", or wants a prompt they can re-run on future models. Once invoked, every later message in the conversation about the app counts as a spec amendment.
---

# Only-oneshot: the prompt is the source, the code is the binary

The spec (`ONESHOT.md`) is the source of truth. The app code is a build output: disposable,
regenerated from the spec, never edited by hand. When this skill finishes, the user has (1) a working
app and (2) a complete spec that rebuilds it from nothing, on any model, at any time.

**The one rule:** never fix the app by editing its code. Every change goes into the spec, then
the app is rebuilt from scratch. (The single exception: the user explicitly says "just patch the code".)

## Layout

```
<project>/
  ONESHOT.md        the spec (source). The only thing the builder sees.
  app/              generated app (binary). Wiped before every build.
  .oneshot/log.md   history: the user's messages verbatim, spec changes, build results.
  .oneshot/runs/    optional side-by-side builds for comparing models (see "Compare").
```

- Project folder: the one the user names; otherwise a new `<short-kebab-name>/` folder next to the current project. If the current
  directory is empty and clearly meant for this, use it.
- `git init` the project (local only; never push unless the user asks). Add a `.gitignore` for
  `node_modules`, `.venv`, `dist`, `build`, `.env`, logs.

## Writing the spec

`ONESHOT.md` structure:

```
# <App name>
## Goal            what it is, who it's for, in the user's words
## Stack           only what the user specified or that was settled in a later round
## Requirements    grouped by area (layout, behavior, data, look and feel, ...)
## Acceptance checks  numbered, observable, one per requirement worth checking
```

Rules:
- **Start from the user's original prompt, kept close to their words.** Organize it into the sections,
  but don't invent features. Where they left something open, leave it open (that's the room a
  better model uses).
- **State the desired end state, in positive terms.** The spec describes what the app *is*, not
  what went wrong in some earlier build. A different model may never make that mistake, so a
  "don't do X" line is noise for it.
  - "stop the button flickering" → "The button keeps a fixed size and position and changes
    color smoothly across all states (idle, hover, loading, disabled)."
  - "don't use a modal for settings" → "Settings live on their own page, reached from the header."
  - "the list is too slow" → "The list stays smooth with 10,000 items (virtualized rendering)."
  - "get rid of the reset button" → list exactly what's there: "The controls are the bill field,
    the tip options and the people count." Naming the removed thing ("no reset button") teaches
    every future model about a thing it would never have added.
  Negatives are allowed, just not the default. They fit when the positive version would be
  clumsy or vaguer, for a hard constraint that stands on its own ("No tracking or analytics"),
  or when the mistake is common enough that most model families plausibly make it. The cost: a
  "don't" aimed at one model's quirk is fragile, and once models stop making that mistake it's
  dead weight. Acceptance checks follow the same preference: check for what should be there
  ("the controls are exactly ..."), not for the absence of a past mistake.
- **Negation review before every commit:** scan the spec for no / not / don't / never / avoid /
  without. For each one, ask whether a clear positive version exists. If it does, use it. If not,
  or if the mistake is common across models, keep the negative.
- **Put each amendment where it belongs**, merged into the right section, rewriting existing
  lines if needed. No changelog or "Corrections" section at the bottom; the spec should read as
  if it had been written that way from the start.
- **No references to past builds** ("like before", "keep the current design", "the bug from
  round 2"). The builder has never seen any of it. If the user likes something the last build did by
  chance, describe it concretely in the spec so it's kept.
- Add an acceptance check for each amendment, plus checks for earlier requirements that matter.

## The loop (every round)

Round 0 is the user's first message. Every later message about the app is a new round.

1. **Log** the user's message verbatim in `.oneshot/log.md` under `## Round N`.
2. **Amend** `ONESHOT.md` (round 0: write it). In the log, note in one or two lines what changed in
   the spec and why.
3. **Commit** everything: `git add -A && git commit -m "round N: <summary>"`. This keeps the
   previous app build and spec revertible.
4. **Wipe** the app folder: `find <project>/app -mindepth 1 -delete` (double-check the path ends
   in `/app` first; never wipe anything else).
5. **Build** with a fresh subagent (Agent tool, `general-purpose`, foreground). Its prompt is
   exactly the builder brief below with the full spec pasted in. Nothing else: no log, no past
   feedback, no hints from earlier builds.
6. **Verify** yourself (the parent): go through every acceptance check, with extra care on this
   round's amendment. Use real evidence: build it, run tests, start it and hit it, take a headless
   screenshot of UI and look at it (stop any server you start). Record pass/fail per check in the log.
7. **If anything fails:** treat it as a spec problem. Decide why the builder missed it (ambiguous,
   too vague, buried, conflicting with another line, phrased as a negative) and rewrite that part
   of the spec. Then repeat steps 3-6. **Max 3 build cycles per round.** If it still fails, stop,
   leave the best build committed, and tell the user what keeps failing and what you tried in the spec.
8. **Commit** the final state: `round N: done (cycle k)`.
9. **Report** to the user (short): what changed in the spec (quote the new lines),
   how many cycles it took, the check results, and how to open/run the app. If it's a web app, offer to
   host or share it.

### Builder brief (subagent prompt)

> Build the application described in the specification below, from scratch, into the empty folder
> `<project>/app/`. Work only inside that folder. Don't read anything else in `<project>`
> (no git history, no `.oneshot/`). Make every requirement in the spec true and make sure each
> acceptance check passes; run the app or its tests yourself before finishing. Where the spec
> leaves something open, use your best judgment and aim for a polished, complete result.
> When done, reply with: how to run it, and any requirement you couldn't meet and why.
>
> --- SPECIFICATION ---
> <full contents of ONESHOT.md>

Model: default is the current model. If the user names one ("build with sonnet"), pass it as the
Agent `model`. For another provider's model, use whatever delegation tool the harness offers, with the same brief.

## Compare (re-running the spec)

When the user asks to re-run or compare ("try it on Fable", "rebuild with the new model", "how
different are the results"): build into `.oneshot/runs/<model>-<date>/` instead of `app/` (no
wipe of `app/`), with the same builder brief and the spec unchanged. Then run the acceptance checks
on each build and give a short side-by-side: checks passed, notable differences, which looks
better (screenshots if UI). Only replace `app/` with a run if the user says so.

## Edge cases

- The user asks a question (not a change): just answer it; no round.
- The user wants to undo a round: `git revert`/checkout the spec at that commit, then run the loop.
- Data the app keeps (databases, uploads) that must survive rebuilds: put it outside `app/`
  (e.g. `<project>/data/`) and say so in the spec.
