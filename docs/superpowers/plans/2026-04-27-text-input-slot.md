# Text-input Slot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a per-slot "type-in" mode to the exercise tool where students type their answer instead of clicking options, with case-insensitive matching.

**Architecture:** Each slot gains a `type` field (`'options'` | `'text'`). `exercise.html` branches on this field in `buildSlot`, `check`/`checkText`, and `reset`. `generator.html` adds a mode toggle per slot card, a `textAnswer` field in slot state, and branches in validation and URL generation. No new files.

**Tech Stack:** Vanilla HTML/CSS/JS. No build step.

---

## File map

| File | Change |
|---|---|
| `exercise.html` | Add `type` to slot parsing; CSS for text input + check button; `buildSlot` branching; new `checkText`; update `reset` |
| `generator.html` | Add `type`/`textAnswer` to slot state; CSS for mode toggle; `renderSlot` branching; `setSlotType`; `updateTextAnswer`; update `updateSlotField`, `generate`, `addSlot` |

---

### Task 1: Update exercise.html

**Files:**
- Modify: `exercise.html`

- [ ] **Step 1: Add CSS for text input and check button**

Add these rules inside the `<style>` block, after the `.opt.wrong { ... }` rule (after line 81):

```css
    .text-input-row {
      display: flex;
      gap: 8px;
      margin-bottom: 1.25rem;
      align-items: center;
    }
    .text-input {
      font-family: Georgia, serif;
      font-size: 28px;
      letter-spacing: 3px;
      border: none;
      border-bottom: 2px solid #ccc;
      background: transparent;
      color: #1a1a1a;
      padding: 4px 8px;
      outline: none;
      width: 180px;
      text-align: center;
      transition: border-color 0.15s;
    }
    .text-input:focus { border-color: #888; }
    .text-input:disabled { color: #aaa; }
    .check-btn {
      font-family: -apple-system, sans-serif;
      font-size: 14px;
      padding: 8px 20px;
      border: 1.5px solid #ccc;
      border-radius: 8px;
      background: #fff;
      color: #555;
      cursor: pointer;
      transition: background 0.15s, border-color 0.15s;
    }
    .check-btn:hover:not(:disabled) { background: #f5f5f5; border-color: #aaa; }
    .check-btn:disabled { cursor: default; color: #bbb; border-color: #eee; }
```

- [ ] **Step 2: Add `type` to slot parsing**

Replace lines 134–143 (the `while` loop body):

```js
  while (p.get(`answer${i}`)) {
    const prefix  = p.get(`prefix${i}`)  ?? '';
    const suffix  = p.get(`suffix${i}`)  ?? '';
    const answer  = p.get(`answer${i}`);
    const options = (p.get(`options${i}`) ?? '').split(',').map(s => s.trim()).filter(Boolean);
    const context = p.get(`context${i}`) ?? '';
    const type    = p.get(`type${i}`) === 'text' ? 'text' : 'options';
    const fullWord = prefix + answer + suffix;
    const exercise = exerciseBase ? `${exerciseBase}_${i}` : `${fullWord}_${i}`;
    slots.push({ prefix, suffix, answer, options, context, type, exercise, attempts: 0 });
    i++;
  }
```

- [ ] **Step 3: Update validation to allow text slots without options**

Replace line 146:

```js
  if (!student || slots.length === 0 || slots.some(s =>
    !s.answer || (s.type === 'options' && s.options.length < 2)
  )) {
    document.body.innerHTML = '<p class="error-msg">Missing parameters.<br>Required: answer1, student; options1 (min 2) for options slots</p>';
  } else {
```

- [ ] **Step 4: Branch `buildSlot` on slot type for the interaction area**

Replace lines 168–170 (the `optBtns` const and the `<div class="options">` line in the return):

Change `buildSlot` so the interaction area is chosen by type. The full updated `buildSlot` function (replace the entire function at lines 159–181):

