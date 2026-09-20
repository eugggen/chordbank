# Chord Bank — Cursor Task Prompt

## What this file is

A self-contained single-file HTML app (~2000 lines) for browsing, auditioning, and exploring chord progressions for electronic and jazz production. No frameworks, no build step, no external dependencies except a Google Fonts stylesheet.

---

## Architecture overview

### Data layer

Every progression is an object in the `const PROGRESSIONS = [...]` array. The shape is:

```js
{
  name: "4AM Vamp",
  tags: ["House", "Modal"],          // style/genre tags
  mood: 'Dreamy',                    // one of 8 moods (see below)
  next: ['Amapiano Glow', 'Basement Glow'],  // 2 curated "next" suggestions by name
  accent: 'var(--teal-dim)',         // legacy field — no longer used visually, keep it
  chords: [
    { roman: 'i9', offset: 0, q: 'min9' },
    { roman: 'ii9', offset: 2, q: 'min9' },
  ],
  desc: "The two-chord dorian pump under half of deep house..."
}
```

**`offset`** is semitones from the selected root (0–11). It is always a fixed integer regardless of key — the transposition happens at render time in `chordInfo()`.

**`q`** references a key in `const QUALITIES`, which maps quality names to interval arrays (`iv`) and display suffixes (`suf`):

```js
const QUALITIES = {
  min:     { iv: [0,3,7],        suf: 'm' },
  maj:     { iv: [0,4,7],        suf: '' },
  min7:    { iv: [0,3,7,10],     suf: 'm7' },
  maj7:    { iv: [0,4,7,11],     suf: 'maj7' },
  dom7:    { iv: [0,4,7,10],     suf: '7' },
  min9:    { iv: [0,3,7,10,14],  suf: 'm9' },
  maj9:    { iv: [0,4,7,11,14],  suf: 'maj9' },
  m7b5:    { iv: [0,3,6,10],     suf: 'm7♭5' },
  dom7sus: { iv: [0,5,7,10],     suf: '7sus4' },
  maj7s11: { iv: [0,4,7,11,18],  suf: 'maj7(♯11)' },
  dim7:    { iv: [0,3,6,9],      suf: 'dim7' },
  dom9:    { iv: [0,4,7,10,14],  suf: '9' },
  add9:    { iv: [0,4,7,14],     suf: 'add9' },
  maj6:    { iv: [0,4,7,9],      suf: '6' },
  min6:    { iv: [0,3,7,9],      suf: 'm6' },
  minMaj7: { iv: [0,3,7,11],     suf: 'm(maj7)' },
  sus4:    { iv: [0,5,7],        suf: 'sus4' },
  dom7b9:  { iv: [0,4,7,10,13],  suf: '7♭9' },
  min11:   { iv: [0,3,7,10,14,17], suf: 'm11' },
};
```

### The 8 moods (used by the mood-mode filter)

`Dreamy`, `Uplifting`, `Playful`, `Melancholic`, `Mysterious`, `Warm`, `Tense`, `Dark`

### Key rendering functions

- **`chordInfo(key, chord, localTranspose, localFlip)`** — given the global key (0–11), a chord object, an optional semitone offset (`localTranspose`), and an optional quality-flip boolean (`localFlip`), returns `{ rootPc, quality, tonePcsAll, uniqueTones }`. All pitch-class math happens here.
- **`buildKeyboardEl(octaves, mini, showLabel)`** — builds a two-octave keyboard DOM element. Black-key positions use CSS variable `--bkey-w` and percentage-based `left` so they stay correct at any responsive size without JS knowing pixel widths.
- **`setKeysChord(keysEl, rootPc, tonePcs)`** — marks root (amber dot) and chord tones (teal dots) on a keyboard element.
- **`renderGrid()`** — clears and rebuilds all card DOM from `filtered` (the currently-visible subset of PROGRESSIONS). Reads `state.key`, `state.mode`, `state.tag`. Sets `card.id = 'prog-' + toSlug(prog.name)` for hash navigation.
- **`renderTagRow()`** — rebuilds the tag/mood filter row. In mood mode it uses CSS grid (`tag-row-grid` class + `--mood-count` CSS variable) so all 9 mood buttons fill exactly one row.
- **`toSlug(name)`** — lowercases and hyphenates a progression name for use as a URL fragment.
- **`startChordHold(rootPc, intervals)`** / `releaseHeldChord()` — Web Audio API hold-to-sustain chord playback through a shared limiter bus. Returns a `{ release() }` handle.
- **`saveSetting(name, value)`** / **`loadSetting(name)`** — thin wrappers around `localStorage` for persisting key, mode, tag, instrument, and octave.

