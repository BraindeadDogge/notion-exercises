# Multi-word exercise design

Date: 2026-04-27

## Overview

Extend the exercise tool to support multiple word slots in one page, optional sentence context per slot, and a no-retry mode.

## URL structure

All params are indexed. Single-word exercises use index 1.

```
exercise.html
  ?prefix1=he&answer1=ll&options1=l,ll&suffix1=o&context1=Say {word} to your friend
  &prefix2=c&answer2=at&options2=at,ot
  &student=alice
  &exercise=lesson-3
  &noRetry=1
```

**Per-slot params** (N = 1, 2, 3…):
- `prefixN` — characters before the blank (may be empty)
- `suffixN` — characters after the blank (may be empty, defaults to empty)
- `answerN` — the correct option value
- `optionsN` — comma-separated list of choices (min 2)
- `contextN` — optional sentence; `{word}` is replaced with the styled word+blank inline

**Global params:**
- `student` — student identifier (required)
- `exercise` — base exercise name (optional; defaults to full word of slot 1). Slots save as `<exercise>_1`, `<exercise>_2`, etc.
- `noRetry=1` — hides all "Try again" buttons

The old flat params (`prefix`, `answer`, `options`, `suffix`) are removed; all embeds use the new indexed format.

## exercise.html behavior

**Parsing:** Scan for `prefix1`, `prefix2`, … up to the first missing index. Require `student` and at least one valid slot (answerN + ≥2 optionsN). Show error if missing.

**Rendering:** Slots stack vertically. Each slot:
1. If `contextN` present: show the sentence with `{word}` replaced by an inline `<span>` containing the styled word (prefix + blank-span + suffix). The blank-span behaves exactly as today.
2. If no context: show the word centered, same as current single-word layout.
3. Options buttons below.
4. Feedback line + optional retry button below options.

**Interaction:** Each slot is independent. Picking an option for slot N:
- Fills that slot's blank, colors it green/red
- Shows feedback for that slot
- Disables that slot's options
- Shows retry button for that slot (unless `noRetry=1`)
- Saves result to JSONBin under key `<exercise>_N` → `{ student: { correct, chosenAnswer, attempts, lastAttempt } }`

Retry resets only that slot (blank, feedback, options re-enabled, attempts counter stays cumulative).

## generator.html changes

**Word slots:** The "Word" card becomes repeatable. Each slot shows:
- Label: "Word 1", "Word 2", …
- Context sentence input (placeholder: `optional — use {word} for the blank position`)
- Existing word-builder (prefix + blank + suffix)
- Options list with correct toggle

"Add word" button appends a new slot. "×" on each slot removes it (minimum 1 slot).

**Details card:** Add a "Disable retry" checkbox that adds `noRetry=1` to the URL.

**URL generation:** Produces indexed params for all slots. Validates all slots before showing output.

## Result saving

Each slot saves independently the moment an answer is chosen, same fetch-then-PUT pattern as today. Key: `<exercise>_N` where N is the 1-based slot index.

## What is NOT changing

- Visual design / styling tokens
- JSONBin integration pattern
- dashboard.html
- Student-facing feedback copy
