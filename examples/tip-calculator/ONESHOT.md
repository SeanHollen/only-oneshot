# Tip Calculator

## Goal
A tip calculator web page. You type in the bill amount, pick a tip percentage (15, 18, 20 or a custom amount) and how many people are splitting it, and it shows the tip, the total and what each person pays.

## Stack
- A single HTML file: `index.html`, with all CSS and JavaScript inline. It opens directly from disk in a browser.

## Requirements

### Inputs
- The controls are the bill field, the tip options and the people count.
- Bill amount: a number field you type into. The bill is always zero or more: the field accepts digits and a decimal point, so a typed minus sign is ignored.
- Tip percentage: choose 15%, 18% or 20%, or enter a custom percentage.
- The custom tip field is roomy: at every screen width, including a 400px-wide phone, its full "Custom" label or placeholder and a typed value such as 12.5 are completely visible.
- Number of people splitting the bill: always a whole number of 1 or more. The field accepts digits only, so a typed minus sign is ignored.

### Results
- The amount each person pays (total ÷ people) is the headline: the largest, most prominent number on the page, shown on its own above or ahead of the other results so it is the first thing you see.
- The tip amount and the total (bill + tip) appear as smaller, secondary figures beneath the per-person amount.
- Results update as the inputs change.

### Look and feel
- Calm and understated. The results sit on the same light, neutral surfaces as the rest of the page (white or a soft gray card), with dark text.
- One muted accent color, used sparingly: the selected tip option and perhaps the per-person figure. Every large area of the page is white or soft gray; color appears only in small touches.
- The layout fits a 400px-wide phone screen and a desktop browser, with touch-friendly controls.

## Acceptance checks
1. `app/index.html` is the whole app, with everything in that one file, and it opens straight from disk.
2. The tip options are 15%, 18% and 20%, plus a field for a custom percentage.
3. Bill $100, 18%, 1 person → tip $18.00, total $118.00, per person $118.00.
4. Bill $100, 20%, 4 people → tip $20.00, total $120.00, per person $30.00.
5. Bill $50, custom 25%, 2 people → tip $12.50, total $62.50, per person $31.25.
6. Changing any input updates the results.
7. The per-person amount is visually the most prominent result: clearly larger than the tip and total, and positioned first among the results.
8. Every large surface (page background, card, results area) is white or soft gray; the accent color appears only on small elements.
9. The controls are exactly the bill field, the tip options and the people count (with any helpers that adjust those, such as +/- buttons).
10. At 400px wide, the custom tip field shows the whole word "Custom" and a typed "12.5" in full.
11. Typing "-20" into the bill field leaves it holding 20; typing "-3" into the people field leaves it holding 3. Every result shown is zero or more.