```js
  function buildSlot(slot, idx) {
    let topHtml;
    if (slot.context && slot.context.includes('{word}')) {
      const inlineWord = `<span class="inline-word">${esc(slot.prefix)}${buildBlank(idx)}${esc(slot.suffix)}</span>`;
      topHtml = `<div class="context-block">${esc(slot.context).replace('{word}', inlineWord)}</div>`;
    } else {
      topHtml = `<div class="word-block"><span>${esc(slot.prefix)}</span>${buildBlank(idx)}<span>${esc(slot.suffix)}</span></div>`;
    }

    let interactionHtml;
    if (slot.type === 'text') {
      interactionHtml = `<div class="text-input-row">
        <input class="text-input" id="text-input-${idx}" type="text" placeholder="type your answer…"
          onkeydown="if(event.key==='Enter')checkText(${idx})">
        <button class="check-btn" id="check-btn-${idx}" onclick="checkText(${idx})">Check</button>
      </div>`;
    } else {
      interactionHtml = `<div class="options">${slot.options.map(o =>
        `<button class="opt" data-choice="${escAttr(o)}" onclick="check(${idx},this,this.dataset.choice)">${esc(o)}</button>`
      ).join('')}</div>`;
    }

    const retryBtn = noRetry ? '' : `<button class="retry" id="retry-${idx}" onclick="reset(${idx})">Try again</button>`;

    return `<div class="slot" id="slot-${idx}">
      ${topHtml}
      ${interactionHtml}
      <div class="feedback" id="fb-${idx}"></div>
      ${retryBtn}
      <div class="saving" id="saving-${idx}"></div>
    </div>`;
  }
```

- [ ] **Step 5: Add `checkText` function**

Add this function after the `check` function (after line 234):

```js
  function checkText(idx) {
    const slot  = slots[idx];
    const input = document.getElementById(`text-input-${idx}`);
    const checkBtn = document.getElementById(`check-btn-${idx}`);
    const choice = input.value;
    if (!choice.trim()) return;
    slot.attempts++;
    input.disabled   = true;
    checkBtn.disabled = true;
    const blank     = document.getElementById(`blank-${idx}`);
    const fb        = document.getElementById(`fb-${idx}`);
    const isCorrect = choice.trim().toLowerCase() === slot.answer.toLowerCase();
    blank.textContent = choice.trim();
    if (isCorrect) {
      blank.classList.add('correct');
      fb.textContent = `Correct! The word is "${slot.prefix}${slot.answer}${slot.suffix}"`;
      fb.className = 'feedback correct';
    } else {
      blank.classList.add('wrong');
      fb.textContent = 'Not quite — try again!';
      fb.className = 'feedback wrong';
    }
    if (!noRetry) document.getElementById(`retry-${idx}`)?.classList.add('show');
    saveResult(idx, choice, isCorrect);
  }
```

- [ ] **Step 6: Update `reset` to handle text slots**

Replace the `reset` function (lines 236–245):

```js
  function reset(idx) {
    const slot = slots[idx];
    if (slot.type === 'text') {
      const input = document.getElementById(`text-input-${idx}`);
      input.value    = '';
      input.disabled = false;
      document.getElementById(`check-btn-${idx}`).disabled = false;
    } else {
      document.querySelectorAll(`#slot-${idx} .opt`).forEach(b => { b.disabled = false; b.className = 'opt'; });
    }
    const blank = document.getElementById(`blank-${idx}`);
    blank.textContent = '_';
    blank.className = 'blank';
    const fb = document.getElementById(`fb-${idx}`);
    fb.textContent = '';
    fb.className = 'feedback';
    document.getElementById(`retry-${idx}`)?.classList.remove('show');
  }
