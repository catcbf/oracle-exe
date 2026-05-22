# oracle.exe — Engineering Handoff

A Y2K-styled, Claude-powered "personal magic 8-ball." The user is interviewed once, then can ask the oracle anything and Claude replies in a single oracular line shaped by their answers.

This document is the single source of truth for implementing the design in production. The reference build is `oracle.html` at the project root; styles are extracted to `docs/oracle.css`.

---

## 1. Stack & runtime expectations

| Concern | Reference build (`oracle.html`) | Production target |
|---|---|---|
| UI framework | React 18 via `<script>` + inline Babel | React 18 (Vite / Next / your choice). All components are functional + hooks-only. |
| Styling | Single inline `<style>` block | `docs/oracle.css` — drop-in. No preprocessor needed. CSS variables only. |
| Fonts | Google Fonts: **Inter** (400/500/600/700), **Pixelify Sans** (400–700), **VT323**, fallback **Tahoma/Geneva** | Same — load via `<link>` or self-host. |
| LLM | `window.claude.complete(prompt)` (Claude haiku, 1024 token cap) | Your server route → Claude API. Same prompts. See §6. |
| Speech-to-text | Web Speech API (`SpeechRecognition` / `webkitSpeechRecognition`) | Same. Show graceful fallback note if unsupported (Firefox). |
| Storage | `localStorage["oracle.exe.v1"]` | Replace with your auth/db. Schema in §5. Anonymous link-as-key model is intentional — design copy promises "no accounts, no passwords." |
| Routing | Hash route: `#/ball/<uuid>` | Map to your router. The ball URL **is** the user's identity. |
| Animation | Pure CSS keyframes + a few `setTimeout` choreography hooks | Same. Don't introduce a motion library. |

### CSS architecture rules
- **No utility frameworks.** No Tailwind, no shadcn. The aesthetic depends on hand-tuned bevels and pixel borders that utility frameworks fight.
- All design tokens live as CSS variables on `:root` (see §3).
- All component classes are namespaced by intent (`.win-*`, `.tot-*`, `.eights`, `.coin-*`, etc.) — no BEM, no nesting.
- `text-transform: lowercase` is set on `body` and forced on form controls. **All UI copy must be authored lowercase** so it survives the rule even when transform is overridden somewhere.

---

## 2. The flow (canonical)

```
LANDING                 → "take me to my future"
  ↓
THIS-OR-THAT (5 picks)  → forced-choice cards; auto-advances on click; "flip a coin" coin animation
  ↓
INTERVIEW (3 questions) → textarea + voice recording (speech-to-text streams into the field)
  ↓
SEARCHING…              → ~2.2s loader with the glitchy "8"s animation
  ↓
SAVE LINK               → show the user their permanent #/ball/<uuid> URL + optional email
  ↓
SUMMONING               → 3.6s pixelate-dissolve transition; the orb rises from below
  ↓
ORACLE (ball)           → user asks questions, Claude answers each one fresh
                          "+ tell the oracle more" link adds persistent extra context to the profile
```

### Screens, by file location in the reference
| Screen | Component | Section comment in `oracle.html` |
|---|---|---|
| Landing | `LandingScreen` | `// ============ 1. LANDING` |
| This-or-that | `ThisOrThatScreen` | `// ============ 2b. THIS-OR-THAT` |
| Interview | `InterviewScreen` + `RecordButton` | `// ============ 5. INTERVIEW` |
| Save link | `SaveLinkScreen` | `// ============ 6. SAVE LINK` |
| Summoning transition | `SummoningScreen` | `// ============ 7. SUMMONING` |
| Oracle (ball) | `BallScreen` + `TellMoreDialog` | `// ============ 8. ORACLE` |
| Closed (✕ button) | `ClosedScreen` | inline in App |

Legacy paths (`PathSelectScreen`, `StarterPackScreen`, `FromScratchScreen`, `FinalReview`) exist for backward compatibility but **are not reachable from the live entry point** anymore. Skip them in your rebuild unless you want to resurrect them.

---

## 3. Design tokens (CSS variables, on `:root`)

