# Multi-word Exercise Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add multi-word slots, optional sentence context, and a no-retry mode to the exercise tool.

**Architecture:** `exercise.html` is rewritten to parse indexed URL params (`prefix1`, `answer1`, … `prefixN`, `answerN`) and render one self-contained slot per word, stacked vertically. `generator.html` is updated to manage multiple word slots in its state and produce the indexed URL format. No new files are created.

**Tech Stack:** Vanilla HTML/CSS/JS, JSONBin REST API. No build step.

---

## File map

| File | Change |
|---|---|
| `exercise.html` | Full rewrite — indexed params, multi-slot render, context sentences, noRetry |
| `generator.html` | Full rewrite of state model and render — multi-slot UI, context input, noRetry checkbox |

---

### Task 1: Rewrite exercise.html

**Files:**
- Modify: `exercise.html`

- [ ] **Step 1: Replace exercise.html with the new implementation**

Replace the entire file content with:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: Georgia, serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      background: #fff;
      padding: 2rem 1rem;
    }
    .slot {
      display: flex;
      flex-direction: column;
      align-items: center;
      width: 100%;
      max-width: 640px;
      padding: 2.5rem 0;
    }
    .slot + .slot { border-top: 1px solid #eee; }
    .word-block {
      display: flex;
      align-items: flex-end;
      gap: 2px;
      font-size: 48px;
      letter-spacing: 4px;
      margin-bottom: 2rem;
      color: #1a1a1a;
    }
    .context-block {
      font-family: Georgia, serif;
      font-size: 26px;
      line-height: 1.6;
      text-align: center;
      margin-bottom: 2rem;
      color: #1a1a1a;
    }
    .inline-word { display: inline; letter-spacing: 2px; }
    .blank {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      min-width: 40px;
      padding: 0 4px;
      border-bottom: 2.5px solid #888;
      color: #1a1a1a;
      transition: color 0.2s, border-color 0.2s;
      letter-spacing: 2px;
      line-height: 1.15;
    }
    .word-block .blank { min-width: 56px; font-size: 48px; }
    .blank.correct { border-color: #3a7d1e; color: #3a7d1e; }
    .blank.wrong   { border-color: #c0392b; color: #c0392b; }
    .options {
      display: flex;
      gap: 12px;
      flex-wrap: wrap;
      justify-content: center;
      margin-bottom: 1.25rem;
    }
    .opt {
      font-family: Georgia, serif;
      font-size: 22px;
      letter-spacing: 2px;
      padding: 10px 32px;
      border: 1.5px solid #ccc;
      border-radius: 10px;
      background: #fff;
      color: #1a1a1a;
      cursor: pointer;
      transition: background 0.15s, border-color 0.15s, transform 0.1s;
    }
    .opt:hover:not(:disabled) { background: #f5f5f5; border-color: #aaa; }
    .opt:active:not(:disabled) { transform: scale(0.97); }
    .opt:disabled { cursor: default; }
    .opt.correct { background: #eaf5e0; border-color: #5aad2e; color: #2e6e10; }
    .opt.wrong   { background: #fdecea; border-color: #e05c52; color: #b02a22; }
    .feedback {
      font-family: -apple-system, sans-serif;
      font-size: 14px;
      min-height: 20px;
      color: #888;
      margin-bottom: 1rem;
      text-align: center;
    }
    .feedback.correct { color: #2e6e10; }
    .feedback.wrong   { color: #b02a22; }
    .retry {
      display: none;
      font-family: -apple-system, sans-serif;
      font-size: 13px;
      padding: 7px 20px;
      border: 1px solid #ddd;
      border-radius: 8px;
      background: #fff;
      color: #555;
      cursor: pointer;
      margin-bottom: 0.5rem;
    }
    .retry:hover { background: #f5f5f5; }
    .retry.show { display: block; }
    .saving {
      font-family: -apple-system, sans-serif;
      font-size: 11px;
      color: #bbb;
      min-height: 16px;
      text-align: center;
    }
    .error-msg {
      font-family: -apple-system, sans-serif;
      font-size: 14px;
      color: #b02a22;
      text-align: center;
      padding: 1rem;
    }
  </style>
</head>
<body>
<script>
  const BIN_ID  = '69eface236566621a8fa7fea';
  const API_KEY = '$2a$10$Cnxn6xpCSfkvNSaRtOd94eMGAUlFzI0FeFdC0IIDtTOc92eWq7PXS';

  const p            = new URLSearchParams(location.search);
  const student      = p.get('student') ?? '';
  const exerciseBase = p.get('exercise') ?? '';
  const noRetry      = p.get('noRetry') === '1';

  const slots = [];
  let i = 1;
  while (p.get(`answer${i}`)) {
    const prefix  = p.get(`prefix${i}`)  ?? '';
    const suffix  = p.get(`suffix${i}`)  ?? '';
    const answer  = p.get(`answer${i}`);
    const options = (p.get(`options${i}`) ?? '').split(',').map(s => s.trim()).filter(Boolean);
    const context = p.get(`context${i}`) ?? '';
    const fullWord = prefix + answer + suffix;
    const exercise = exerciseBase ? `${exerciseBase}_${i}` : `${fullWord}_${i}`;
    slots.push({ prefix, suffix, answer, options, context, exercise, attempts: 0 });
    i++;
  }

  if (!student || slots.length === 0 || slots.some(s => s.options.length < 2 || !s.answer)) {
    document.body.innerHTML = '<p class="error-msg">Missing parameters.<br>Required: answer1, options1 (min 2), student</p>';
  } else {
    document.body.innerHTML = slots.map((slot, idx) => buildSlot(slot, idx)).join('');
  }

  function esc(s) { return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
  function escQ(s) { return String(s).replace(/'/g,"\\'"); }

  function buildBlank(idx) {
    return `<span class="blank" id="blank-${idx}">_</span>`;
  }

  function buildSlot(slot, idx) {
    let topHtml;
    if (slot.context) {
      const inlineWord = `<span class="inline-word">${esc(slot.prefix)}${buildBlank(idx)}${esc(slot.suffix)}</span>`;
      topHtml = `<div class="context-block">${slot.context.replace('{word}', inlineWord)}</div>`;
    } else {
      topHtml = `<div class="word-block"><span>${esc(slot.prefix)}</span>${buildBlank(idx)}<span>${esc(slot.suffix)}</span></div>`;
    }

    const optBtns = slot.options.map(o =>
      `<button class="opt" onclick="check(${idx},this,'${escQ(o)}')">${esc(o)}</button>`
    ).join('');

    const retryBtn = noRetry ? '' : `<button class="retry" id="retry-${idx}" onclick="reset(${idx})">Try again</button>`;

    return `<div class="slot" id="slot-${idx}">
      ${topHtml}
      <div class="options">${optBtns}</div>
      <div class="feedback" id="fb-${idx}"></div>
      ${retryBtn}
      <div class="saving" id="saving-${idx}"></div>
    </div>`;
  }

  async function saveResult(idx, chosen, isCorrect) {
    const slot   = slots[idx];
    const saving = document.getElementById(`saving-${idx}`);
    saving.textContent = 'saving…';
    try {
      const getRes  = await fetch(`https://api.jsonbin.io/v3/b/${BIN_ID}/latest`, { headers: { 'X-Master-Key': API_KEY } });
      const data    = await getRes.json();
      const current = data.record ?? {};
      if (!current[slot.exercise]) current[slot.exercise] = {};
      current[slot.exercise][student] = {
        correct: isCorrect, chosenAnswer: chosen,
        attempts: slot.attempts, lastAttempt: new Date().toISOString()
      };
      await fetch(`https://api.jsonbin.io/v3/b/${BIN_ID}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json', 'X-Master-Key': API_KEY },
        body: JSON.stringify(current)
      });
      saving.textContent = '';
    } catch {
      saving.textContent = 'could not save result';
    }
  }

  function check(idx, btn, choice) {
    const slot = slots[idx];
    slot.attempts++;
    document.querySelectorAll(`#slot-${idx} .opt`).forEach(b => b.disabled = true);
    const blank     = document.getElementById(`blank-${idx}`);
    const fb        = document.getElementById(`fb-${idx}`);
    const isCorrect = choice === slot.answer;
    blank.textContent = choice;
    if (isCorrect) {
      btn.classList.add('correct');
      blank.classList.add('correct');
      fb.textContent = `Correct! The word is "${slot.prefix}${slot.answer}${slot.suffix}"`;
      fb.className = 'feedback correct';
    } else {
      btn.classList.add('wrong');
      blank.classList.add('wrong');
      fb.textContent = 'Not quite — try again!';
      fb.className = 'feedback wrong';
    }
    if (!noRetry) document.getElementById(`retry-${idx}`)?.classList.add('show');
    saveResult(idx, choice, isCorrect);
  }

  function reset(idx) {
    document.querySelectorAll(`#slot-${idx} .opt`).forEach(b => { b.disabled = false; b.className = 'opt'; });
    const blank = document.getElementById(`blank-${idx}`);
    blank.textContent = '_';
    blank.className = 'blank';
    const fb = document.getElementById(`fb-${idx}`);
    fb.textContent = '';
    fb.className = 'feedback';
    document.getElementById(`retry-${idx}`)?.classList.remove('show');
  }
</script>
</body>
</html>
```

- [ ] **Step 2: Manually verify — single word, no context**

Open in browser:
```
exercise.html?prefix1=he&answer1=ll&options1=l,ll&suffix1=o&student=test
```
Expected: centered "he___o", two option buttons "l" and "ll". Clicking "ll" shows green, "Correct! The word is "hello"". Retry button appears.

- [ ] **Step 3: Manually verify — single word with context**

Open in browser:
```
exercise.html?prefix1=he&answer1=ll&options1=l,ll&suffix1=o&context1=Say {word} to your friend&student=test
```
Expected: sentence "Say he___o to your friend" with blank inline, options below.

- [ ] **Step 4: Manually verify — two words**

Open in browser:
```
exercise.html?prefix1=he&answer1=ll&options1=l,ll&suffix1=o&prefix2=c&answer2=at&options2=at,ot&student=test
```
Expected: two slots stacked, separated by a thin line. Each has independent options and feedback.

- [ ] **Step 5: Manually verify — noRetry**

Open in browser:
```
exercise.html?prefix1=he&answer1=ll&options1=l,ll&suffix1=o&student=test&noRetry=1
```
Expected: after clicking an option, no "Try again" button appears.

- [ ] **Step 6: Commit**

```bash
git add exercise.html
git commit -m "Rewrite exercise.html: multi-slot, context sentences, noRetry"
```

---

### Task 2: Rewrite generator.html

**Files:**
- Modify: `generator.html`

- [ ] **Step 1: Replace generator.html with the new implementation**

Replace the entire file content with:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Exercise link generator</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: #f5f4f0;
      color: #1a1a1a;
      min-height: 100vh;
      padding: 2.5rem 1rem;
    }
    .container { max-width: 560px; margin: 0 auto; }
    .page-title { font-size: 22px; font-weight: 500; margin-bottom: 0.25rem; letter-spacing: -0.3px; }
    .page-sub { font-size: 13px; color: #888; margin-bottom: 2.5rem; }

    .card {
      background: #fff;
      border: 1px solid #e8e6e1;
      border-radius: 14px;
      padding: 1.5rem;
      margin-bottom: 1rem;
    }
    .slot-card { position: relative; }

    .section-label {
      font-size: 11px;
      font-weight: 500;
      letter-spacing: 0.6px;
      text-transform: uppercase;
      color: #aaa;
      margin-bottom: 1rem;
    }
    .slot-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 1rem;
    }
    .slot-header .section-label { margin-bottom: 0; }

    .word-builder {
      display: flex;
      align-items: center;
      gap: 6px;
      margin-bottom: 1.25rem;
    }
    .word-builder input[type=text] {
      font-family: Georgia, serif;
      font-size: 28px;
      letter-spacing: 3px;
      width: 120px;
      border: none;
      border-bottom: 2px solid #ddd;
      border-radius: 0;
      background: transparent;
      color: #1a1a1a;
      padding: 4px 6px;
      outline: none;
      text-align: center;
      transition: border-color 0.15s;
    }
    .word-builder input[type=text]:focus { border-color: #888; }
    .word-builder input[type=text]::placeholder { color: #ccc; font-size: 16px; letter-spacing: 1px; }
    .blank-box {
      font-family: Georgia, serif;
      font-size: 28px;
      width: 52px;
      height: 44px;
      border-bottom: 2.5px solid #bbb;
      display: flex;
      align-items: center;
      justify-content: center;
      color: #ccc;
      flex-shrink: 0;
    }
    .blank-box.has-answer { color: #1a1a1a; border-color: #555; }
    .word-sep { font-size: 20px; color: #ccc; flex-shrink: 0; user-select: none; }

    .preview { font-family: Georgia, serif; font-size: 13px; color: #aaa; margin-bottom: 0; }
    .preview span { color: #555; font-size: 14px; }

    .options-list { display: flex; flex-direction: column; gap: 8px; margin-bottom: 1rem; }
    .option-row { display: flex; align-items: center; gap: 10px; }
    .option-row input[type=text] {
      font-family: Georgia, serif;
      font-size: 18px;
      letter-spacing: 2px;
      flex: 1;
      border: 1px solid #e0deda;
      border-radius: 8px;
      padding: 8px 12px;
      background: #faf9f7;
      color: #1a1a1a;
      outline: none;
      transition: border-color 0.15s, background 0.15s;
    }
    .option-row input[type=text]:focus { border-color: #aaa; background: #fff; }
    .option-row input[type=text].is-answer { border-color: #5aad2e; background: #f2faea; }

    .correct-toggle {
      font-size: 11px;
      padding: 5px 10px;
      border-radius: 20px;
      border: 1px solid #ddd;
      background: #fff;
      color: #999;
      cursor: pointer;
      white-space: nowrap;
      transition: all 0.15s;
      flex-shrink: 0;
      user-select: none;
    }
    .correct-toggle.selected { background: #eaf5e0; border-color: #5aad2e; color: #2e6e10; font-weight: 500; }

    .remove-btn {
      background: none;
      border: none;
      color: #ccc;
      cursor: pointer;
      font-size: 18px;
      line-height: 1;
      padding: 4px;
      flex-shrink: 0;
      transition: color 0.15s;
    }
    .remove-btn:hover { color: #e05c52; }

    .remove-slot-btn {
      background: none;
      border: none;
      color: #ccc;
      cursor: pointer;
      font-size: 20px;
      line-height: 1;
      padding: 2px 6px;
      transition: color 0.15s;
    }
    .remove-slot-btn:hover { color: #e05c52; }

    .add-option-btn {
      font-size: 13px;
      color: #888;
      background: none;
      border: 1px dashed #ddd;
      border-radius: 8px;
      padding: 8px 16px;
      cursor: pointer;
      width: 100%;
      transition: border-color 0.15s, color 0.15s;
    }
    .add-option-btn:hover { border-color: #aaa; color: #555; }

    .add-slot-btn {
      width: 100%;
      font-size: 14px;
      font-family: inherit;
      color: #888;
      background: none;
      border: 1px dashed #ddd;
      border-radius: 14px;
      padding: 12px;
      cursor: pointer;
      margin-bottom: 1rem;
      transition: border-color 0.15s, color 0.15s;
    }
    .add-slot-btn:hover { border-color: #aaa; color: #555; }

    .field { margin-bottom: 1rem; }
    .field:last-child { margin-bottom: 0; }
    .field label { display: block; font-size: 12px; color: #888; margin-bottom: 5px; }
    .field input[type=text] {
      width: 100%;
      font-size: 14px;
      border: 1px solid #e0deda;
      border-radius: 8px;
      padding: 9px 12px;
      background: #faf9f7;
      color: #1a1a1a;
      outline: none;
      font-family: inherit;
      transition: border-color 0.15s, background 0.15s;
    }
    .field input[type=text]:focus { border-color: #aaa; background: #fff; }

    .checkbox-row {
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 14px;
      color: #555;
      cursor: pointer;
      user-select: none;
    }
    .checkbox-row input[type=checkbox] { width: 15px; height: 15px; cursor: pointer; accent-color: #1a1a1a; }

    .output-card {
      background: #fff;
      border: 1.5px solid #e8e6e1;
      border-radius: 14px;
      padding: 1.25rem 1.5rem;
      margin-bottom: 1rem;
      display: none;
    }
    .output-card.visible { display: block; }
    .output-label {
      font-size: 11px;
      font-weight: 500;
      letter-spacing: 0.6px;
      text-transform: uppercase;
      color: #aaa;
      margin-bottom: 0.75rem;
    }
    .url-box { display: flex; align-items: center; gap: 10px; }
    .url-text {
      flex: 1;
      font-family: 'Menlo', 'Monaco', monospace;
      font-size: 12px;
      color: #444;
      background: #f5f4f0;
      border: 1px solid #e0deda;
      border-radius: 8px;
      padding: 9px 12px;
      word-break: break-all;
      line-height: 1.5;
      min-height: 40px;
    }
    .copy-btn {
      font-size: 13px;
      font-family: inherit;
      padding: 9px 18px;
      border: 1px solid #e0deda;
      border-radius: 8px;
      background: #fff;
      color: #555;
      cursor: pointer;
      white-space: nowrap;
      transition: all 0.15s;
      flex-shrink: 0;
    }
    .copy-btn:hover { background: #f5f4f0; border-color: #bbb; }
    .copy-btn.copied { background: #eaf5e0; border-color: #5aad2e; color: #2e6e10; }

    .validation-msg { font-size: 12px; color: #e05c52; margin-top: 0.5rem; min-height: 16px; }
  </style>
</head>
<body>
<div class="container">

  <h1 class="page-title">Exercise link generator</h1>
  <p class="page-sub">Fill in the details — get a ready-to-embed URL</p>

  <!-- Base URL -->
  <div class="card">
    <div class="section-label">Base URL</div>
    <div class="field" style="margin:0">
      <label>Your hosted exercise.html URL</label>
      <input type="text" id="baseUrl" placeholder="https://yourusername.github.io/repo/exercise.html" oninput="saveBase(); generate()">
    </div>
  </div>

  <!-- Word slots -->
  <div id="slotsContainer"></div>
  <button class="add-slot-btn" onclick="addSlot()">+ Add another word</button>

  <!-- Details -->
  <div class="card">
    <div class="section-label">Details</div>
    <div class="field">
      <label>Student ID</label>
      <input type="text" id="studentId" placeholder="e.g. alice" oninput="generate()">
    </div>
    <div class="field">
      <label>Exercise name <span style="color:#ccc">(optional — auto-generated from word 1)</span></label>
      <input type="text" id="exerciseName" placeholder="leave blank to auto-generate" oninput="generate()">
    </div>
    <div class="field" style="margin-bottom:0">
      <label class="checkbox-row">
        <input type="checkbox" id="noRetry" onchange="generate()">
        Disable retry (student can't try again after answering)
      </label>
    </div>
  </div>

  <!-- Output -->
  <div class="validation-msg" id="validationMsg"></div>
  <div class="output-card" id="outputCard">
    <div class="output-label">Generated link</div>
    <div class="url-box">
      <div class="url-text" id="urlText"></div>
      <button class="copy-btn" id="copyBtn" onclick="copyUrl()">Copy</button>
    </div>
  </div>

</div>
<script>
  // ── helpers ─────────────────────────────────────────────────────────────────
  function esc(s) { return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
  function escQ(s) { return String(s).replace(/'/g,"\\'"); }

  // ── state ───────────────────────────────────────────────────────────────────
  let wordSlots = [
    { id: 1, prefix: '', suffix: '', context: '',
      options: [{ id: 1, value: 'l', isAnswer: false }, { id: 2, value: 'll', isAnswer: true }],
      nextOptId: 3 }
  ];
  let nextSlotId = 2;

  // ── render ───────────────────────────────────────────────────────────────────
  function renderAllSlots() {
    document.getElementById('slotsContainer').innerHTML =
      wordSlots.map((slot, si) => renderSlot(slot, si)).join('');
    generate();
  }

  function renderSlot(slot, si) {
    const canRemove = wordSlots.length > 1;
    const answerVal = slot.options.find(o => o.isAnswer)?.value?.trim() ?? '';
    const fullWord  = slot.prefix + answerVal + slot.suffix;

    return `<div class="card slot-card" id="slot-card-${slot.id}">
      <div class="slot-header">
        <div class="section-label" style="margin:0">Word ${si + 1}</div>
        ${canRemove ? `<button class="remove-slot-btn" onclick="removeSlot(${slot.id})" title="Remove word">×</button>` : ''}
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

      <div class="section-label" style="margin-top:1.25rem">Options</div>
      <div class="options-list" id="options-list-${slot.id}">
        ${slot.options.map(opt => renderOption(slot.id, opt)).join('')}
      </div>
      <button class="add-option-btn" onclick="addOption(${slot.id})">+ Add option</button>
    </div>`;
  }

  function renderOption(slotId, opt) {
    return `<div class="option-row">
      <input type="text" class="${opt.isAnswer ? 'is-answer' : ''}" value="${esc(opt.value)}" placeholder="option"
        oninput="updateOption(${slotId},${opt.id},this.value)">
      <button class="correct-toggle ${opt.isAnswer ? 'selected' : ''}" onclick="setAnswer(${slotId},${opt.id})">
        ${opt.isAnswer ? '✓ correct' : 'mark correct'}
      </button>
      <button class="remove-btn" onclick="removeOption(${slotId},${opt.id})" title="Remove">×</button>
    </div>`;
  }

  // ── slot actions ─────────────────────────────────────────────────────────────
  function addSlot() {
    wordSlots.push({
      id: nextSlotId++, prefix: '', suffix: '', context: '',
      options: [{ id: 1, value: '', isAnswer: false }, { id: 2, value: '', isAnswer: true }],
      nextOptId: 3
    });
    renderAllSlots();
    const last = document.querySelector('.slot-card:last-child');
    if (last) last.scrollIntoView({ behavior: 'smooth', block: 'start' });
  }

  function removeSlot(slotId) {
    if (wordSlots.length <= 1) return;
    wordSlots = wordSlots.filter(s => s.id !== slotId);
    renderAllSlots();
  }

  function updateSlotField(slotId, field, value) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (!slot) return;
    slot[field] = value;
    const answerVal = slot.options.find(o => o.isAnswer)?.value?.trim() ?? '';
    const bb = document.getElementById(`blank-box-${slotId}`);
    if (bb) { bb.textContent = answerVal || '_'; bb.className = 'blank-box' + (answerVal ? ' has-answer' : ''); }
    generate();
  }

  function updateContext(slotId, value) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (slot) { slot.context = value; generate(); }
  }

  // ── option actions ────────────────────────────────────────────────────────────
  function addOption(slotId) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (!slot) return;
    slot.options.push({ id: slot.nextOptId++, value: '', isAnswer: false });
    renderAllSlots();
    const inputs = document.querySelectorAll(`#options-list-${slotId} input`);
    if (inputs.length) inputs[inputs.length - 1].focus();
  }

  function removeOption(slotId, optId) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (!slot || slot.options.length <= 2) return;
    const wasAnswer = slot.options.find(o => o.id === optId)?.isAnswer;
    slot.options = slot.options.filter(o => o.id !== optId);
    if (wasAnswer && slot.options.length) slot.options[0].isAnswer = true;
    renderAllSlots();
  }

  function updateOption(slotId, optId, value) {
    const slot = wordSlots.find(s => s.id === slotId);
    const opt  = slot?.options.find(o => o.id === optId);
    if (opt) { opt.value = value; generate(); }
  }

  function setAnswer(slotId, optId) {
    const slot = wordSlots.find(s => s.id === slotId);
    if (!slot) return;
    slot.options.forEach(o => o.isAnswer = o.id === optId);
    renderAllSlots();
  }

  // ── URL generation ────────────────────────────────────────────────────────────
  function generate() {
    const base    = document.getElementById('baseUrl').value.trim();
    const student = document.getElementById('studentId').value.trim();
    const exName  = document.getElementById('exerciseName').value.trim();
    const noRetry = document.getElementById('noRetry').checked;
    const validationMsg = document.getElementById('validationMsg');
    const outputCard    = document.getElementById('outputCard');

    const errors = [];
    if (!base)    errors.push('Add the base URL of exercise.html');
    if (!student) errors.push('Add a student ID');

    wordSlots.forEach((slot, i) => {
      const n       = i + 1;
      const answer  = slot.options.find(o => o.isAnswer)?.value?.trim() ?? '';
      const opts    = slot.options.map(o => o.value.trim()).filter(Boolean);
      if (!answer)          errors.push(`Word ${n}: mark one option as correct`);
      if (opts.length < 2)  errors.push(`Word ${n}: add at least 2 options`);
      if (slot.options.some(o => !o.value.trim())) errors.push(`Word ${n}: fill in all option values`);
    });

    if (errors.length) {
      validationMsg.textContent = errors[0];
      outputCard.classList.remove('visible');
      return;
    }
    validationMsg.textContent = '';

    const params = new URLSearchParams();
    wordSlots.forEach((slot, i) => {
      const n      = i + 1;
      const answer = slot.options.find(o => o.isAnswer).value.trim();
      const opts   = slot.options.map(o => o.value.trim()).join(',');
      if (slot.prefix)  params.set(`prefix${n}`,  slot.prefix);
      if (slot.suffix)  params.set(`suffix${n}`,  slot.suffix);
      params.set(`answer${n}`,  answer);
      params.set(`options${n}`, opts);
      if (slot.context) params.set(`context${n}`, slot.context);
    });
    params.set('student', student);
    if (exName)  params.set('exercise', exName);
    if (noRetry) params.set('noRetry', '1');

    const url = `${base}?${params.toString()}`;
    document.getElementById('urlText').textContent = url;
    outputCard.classList.add('visible');
    const copyBtn = document.getElementById('copyBtn');
    copyBtn.textContent = 'Copy';
    copyBtn.className = 'copy-btn';
  }

  function copyUrl() {
    const url = document.getElementById('urlText').textContent;
    navigator.clipboard.writeText(url).then(() => {
      const btn = document.getElementById('copyBtn');
      btn.textContent = 'Copied!';
      btn.className = 'copy-btn copied';
      setTimeout(() => { btn.textContent = 'Copy'; btn.className = 'copy-btn'; }, 2000);
    });
  }

  function saveBase() {
    const v = document.getElementById('baseUrl').value.trim();
    if (v) localStorage.setItem('exerciseBaseUrl', v);
  }

  // ── init ──────────────────────────────────────────────────────────────────────
  const savedBase = localStorage.getItem('exerciseBaseUrl');
  if (savedBase) document.getElementById('baseUrl').value = savedBase;
  renderAllSlots();
</script>
</body>
</html>
```

- [ ] **Step 2: Manually verify — single word flow**

Open `generator.html`. Fill in base URL, student ID. Enter prefix "he", suffix "o", options "l" and "ll" (mark "ll" correct). Check that the output URL uses `prefix1=he&answer1=ll&options1=l,ll&suffix1=o&student=...`.

- [ ] **Step 3: Manually verify — add second word**

Click "+ Add another word". Fill in prefix "c", answer "at" (mark correct), add option "ot". Verify URL contains `prefix2=c&answer2=at&options2=at,ot` (or similar).

- [ ] **Step 4: Manually verify — context sentence**

In Word 1, enter context `Say {word} to your friend`. Verify URL contains `context1=Say+%7Bword%7D+to+your+friend`.

- [ ] **Step 5: Manually verify — noRetry checkbox**

Check "Disable retry". Verify URL contains `noRetry=1`. Open the generated URL in exercise.html — confirm no retry button after answering.

- [ ] **Step 6: Manually verify — remove slot**

Click "+ Add another word" twice (3 slots total). Click × on Word 2. Verify Word 3 becomes Word 2 in labels and URL uses `answer1` and `answer2` without gaps.

- [ ] **Step 7: Commit**

```bash
git add generator.html
git commit -m "Rewrite generator.html: multi-slot UI, context sentences, noRetry checkbox"
```