```

- [ ] **Step 7: Manually verify — options slot unchanged**

Open in browser:
```
exercise.html?prefix1=he&answer1=ll&options1=l,ll&suffix1=o&student=test
```
Expected: option buttons shown, clicking "ll" shows correct feedback, retry works.

- [ ] **Step 8: Manually verify — text slot**

Open in browser:
```
exercise.html?prefix1=he&answer1=ll&type1=text&suffix1=o&student=test
```
Expected: text input + "Check" button shown instead of options. Typing "ll" and pressing Enter (or clicking Check) shows correct green feedback. Typing "LL" also counts as correct. Wrong answer shows red. Retry clears input.

- [ ] **Step 9: Manually verify — mixed slots**

Open in browser:
```
exercise.html?prefix1=he&answer1=ll&options1=l,ll&suffix1=o&prefix2=c&answer2=at&type2=text&student=test
```
Expected: slot 1 has option buttons, slot 2 has text input. Both work independently.

- [ ] **Step 10: Commit**

```bash
git add exercise.html
git commit -m "Add text-input slot mode to exercise.html"
```

---

### Task 2: Update generator.html

**Files:**
- Modify: `generator.html`

- [ ] **Step 1: Add CSS for mode toggle**

Add inside the `<style>` block, after `.remove-slot-btn:hover { color: #e05c52; }` (after line 141):

```css
    .mode-toggle {
      display: flex;
      gap: 4px;
      margin-bottom: 1.25rem;
    }
    .mode-btn {
      font-size: 12px;
      font-family: inherit;
      padding: 5px 14px;
      border: 1px solid #e0deda;
      border-radius: 20px;
      background: #fff;
      color: #888;
      cursor: pointer;
      transition: all 0.15s;
    }
    .mode-btn.active { background: #1a1a1a; border-color: #1a1a1a; color: #fff; }
```

- [ ] **Step 2: Add `type` and `textAnswer` to initial slot state**

Replace lines 301–305:

```js
  let wordSlots = [
    { id: 1, prefix: '', suffix: '', context: '', type: 'options', textAnswer: '',
      options: [{ id: 1, value: 'l', isAnswer: false }, { id: 2, value: 'll', isAnswer: true }],
      nextOptId: 3 }
  ];
```

- [ ] **Step 3: Update `renderSlot` — add mode toggle and branch answer section**

Replace the entire `renderSlot` function (lines 314–348):

```js
  function renderSlot(slot, si) {
    const canRemove = wordSlots.length > 1;
    const answerVal = slot.type === 'text'
      ? (slot.textAnswer ?? '').trim()
      : slot.options.find(o => o.isAnswer)?.value?.trim() ?? '';
    const fullWord  = slot.prefix + answerVal + slot.suffix;

    const answerSection = slot.type === 'text' ? `
      <div class="section-label" style="margin-top:1.25rem">Correct answer</div>
      <div class="field" style="margin-bottom:0">
        <input type="text" value="${esc(slot.textAnswer)}" placeholder="correct answer"
          oninput="updateTextAnswer(${slot.id}, this.value)">
      </div>` : `
      <div class="section-label" style="margin-top:1.25rem">Options</div>
      <div class="options-list" id="options-list-${slot.id}">
        ${slot.options.map(opt => renderOption(slot.id, opt)).join('')}
      </div>
      <button class="add-option-btn" onclick="addOption(${slot.id})">+ Add option</button>`;

    return `<div class="card" id="slot-card-${slot.id}">
      <div class="slot-header">
        <div class="section-label" style="margin:0">Word ${si + 1}</div>
        ${canRemove ? `<button class="remove-slot-btn" onclick="removeSlot(${slot.id})" title="Remove word">&times;</button>` : ''}
      </div>

      <div class="mode-toggle">
        <button class="mode-btn ${slot.type === 'options' ? 'active' : ''}" onclick="setSlotType(${slot.id},'options')">Options</button>
        <button class="mode-btn ${slot.type === 'text' ? 'active' : ''}" onclick="setSlotType(${slot.id},'text')">Type-in</button>
      </div>

      <div class="field">
        <label>Context sentence <span style="color:#ccc">(optional — use {word} for the blank position)</span></label>
        <input type="text" value="${esc(slot.context)}" placeholder="e.g. Say {word} to your friend"
          oninput="updateContext(${slot.id}, this.value)">
      </div>

      <div class="word-builder">
        <input type="text" value="${esc(slot.prefix)}" placeholder="he" maxlength="20"
          oninput="updateSlotField(${slot.id},'prefix',this.value)">
        <div class="word-sep">+</div>
        <div class="blank-box ${answerVal ? 'has-answer' : ''}" id="blank-box-${slot.id}">${esc(answerVal) || '_'}</div>
        <div class="word-sep">+</div>
        <input type="text" value="${esc(slot.suffix)}" placeholder="o" maxlength="20"
          oninput="updateSlotField(${slot.id},'suffix',this.value)">
      </div>
      <p class="preview">Preview: <span>${esc(fullWord) || '—'}</span></p>

      ${answerSection}
    </div>`;
  }
```

- [ ] **Step 4: Add `type` and `textAnswer` to `addSlot`**

Replace lines 361–370 (`addSlot` function):

```js
  function addSlot() {
    wordSlots.push({
      id: nextSlotId++, prefix: '', suffix: '', context: '', type: 'options', textAnswer: '',
      options: [{ id: 1, value: '', isAnswer: false }, { id: 2, value: '', isAnswer: true }],
      nextOptId: 3
    });
    renderAllSlots();
    const last = document.querySelector('#slotsContainer .card:last-child');
    if (last) last.scrollIntoView({ behavior: 'smooth', block: 'start' });
  }
```

- [ ] **Step 5: Update `updateSlotField` to handle text-type blank-box**

Replace lines 378–386 (`updateSlotField` function):

```js
  function updateSlotField(slotId, field, value) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (!slot) return;
    slot[field] = value;
    const answerVal = slot.type === 'text'
      ? (slot.textAnswer ?? '').trim()
      : slot.options.find(o => o.isAnswer)?.value?.trim() ?? '';
    const bb = document.getElementById(`blank-box-${slotId}`);
    if (bb) { bb.textContent = answerVal || '_'; bb.className = 'blank-box' + (answerVal ? ' has-answer' : ''); }
    generate();
  }
```

- [ ] **Step 6: Add `setSlotType` and `updateTextAnswer` functions**

Add after `updateContext` (after line 391):

```js
  function setSlotType(slotId, type) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (slot) { slot.type = type; renderAllSlots(); }
  }

  function updateTextAnswer(slotId, value) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (!slot) return;
    slot.textAnswer = value;
    const answerVal = value.trim();
    const bb = document.getElementById(`blank-box-${slotId}`);
    if (bb) { bb.textContent = answerVal || '_'; bb.className = 'blank-box' + (answerVal ? ' has-answer' : ''); }
    generate();
  }
```

- [ ] **Step 7: Update `generate` validation to branch on slot type**

Replace lines 443–451 (the `wordSlots.forEach` validation block):

```js
    wordSlots.forEach((slot, i) => {
      const n = i + 1;
      if (slot.type === 'text') {
        if (!slot.textAnswer.trim()) errors.push(`Word ${n}: the correct answer must not be empty`);
      } else {
        const answer = slot.options.find(o => o.isAnswer)?.value?.trim() ?? '';
        const opts   = slot.options.map(o => o.value.trim()).filter(Boolean);
        if (!answer)         errors.push(`Word ${n}: the correct option must not be empty`);
        if (opts.length < 2) errors.push(`Word ${n}: add at least 2 options`);
        if (slot.options.some(o => !o.value.trim())) errors.push(`Word ${n}: fill in all option values`);
        if (slot.options.some(o => o.value.includes(','))) errors.push(`Word ${n}: option values cannot contain commas`);
      }
    });
```

- [ ] **Step 8: Update `generate` URL building to branch on slot type**

Replace lines 460–470 (the `wordSlots.forEach` URL-building block):

```js
    const params = new URLSearchParams();
    wordSlots.forEach((slot, i) => {
      const n = i + 1;
      if (slot.prefix)  params.set(`prefix${n}`, slot.prefix);
      if (slot.suffix)  params.set(`suffix${n}`, slot.suffix);
      if (slot.context) params.set(`context${n}`, slot.context);
      if (slot.type === 'text') {
        params.set(`answer${n}`, slot.textAnswer.trim());
        params.set(`type${n}`,   'text');
      } else {
        const answer = slot.options.find(o => o.isAnswer).value.trim();
        const opts   = slot.options.map(o => o.value.trim()).filter(Boolean).join(',');
        params.set(`answer${n}`,  answer);
        params.set(`options${n}`, opts);
      }
    });
```

- [ ] **Step 9: Manually verify — mode toggle switches UI**

Open `generator.html`. Default shows Options mode with options list. Click "Type-in" — options list disappears, "Correct answer" field appears. Click "Options" — switches back.

- [ ] **Step 10: Manually verify — text slot URL generation**

Set student ID, switch Word 1 to Type-in, enter "ll" as correct answer, set prefix "he" and suffix "o". Verify generated URL contains `answer1=ll&type1=text` and does NOT contain `options1`.

- [ ] **Step 11: Manually verify — mixed URL opens correctly in exercise.html**

Copy the generated URL from a mixed exercise (one options slot + one text slot) and open it in exercise.html. Verify slot 1 shows option buttons and slot 2 shows text input.

- [ ] **Step 12: Commit**

```bash
git add generator.html
git commit -m "Add text-input slot mode to generator.html"
```