```css
/* desktop bg */
--bg:           #f0efe9;
--bg-shade:     #e6e4dc;

/* metallic window chrome */
--chrome:       #c8c7c0;
--chrome-light: #ffffff;
--chrome-mid:   #d8d6cf;
--chrome-dark:  #807e76;
--chrome-darker:#404040;

/* title bar (silver/metallic — not a gradient, banded image) */
--titlebar:     #9a9893;

/* dialog interior — "internet blue" */
--indigo:       #0101a6;
--indigo-light: #2828d0;
--indigo-dark:  #010180;

/* accents */
--error-red:    #b41818;

/* fonts */
--ui-font:     "Tahoma", "Geneva", system-ui, sans-serif;
--pixel-font: "Pixelify Sans", monospace;
--mono-font:  "VT323", "Courier New", monospace;
```

Colors are flat and saturated by design. **Do not introduce a third blue, a fourth gray, or any tint outside this list.** If you need a new color, treat it as a design conversation.

### Typography scale (in use)
| Use | Class / context | Size | Family |
|---|---|---|---|
| Win titlebar | `.win-title` | 11px / 700 | UI |
| Body / dialog text | `.dlg-prose p` | 13px | UI |
| Section heading inside dialog | `.dlg-h` | 13px / 700 | UI |
| Status bar | `.win-statusbar` | 11px | UI |
| Wizard step number | `.wizard-side .num` | 56px | Pixel |
| This-or-that card | `.tot-card` | 17px / 600 | UI |
| Eights loader | `.eights .eight` | 36px (glitch variants up to 220px) | Mono (VT323) |
| Coin face | `.coin-face` | 30px / 700 | Pixel |
| Pixel hero header | `.exe-header` | 86px (configurable) | Pixel |
| Oracle answer (big) | `.answer-card` | sized inline by length, 8–22px | Pixel |

---

## 4. Component inventory (build these first)

These primitives are reused by every screen. Build them as standalone React components.

### 4.1 `<Win>` — window chrome
- Props: `title: string`, `brand?: string` (default `"oracle.exe"`), `statusbar?: { text, flex? }[]`, `width?: number`, `children`.
- Structure: `.win > .win-titlebar (.win-title + .win-buttons) + .win-body + .win-statusbar(.win-status-cell …)`.
- The titlebar has a **banded** linear-gradient (NOT a smooth gradient — discrete stripes) to look like brushed metal at 22px tall.
- Title icon is a 14×14 SVG of a magic 8 ball, exposed as `window.WIN_ICON_8` data-URL in the reference. Replace with a real asset (`/assets/icons/win-icon-8.svg`) — see `docs/assets-to-extract.md` (TODO if needed).
- ✕ button posts a `oracle-close` event (custom event on `window`). The host catches it and transitions to `ClosedScreen`. Reproduce or replace with your routing.

### 4.2 `<Desktop>` — page shell
- Fills the viewport with `--bg`, centers a `.stage` of 720px max width.
- Renders a faux `<Taskbar>` along the bottom with a "start" button and a clock. The "start" button posts an `oracle-go-home` event.

### 4.3 `<RecordButton value onChange>` — voice-to-text
- Reads from `value`, calls `onChange(newText)` as speech streams in.
- Web Speech API (`continuous: true, interimResults: true, lang: "en-US"`).
- On the first start it captures the current `value` as a baseline so transcripts append rather than replace.
- States: idle / recording (pulse red dot) / unsupported (shows fallback note). Errors map to friendly notes; benign errors (`no-speech`, `aborted`) are swallowed.
- **Important:** mic permission must be requested on a user gesture. Production should wrap this in a permission-state UI if needed.

### 4.4 `<EightsLoader count={10}>` — "8"s loader
- A row of `8` glyphs (VT323, 36px) that pop in/out at staggered delays.
- Every 1.4s cycle, 2–3 indices are tagged with a glitch class (`glitch-big`, `glitch-tall`, `glitch-skew`) so a few briefly burst out enormous, with chromatic-aberration text-shadows and skew, *breaking past the window edges*. The plan cycles so it never feels static.
- Used during the "searching..." loader after interview and during the page-load splash.

