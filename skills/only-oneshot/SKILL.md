---
name: only-oneshot
description: Build an app from a spec prompt that acts as its source code (one file, or a tree of module specs for large apps). Every later correction or follow-up is written into that spec (never patched into the code). Then the app, or only the affected modules, is deleted and fresh subagents regenerate it from the amended spec alone, and the parent checks the result. Use when the user says "only-oneshot", "/only-oneshot", "only one shot", "build this one-shot style", or wants a prompt they can re-run on future models. Once invoked, every later message in the conversation about the app counts as a spec amendment.
---

# Only-oneshot: the prompt is the source, the code is the binary

The spec (`ONESHOT.md`, plus one spec file per module under `spec/` for large apps) is the source of truth. The app code is a build output: disposable,
regenerated from the spec, never edited by hand. When this skill finishes, the user has (1) a working
app and (2) a complete spec that rebuilds it from nothing, on any model, at any time.

**The one rule:** never fix the app by editing its code. Every change goes into the spec, then
the affected code is deleted and regenerated from scratch (the whole app, or only the affected
modules of a modular app). (The single exception: the user explicitly says "just patch the code".)

## Layout

```
<project>/
  ONESHOT.md        the spec (source), or its top level for a large app. The builder sees only spec files.
  spec/             optional module specs for large apps (see "Modular apps").
  app/              generated app (binary). Wiped before every full build.
  .oneshot/manifest.json  which code paths each module produced (modular apps; a build output, not spec).
  .oneshot/wt/      temporary git worktrees for parallel module builders.
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
  better model uses). Obvious-seeming extras (edit, delete, rename, settings, export) count as
  features too: offer them in the report as suggestions, and add them only if the user says yes.
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

## Modular apps: one spec file per module

Default is a single `ONESHOT.md`; most small and mid-size apps fit in one file and are always
rebuilt whole. Make the spec modular when the app has clearly separate concerns, or one file stops
being readable (roughly 400+ lines). Tell the user when you split and why. "Module" is meant
loosely: a package, service, feature folder, screen, crate, whatever the stack's natural unit is.

```
ONESHOT.md                 top level: the whole app at a glance + the module map
spec/
  auth.md                  one file = one module
  feed/README.md           a module can have submodules, each its own file
  feed/ranking.md
  shared/data-model.md     contracts several modules depend on
```

- **Each spec file is one module with one concern.** Draw module boundaries so modules depend on
  each other as little as possible and only through stated interfaces. That's what makes it
  possible to regenerate one module without touching the rest.
- **The spec asks for modular code.** The top level's Stack section states that the code is split
  into modules matching the spec's module map, in whatever form is idiomatic for the stack, and
  that modules talk to each other only through their stated interfaces.
- **`ONESHOT.md` stays a complete summary:** goal, stack, the module map (every module, a one-line
  purpose, a link to its spec, and what it depends on), cross-cutting requirements (look and feel,
  performance, accessibility) and end-to-end acceptance checks. Someone reading only this file
  understands the app and how its modules fit together.
- **Each module file** has: its purpose, its **interface** (what it offers other modules:
  functions, endpoints, components, events, in behavioral terms), `Depends on:` (links to other
  modules or shared contracts), its requirements and its own acceptance checks. A module with
  submodules uses `README.md` as its top file, pointing to its submodule files the same way.
- **No file paths in the spec.** Where the code for a module lives is an implementation choice of
  the builder; it's tracked in `.oneshot/manifest.json`, which is a build output.
- **Shared contracts** (data model, API shapes, events, design tokens) live under `spec/shared/`,
  each defined in exactly one place, and modules link to them instead of restating them. Keep
  each contract file narrow, split by who uses it: the stored record (storage and its callers)
  is a different file from the API response shape (API and UI). A module links only to the
  contracts it actually uses, so a change to one doesn't drag unrelated modules into a rebuild.
- **Every spec file is reachable from `ONESHOT.md`** through links.
- **Amendments go in the module that owns the behavior.** Update `ONESHOT.md` too when the change
  affects the overview, the module map, a cross-cutting rule or an end-to-end check. If a change
  forces two modules to know about each other's internals, fix the boundary (move the behavior,
  or add it to an interface or shared contract).
- All the writing rules above (positive phrasing, negation review, no references to past builds)
  apply to every file.

### The manifest

`.oneshot/manifest.json` maps each module to the code paths it produced, taken from git diffs (see
"Worktrees"), never from what a builder says it did:

```json
{"modules": {"auth": ["app/src/auth/**"], "feed": ["app/src/feed/**"]},
 "integration": ["app/src/main.ts", "app/package.json", "app/src/routes.ts"]}
```

Glue that several modules touch (entry point, routing, dependency lists, build config) belongs to
`integration`, which is regenerated whenever any module is.

## The loop (every round)

Round 0 is the user's first message. Every later message about the app is a new round.

1. **Log** the user's message verbatim in `.oneshot/log.md` under `## Round N`.
2. **Amend** the spec (round 0: write it; make it modular only if "Modular apps" calls for it). In the log, note in one or two lines what changed in
   the spec and why.