### State

```js
let state = { key: 0, mode: 'mood', tag: 'All', locked: null };
let filtered = [];          // current visible progressions, set at top of renderGrid()
let currentInstrument = 'piano';
let octaveOffset = 0;
let heldChord = null;
```

`state.mode` is `'mood'` or `'style'`. Moods is the default.

### Per-card local transposition

Each card in `renderGrid()` has its own closure variables `localTranspose` (semitone int, 0 by default) and `localFlip` (bool). The `TRANSPOSE_STEPS` array defines the buttons:

```js
const TRANSPOSE_STEPS = [
  { label:'+1', steps: 1 }, { label:'+3', steps: 3 },
  { label:'+6', steps: 6 }, { label:'+7', steps: 7 },
  { label:'−3', steps: -3 }, { label:'−5', steps: -5 },
  { label:'−7', steps: -7 }, { label:'flip', steps: 'flip' },
];
```

`flip` inverts chord quality using `FLIP_MAP`. `steps` is a number or the string `'flip'`. Pressing the same button twice resets. `flip` and a semitone shift can both be active at once.

### Quality flip map

```js
const FLIP_MAP = {
  min: 'maj',   maj: 'min',
  min7: 'maj7', maj7: 'min7',
  min9: 'maj9', maj9: 'min9',
  min6: 'maj6', maj6: 'min6',
  minMaj7: 'maj7',
  maj7s11: 'min7',
  m7b5: 'dom7', dom7: 'm7b5',
  min11: 'maj9',
  add9: 'min9',
  // sus4, dom7sus, dom9, dom7b9, dim7 — untouched (dominant/suspended/diminished)
};
```

### Diatonic vs. borrowed — not yet implemented

Progressions have not yet been tagged for diatonic purity. Here is the intended logic and classification for use when implementing:

**Diatonic** means every chord root in the progression falls within the natural major or natural minor scale relative to its tonal center. Specifically: in a minor-tonic progression, roots may only land on offsets 0, 2, 3, 5, 7, 8, 10 (natural minor); in a major-tonic progression, only 0, 2, 4, 5, 7, 9, 11. Chord quality can be whatever — only the root offset matters for this classification.

**Borrowed** means at least one chord root is outside the home scale — ♭II, ♭III (in major), ♭VI (in major), ♭VII (in major), the tritone, or any chromatic root. This includes modal mixture, Phrygian ♭II moves, and secondary dominants with chromatic roots.

A progression should be tagged **both** if it has a borrowed variant and a diatonic reading (rare in this set — when in doubt, tag as borrowed).

### "next" suggestions

Each progression has a `next: [name1, name2]` field with two hand-curated "next progression" names (structural companion suggestions, same-key verse→chorus feel). These names must exactly match another progression's `name` field. Clicking a "next" button pushes a URL hash (`#slug`) and triggers `hashchange`, which scrolls to and flashes the target card teal. If the target is filtered out, it resets `state.tag` to `'All'` first.

### Settings persistence

`localStorage` keys are prefixed `chordbank_`. Currently saved: `key` (0–11), `mode` ('mood'|'style'), `tag` (string), `instrument` (string), `octave` (-2 to 2).

### Responsive layout

- Desktop (>700px): mini keyboards use `--wkey-w: 26px`, bottom bar is a single row.
- Mobile (≤700px): mini keyboards use `--wkey-w: 22px`, bottom bar stacks into: key+octave row / instrument buttons row / readout row.
- Bottom bar is fixed, so `body` has `padding-bottom: 130px`.

---

## Task 1 — Add a diatonic / borrowed / all filter

