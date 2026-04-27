# Text-input slot design

Date: 2026-04-27

## Overview

Add a "type-in" slot mode to the exercise tool. Each slot can independently be either the existing options-button mode or a free-text input mode. Matching is case-insensitive.

## URL structure

One new optional param per slot:

- `typeN=text` — slot renders a text input. Absent (or any other value) = options mode. Backward compatible.
- For text-type slots, `optionsN` is omitted entirely.
- All other params unchanged: `prefixN`, `suffixN`, `answerN`, `contextN`, `student`, `exercise`, `noRetry`.

Example mixed exercise:
```
exercise.html
  ?prefix1=he&answer1=ll&options1=l,ll&suffix1=o
  &prefix2=c&answer2=at&type2=text&suffix2=
  &student=alice&exercise=lesson-3
```

## exercise.html behavior

**Slot parsing:** read `typeN` param. If value is `"text"`, slot is text mode; otherwise options mode.

**Text-mode slot rendering:** replace the options buttons with a `<input type="text">` and a "Check" button. The blank display (prefix + blank span + suffix, or context sentence with inline word) is identical to options mode.

**Submission:** triggered by clicking "Check" or pressing Enter. Compare `input.value.trim().toLowerCase()` against `slot.answer.toLowerCase()`.

- Correct: disable input, fill blank green with typed value, show "Correct! The word is …" feedback
- Wrong: disable input, fill blank red with typed value, show "Not quite — try again!" feedback
- Retry (if `noRetry` absent): clear input value, re-enable input, reset blank and feedback
- Save to JSONBin: `chosenAnswer` = typed value (raw, untrimmed). Same structure as options mode.

**Validation:** text-mode slots require only `answerN` + `student`. Missing `optionsN` is not an error for text slots.

## generator.html changes

**Per-slot mode toggle:** two pill buttons "Options" and "Type-in" at the top of each slot card. Default: Options.

**Options mode:** existing UI unchanged.

**Type-in mode:** options list hidden. A single "Correct answer" text field replaces it. The blank-box preview reads from this field (same live-update logic as before).

**State:** each slot in `wordSlots` gains a `type` field: `'options'` (default) or `'text'`.

**URL generation:**
- Options slots: existing behavior (emit `optionsN`, no `typeN`)
- Text slots: emit `typeN=text`, skip `optionsN`

**Validation:**
- Options slots: existing rules (≥2 options, one correct, all filled, no commas)
- Text slots: correct answer field must be non-empty

## What is NOT changing

- Visual styling tokens
- Context sentence rendering
- noRetry behavior
- JSONBin integration pattern
- dashboard.html
- Save queue / race condition handling