3. **Commit** everything: `git add -A && git commit -m "round N: <summary>"`. This keeps the
   previous app build and spec revertible.
4. **Decide the rebuild scope** (modular apps only; a single-file spec is always a full rebuild).
   See "Choosing what to rebuild". Write the decision and the reason in the log.
5. **Delete** what's being rebuilt, before any builder starts.
   - Full rebuild: `find <project>/app -mindepth 1 -delete` (double-check the path ends in `/app`).
   - Selective: delete exactly the manifest paths of the chosen modules and of `integration`, and
     nothing else.
6. **Build** with fresh subagents (Agent tool, `general-purpose`, foreground). Each prompt is the
   builder brief below with its spec files pasted in. Nothing else: no log, no past feedback, no
   hints from earlier builds. Single-file spec: one builder. Modular spec: see "Modular build".
7. **Verify** yourself (the parent): go through every acceptance check in every spec file
   (after a selective rebuild too, to catch regressions in modules that were kept), with extra care on this
   round's amendment. Use real evidence: build it, run tests, start it and hit it, take a headless
   screenshot of UI and look at it (stop any server you start). Record pass/fail per check in the log.
8. **If anything fails:** treat it as a spec problem. Decide why the builder missed it (ambiguous,
   too vague, buried, conflicting with another line, phrased as a negative) and rewrite that part
   of the spec. Then repeat steps 3-7. If a selective rebuild fails in a way that points at a
   kept module (an interface mismatch, say), widen the scope. **Max 3 build cycles per round.** If it still fails, stop,
   leave the best build committed, and tell the user what keeps failing and what you tried in the spec.
9. **Commit** the final state: `round N: done (cycle k)`.
10. **Report** to the user (short): what changed in the spec (quote the new lines), what was
   rebuilt (whole app or which modules, and why), how many cycles it took, the check results, and how to open/run the app. If it's a web app, offer to
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
> (modular spec: then each module file this builder gets, headed `--- spec/<path> ---`)

For module, scaffold and integration builders, replace the first two sentences with their role
from "Modular build": which folder or worktree they work in, which module they own, that they
change only that module (or only glue code), and, for a selective rebuild, that the rest of the
app is already there and may be read. The rest of the brief stays the same.

### Choosing what to rebuild (modular apps)

Start from the spec diff of this round (`git diff HEAD~1 -- ONESHOT.md spec/`):
- A module's internal requirements changed, interface unchanged → rebuild that module only.
- A module's interface changed → rebuild it and every module that depends on it.
- A shared contract changed → rebuild every module that uses the changed part. An added field
  that a module never reads (say, a computed field in an API response, from storage's point of
  view) leaves that module alone. If a small change keeps forcing wide rebuilds, the contract is
  too broad: split it.
- A cross-cutting rule or the stack changed, or the module map itself changed → full rebuild.
- Unsure how far a change reaches → include more, not less.
Always regenerate `integration` along with any module. Also do a **full rebuild** when the user
asks, when comparing models, and before calling the spec finished: a selective rebuild keeps old
code, and only a full rebuild proves the spec alone still produces the whole app.

### Modular build

Each module builder works in its **own git worktree**, so its changes can be checked:
`git worktree add -b build/<module> .oneshot/wt/<module> HEAD`. Its brief points at
`.oneshot/wt/<module>/app/` instead of `app/`.

**Full build:**
1. **Scaffold** (in the main checkout): one builder gets the whole spec and sets up only the
   skeleton: stack, a module layout matching the module map, shared contracts and module
   interfaces as real code (types, schemas, stubs). Commit. Everything it wrote is `integration`.
2. **Modules:** one builder per module, in parallel (several Agent calls in one message), each in a
   worktree branched from the scaffold commit. Each gets `ONESHOT.md`, its module file(s) and the
   files its `Depends on:` links to. It implements only its own module, codes against the
   interfaces, and commits in its worktree.
3. **Integrate:** merge, wire up and check (below).

**Selective rebuild:** after the deletion in step 5, one builder per chosen module, in parallel,
each in a worktree branched from the current commit. Same brief as above, plus: the rest of the
app is already in place, and it may read the other modules' code but changes only its own module.

**After the module builders finish** (both cases):
1. For each worktree, `git diff --name-only <base>..build/<module>` lists the paths it touched.
   Paths that belong to another module in the manifest are a boundary violation: redo that
   module with a sharper brief or spec, or if the change really is shared glue, move it to
   `integration`.
2. Merge each branch into the main checkout; module paths don't overlap, so conflicts mean a
   boundary problem.
3. **Integrate:** one builder gets the whole spec and wires the modules together (entry point,
   routing, dependency lists), makes the app build and run, and runs the end-to-end checks. It
   changes only glue code.
4. Update `.oneshot/manifest.json` from the diffs (each module's paths, glue under
   `integration`), remove the worktrees and branches (`git worktree remove`, `git branch -D build/...`).
Then the parent verifies as usual (step 7).

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
- No manifest yet, or it no longer matches the code (paths missing, unknown files): do a full
  rebuild, which writes a fresh manifest.
- Data the app keeps (databases, uploads) that must survive rebuilds: put it outside `app/`
  (e.g. `<project>/data/`) and say so in the spec.
