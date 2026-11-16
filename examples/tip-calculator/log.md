# One-shot log

## Round 0
User: "Build a tip calculator web page as a single HTML file. You type in the bill amount, pick a tip percentage (15, 18, 20 or a custom amount) and how many people are splitting it, and it shows the tip, the total and what each person pays."

Spec: wrote ONESHOT.md from the prompt (single-file stack, inputs, results, 6 acceptance checks).

Build cycle 1 (claude-opus-5-5): app/index.html, single file, no external requests.
Verified in headless Chrome at 400px:
1. PASS — single file, opens from file://, zero external requests
2. PASS — 15/18/20 buttons + custom % field
3. PASS — $18.00 / $118.00 / $118.00
4. PASS — $20.00 / $120.00 / $30.00
5. PASS — $12.50 / $62.50 / $31.25
6. PASS — + person → $20.83 per person; 15% → $7.50 / $57.50 / $19.17

## Round 1
User: "I don't like the big green card, it's way too loud. And get rid of the reset button, it's pointless. Also the per-person amount shouldn't be buried with the other numbers, that's the one I actually care about."

Spec: added a Look and feel section (calm, neutral light surfaces, one muted accent used sparingly); made per-person the headline result with tip/total as secondary; stated the page holds only inputs + results with no reset button. Added checks 7-9.

Build cycle 1 (claude-opus-5-5). Verified in headless Chrome at 400px:
1. PASS — single file, file://, zero external requests
2. PASS — 15/18/20 buttons + custom % field
3. PASS — per person $118.00, tip $18.00, total $118.00
4. PASS — $30.00 / $20.00 / $120.00
5. PASS — $31.25 / $12.50 / $62.50
6. PASS — + person → $20.83; 15% → $19.17 / $7.50 / $57.50
7. PASS — "Each person pays" first, 48px; tip/total in small tiles below
8. PASS — white card on soft gray page, muted slate-teal accent only on selected tip + per-person figure
9. PASS — buttons are only 15%/18%/20%/−/+
Note (not a check): custom field placeholder truncates to "Cust" at 400px.

## Round 2
User: "the custom tip box is too cramped, the word Custom gets cut off. Also it shouldn't let me type a negative number of people or a negative bill."

Spec: custom tip field must show the full "Custom" and a typed 12.5 at 400px; bill accepts digits + decimal point only (≥0), people a whole number ≥1 (minus sign ignored); added phone-width layout line. Per the updated skill: rewrote "no reset button" as an exact list of controls, and the color/stack lines and checks 1/8/9 as positive statements. Negation sweep clean. Added checks 10-11.

Skill file changed again mid-round (negatives allowed when the positive is clumsy or the mistake is common across models). Spec is fully positive and clear, so no change needed; kept this build.

Build cycle 1 (claude-opus-5-5). Verified in headless Chrome at 400px (keystroke typing, not fill):
1. PASS — single file, file://, zero external requests
2. PASS — 15/18/20 + custom field
3. PASS — $118.00 / $18.00 / $118.00
4. PASS — $30.00 / $20.00 / $120.00
5. PASS — $31.25 / $12.50 / $62.50
6. PASS — + person → $20.83; 15% → $19.17
7. PASS — "Each person pays" card at top, large; tip/total smaller below
8. PASS — soft gray / white surfaces, muted slate-blue accent on small elements only
9. PASS — buttons: 15%, 18%, 20%, −, +
10. PASS — custom field full width (274px), "Custom" and "12.5" fully visible, no horizontal scroll
11. PASS — typed "-20" → 20, "-3" → 3; all results positive
Note: this build puts the results card above the inputs (builder's choice; spec allows "above or ahead").