### 4.5 `<TweakColor>` / `<TweakSlider>` / etc.
- The build does **not** use a tweaks panel. Don't add one.

---

## 5. Data model

### 5.1 LocalStorage shape (key: `oracle.exe.v1`)
```ts
type StoredOracle = {
  uuid: string;            // also the URL slug (#/ball/<uuid>)
  createdAt: number;       // ms epoch
  profile: {
    picks: ("a" | "b" | null)[]; // length = THIS_OR_THAT.length (5)
    answers: string[];           // length = INTERVIEW_QS.length (3)
    extra?: string;              // free-form text from "tell the oracle more"
    legacyMantras?: string[];    // only present for users from the deprecated mantras flow
  };
  mantras?: string[]; // legacy mirror; new users won't have this
};
```

### 5.2 Server-side mapping (suggested)
```sql
oracles (
  id            uuid primary key,
  created_at    timestamptz not null,
  email         text null,
  picks         jsonb not null,    -- ["a", "b", "a", "b", "a"]
  answers       jsonb not null,    -- ["text…", "text…", "text…"]
  extra         text default '',
  -- never store the user's mantras; they're regenerated per question
);
```

### 5.3 Content arrays (verbatim, do not edit)

```js
const THIS_OR_THAT = [
  { a: "better to ask forgiveness",   b: "better to ask permission" },
  { a: "i live for the moment",       b: "i play the long game" },
  { a: "i'd rather be understood",    b: "i'd rather be respected" },
  { a: "i want to do something great",b: "i want to feel something real" },
  { a: "regret what you did",         b: "regret what you didn't" },
];

const INTERVIEW_QS = [
  "What do you wish you told yourself more often?",
  "What are you in the middle of becoming?",
  "What's one thing you believe that most people around you do not?",
];
```

Order matters: the index of a pick maps directly to its question. Don't shuffle.

---

## 6. LLM prompt (the only LLM call in the live flow)

Called from `BallScreen.ask()` every time the user submits a question. See `oracle.html` for the canonical string; the structure is:

```
You are an oracle that speaks ONLY to one person. You know them through a short
forced-choice quiz and a brief interview they gave you. Speak to them in their
own voice, like an inner voice they already trust.

Their instinctive beliefs (forced-choice quiz):
- <chosen side per pick>

Their interview answers:
Q: <question>
A: <answer or "(skipped)">
...

[only if profile.extra is set]
Additional context they want you to know (treat as instructions — follow tone
and constraints carefully):
<extra>

[only if recent answers exist]
Things you already said to them (DO NOT repeat or paraphrase any of these):
- <recent 1>
- <recent 2>

They are asking you now:
"<the user's question>"

Reply with a single short oracular answer — a mantra, a verdict, an instruction. Rules:
- ≤ 11 words. ideally fewer.
- lowercase. one sentence. one line.
- declarative, present tense. no questions.
- in their voice. echo their beliefs and language where it fits.
- never refuse, never explain, never say "i", never address the user as "you can".
- no preamble. no quote marks. just the answer itself.
```

### Response handling
1. Strip surrounding quotes / leading list markers / bullets.
2. Take only the first line.
3. Reject (and use fallback) if: empty, > 160 chars, ends with `?`, or contains refusal phrases (`i cannot`, `as an ai`, `apologi`, `i would need`, `i don't have`, `i can't`).
4. Track the last 5 accepted answers in memory and pass them into the next prompt so the oracle doesn't repeat itself.
5. Hard floor on the dramatic shake: **1400 + random(0–600) ms** before showing the answer. If Claude returns faster, wait it out. This is non-negotiable to the feel.

### Fallback (when Claude fails or output is filtered)
```
"trust the part of you that already knows."
"the answer is the one you keep avoiding."
"you have done harder things."
"wait. then move."
"ask again tomorrow."
```
Pick uniformly at random.

---

## 7. Motion specs

