# only-oneshot

A Claude Code skill that treats your **prompt as the source code** and the generated app as a
**compiled binary**.

The usual way to build with an AI agent: "build thing", then a pile of follow-ups like "add
detail" or "don't make the button flicker". The fixes live in a chat history, and the app is stuck
at the level of the model that built it.

With one-shot, every follow-up is written back into a single spec (`ONESHOT.md`). Then the app
folder (or, for a modular app, just the affected modules) is wiped and fresh subagents regenerate it
from that spec alone. The parent agent
checks the new build against the spec's acceptance checks. If a check fails, it rewrites the spec
(never the code) and tries again, up to 3 cycles.

At the end you have a working app **and** a complete prompt that rebuilds it from nothing. When a
better model comes out, run the spec on it and compare the results.

Inspired by [this post from Taelin](https://x.com/victortaelin/status/2120912867276841258).

## Commits and amends

Regenerating the app every round is the point, but it costs tokens. So each round is one of two kinds:

- **Commit (default):** update the spec, delete the code, regenerate it from the spec alone, verify.
  It proves the spec is complete at that snapshot.
- **Amend (cheap):** update the spec the same way, but patch the existing code instead of
  regenerating it. Same checks, far fewer tokens, but the code may drift from what the spec alone
  would produce until the next commit.

Say "amend: ..." or "commit: ..." on any message, or set a policy in `.oneshot/config.json`, e.g.
`{"rounds": "amend", "commit_every": 3}`. With no setting, every round is a commit. A commit (full
rebuild) also runs before the spec is called finished if anything was amended since the last one.

## How the spec is written

- **The end state, not the history.** The spec reads as if it had been written that way from the
  start: no changelog, and no "like the last build".
- **Positive by default.** "Stop the button flickering" becomes "The button keeps a fixed size and
  position across all states". A "don't do X" aimed at one model's quirk is fragile: the next model
  may never make that mistake, and then the line is just noise. Negatives are still allowed when
  the positive version would be clumsy, or when the mistake is common across model families.
- **Removals list what's there.** "Get rid of the reset button" becomes "The controls are the bill
  field, the tip options and the people count."
- **Every requirement has an acceptance check** that the parent verifies with real evidence
  (it runs the app, takes screenshots, runs tests).

## Modular apps

Most apps fit in one `ONESHOT.md` and are always regenerated whole. For a large app the spec
becomes a tree where **each file is one module** with one concern: `ONESHOT.md` holds the overview,
the module map and cross-cutting rules, each module file states its purpose, its interface and
what it depends on, and shared contracts (data model, API shapes) live in `spec/shared/`. The
spec asks for code split the same way, with modules talking only through their interfaces. It
never names file paths; those are the builder's choice.

That makes **selective rebuilds** possible. After each amendment the supervisor looks at what
changed in the spec and decides the scope: an internal change rebuilds one module, an interface
change also rebuilds the modules that depend on it, a shared contract change rebuilds everything
that uses it, and a cross-cutting change rebuilds the whole app. It deletes exactly that code,
then regenerates it.

Each module builder runs in its own **git worktree**. Its diff shows which files it actually
touched; that feeds `.oneshot/manifest.json` (module → code paths, a build output) and catches
builders that wander into another module's code. An integration step then wires the modules
together, and every acceptance check is rerun so regressions in kept modules show up. A full
rebuild still happens before the spec is called finished, since only that proves the spec
alone produces the whole app.

## Install

```sh
git clone https://github.com/SeanHollen/only-oneshot
ln -s "$PWD/only-oneshot/skills/only-oneshot" ~/.claude/skills/only-oneshot
```

Then in Claude Code:

```
/only-oneshot Build a tip calculator web page as a single HTML file...
```

Every later message in that conversation counts as an amendment to the spec. To compare models:
"rebuild this with sonnet and compare". That builds into `.oneshot/runs/` and leaves `app/` alone.

## Project layout

```
<project>/
  ONESHOT.md        the spec (or its top level): spec files are all the builder sees
  spec/             module specs, only for large apps
  .oneshot/manifest.json  module → code paths (modular apps)
  app/              generated app, wiped before every build
  .oneshot/log.md   your messages verbatim, how the spec changed, check results per build
  .oneshot/runs/    side-by-side builds for model comparisons
```

Each round is committed to a local git repo before the wipe, so any build can be restored.

## Example

[`examples/tip-calculator`](examples/tip-calculator) is a real test run with three rounds:
1. The first build from a one-paragraph prompt.
2. "I don't like the big green card, it's way too loud. And get rid of the reset button. Also the
   per-person amount shouldn't be buried."
3. "The custom tip box is too cramped. Also it shouldn't let me type a negative number."

Compare the final `ONESHOT.md` with those complaints to see how they were turned into positive
statements. `log.md` shows each round's check results.

## License

MIT