Add a `harmony` property to each progression object:

```js
harmony: 'diatonic'  // or 'borrowed'
```

Classification rules:
- Determine tonal center polarity from the progression's root chord (offset 0 if present, else first chord). Minor-family quality (`min`, `min7`, `min9`, `m7b5`, `min6`, `minMaj7`, `min11`) → minor tonic. Major-family or dominant → major tonic.
- **Minor tonic diatonic offsets:** 0, 2, 3, 5, 7, 8, 10
- **Major tonic diatonic offsets:** 0, 2, 4, 5, 7, 9, 11
- If every chord's `offset % 12` falls in the allowed set → `'diatonic'`. Otherwise → `'borrowed'`.

After tagging all 74 progressions, add a small two-button toggle (similar to the existing Styles/Moods toggle) that filters by `harmony`. Three states: `'all'` (default), `'diatonic'`, `'borrowed'`. This filter stacks with the existing mood/style/tag filter — both must pass for a progression to show. Save the selected harmony state to `localStorage` as `chordbank_harmony`.

Style it to match the existing `.mode-btn` / `.mode-switch` pattern.

---

## Task 2 — Double the number of progressions (74 → ~148)

Add approximately 74 new progressions, bringing the total to around 148. Follow all existing data conventions exactly:

```js
{
  name: "Unique Name Here",
  tags: ["Genre", "Subgenre"],
  mood: 'Dark',                          // one of the 8 existing moods
  next: ['ExistingProgressionName', 'AnotherExistingName'],  // must match real names
  accent: 'var(--teal-dim)',             // alternate with var(--amber-dim)
  harmony: 'borrowed',                   // 'diatonic' or 'borrowed' per Task 1 rules
  chords: [
    { roman: 'i7', offset: 0, q: 'min7' },
    { roman: '♭VImaj7', offset: 8, q: 'maj7' },
  ],
  desc: "One or two sentences describing the sound and genre context..."
}
```

**Coverage priorities — we need more of:**
- Dark/moody electronic: industrial techno, EBM, cold wave, dark ambient, noise-adjacent
- Deep/hypnotic house subgenres: organic house, afro-tech, melodic house and techno
- More jazz flavors: bebop turnarounds, post-bop, free jazz-adjacent tension, stride
- More world/global: cumbia, baile funk, Afropop, UK grime variants, kuduro
- More broken beat / UK funk / nu-jazz
- Modal and drone-based: Aeolian vamps, Dorian modulations, Lydian floating progressions
- Classical harmony brought into electronic context: Neapolitan sixth, augmented sixth chords, deceptive cadences

**Quality rules:**
- Only use quality names already in `QUALITIES` (list above), or add new ones to `QUALITIES` first with correct `iv` arrays and `suf` strings.
- `next` references must exactly match the `name` of another progression in the final array — check both new and existing names.
- `mood` must be one of the 8 existing values — do not introduce new moods.
- Descriptions should be grounded and characterful, not academic. Write like a producer talking to another producer.
- Vary chord count: 2-chord vamps, 3-chord loops, 4-chord patterns, and occasional 5-chord sequences.
- Bring genuine genre knowledge — don't invent genre names or use non-standard roman numeral notation.

**Roman numeral conventions used in this codebase:**
- Uppercase = major-quality root chord, lowercase = minor-quality
- ♭ prefix for flat-scale-degree roots (♭VII, ♭VI, ♭III, ♭II)
- Quality suffix appended directly: `ii7`, `♭VImaj7`, `V7sus`, `iii9`, `I(add9)`
- Half-diminished: `ii⌀7` (using ⌀ character)

---

## Do not change

- The CSS design system (colors, typography, spacing, responsive breakpoints)
- The audio engine (`startChordHold`, `getLimiterBus`, the six instrument voice builders)
- The keyboard rendering (`buildKeyboardEl`, `setKeysChord`)
- The hash navigation logic (`toSlug`, `hashchange` handler, card `id` attributes)
- The `next` fields on existing progressions
- The `FLIP_MAP` or `TRANSPOSE_STEPS`
- The localStorage key naming convention (`chordbank_*`)