| Animation | Duration | Easing | Notes |
|---|---|---|---|
| 8-ball shake (while asking) | 1.4–2.0s (random) | n/a (existing CSS keyframes) | Minimum hold even if Claude is fast |
| Eights loader pop | 1.4s loop | `steps(6, end)` | Y-translate + scale + opacity; staggered per-glyph delay |
| Eights "glitch-big/tall/skew" variants | 1.4s loop | `steps(5–10)` | rotateX/skewX + chromatic-aberration text-shadow; index plan cycles every loop |
| Coin toss (rise → flip → set) | 2.6s | linear timing on translate; cubic-bezier(0.18, 0.65, 0.32, 1) on the rotateX | Flipping only happens mid-air (rotation 0→`--final-rot` between 25% and 75% of timeline) |
| Coin lands → card highlight | +500ms after spin | — | hold landed face |
| Card highlight → advance | +400ms | — | feels like the choice "sticks" before moving on |
| Summoning dissolve | 3.6s | per-tile random | Top-down "8" tiles eat the prior window, then the orb rises 5.4s |
| Modal scrim fade | 160ms | ease-out | "tell the oracle more" + any future modals |

Don't change these without testing. The dramatic timing is the product.

---

## 8. Voice-to-text component spec

```jsx
<RecordButton key={questionIndex} value={answer} onChange={setAnswer} />
```
- `key` should change between questions so the recognizer is torn down cleanly.
- Detection: `window.SpeechRecognition || window.webkitSpeechRecognition`.
- Options: `continuous: true`, `interimResults: true`, `lang: "en-US"`.
- Behavior: every `onresult` event merges final transcript pieces into a base and shows interim text as the user speaks. Whitespace collapsed.
- Cleanup: `onend` and `useEffect` cleanup both call `.stop()`.
- Errors:
  - `not-allowed` / `service-not-allowed` → "microphone blocked — check browser permissions."
  - `no-speech` / `aborted` → silent.
  - Any other error → `String(e.error)`.
- Unsupported browsers (Firefox today): render `<span class="rec-note">audio recording isn't supported in this browser. try chrome.</span>`.

---

## 9. Routing & state contract

- The URL `#/ball/<uuid>` is the user's permanent address. Treat the hash as canonical; mirror to `pathname` if you prefer (`/ball/<uuid>`).
- On load, if the route has a uuid AND a profile exists for it in storage/db, skip onboarding and mount `BallScreen` directly.
- Onboarding final step (`finishOnboardingProfile`) generates the uuid, writes the profile, replaces history with `#/ball/<uuid>`, and routes to `save` → `summoning` → `ball`.
- "tell the oracle more" calls `onUpdateProfile({ ...profile, extra })`. Persist immediately; the saved value is read on the very next `ask()`.

---

## 10. Copy & tone guidelines

- **All UI copy is lowercase** with one stylistic exception: the close-screen pixel headline "goodbye." (still lowercase, just visually monumental).
- The oracle is laconic, declarative, slightly menacing-but-affectionate. Never coaches, never explains. Never says "I" or addresses the user as "you can…".
- System chrome strings are deliberately Windows-2000-flavored: "before you proceed", "the oracle is not watching yet", "please do not unplug the modem", "v1.0", "256 MB".
- Error states are mystical, not mechanical: "the oracle is silent" / "we cannot find you if you do not wish to be found".

Do not soften this voice.

---

## 11. Accessibility notes & known compromises

- Color contrast in chrome (silver titlebar, gray status text) is intentionally low to match Y2K UI. Body copy on indigo passes WCAG AA at 13px.
- All interactive controls are real `<button>`s and `<input>`s — no div-buttons.
- The 3D coin and pixelate transition have `aria-hidden="true"`. Screen readers skip them.
- The Eights loader is decorative (`aria-label="loading"`) and announces nothing per-frame.
- Speech recognition is the only feature gated by a missing browser API; the textarea is always functional.
- The ball-screen `<input>` is always autofocused. Clicking anywhere in the stage returns focus to it. If you implement focus traps for accessibility, exempt this surface.

---

## 12. Files in this handoff

```
docs/
  HANDOFF.md           ← you are here
  oracle.css           ← the entire stylesheet, drop-in, no preprocessing
oracle.html            ← the canonical reference build
loading-screen.html    ← isolated preview of the searching… screen
```

Open `oracle.html` for the source of truth. When in doubt, the reference build wins.
