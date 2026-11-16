# one-shot

A Claude Code skill that treats your **prompt as the source code** and the generated app as a
**compiled binary**.

The usual way to build with an AI agent: "build thing", then a pile of follow-ups like "add
detail" or "don't make the button flicker". The fixes live in a chat history, and the app is stuck
at the level of the model that built it.

With one-shot, every follow-up is written back into a single spec (`ONESHOT.md`). Then the app
folder is wiped and a fresh subagent rebuilds the whole app from that spec alone. The parent agent
checks the new build against the spec's acceptance checks. If a check fails, it rewrites the spec
(never the code) and tries again, up to 3 cycles.

At the end you have a working app **and** a complete prompt that rebuilds it from nothing. When a
better model comes out, run the spec on it and compare the results.

Inspired by [this post from Taelin](https://x.com/victortaelin/status/2120912867276841258).

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

## Install

```sh
git clone https://github.com/SeanHollen/one-shot
ln -s "$PWD/one-shot/skills/one-shot" ~/.claude/skills/one-shot
```

Then in Claude Code:

```
/one-shot Build a tip calculator web page as a single HTML file...
```

Every later message in that conversation counts as an amendment to the spec. To compare models:
"rebuild this with sonnet and compare". That builds into `.oneshot/runs/` and leaves `app/` alone.

## Project layout

```
<project>/
  ONESHOT.md        the spec: the only thing the builder subagent sees
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
