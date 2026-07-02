# Design Doc: Count-Up Timer with Sound

## 1. Overview

A single-page web app with a numeric timer display and three controls: **Start**, **Pause**, **Stop**. On Start, the timer runs a 5-second countdown ("prep"), then counts up indefinitely, playing a generated musical sequence in sync with each second. Pause freezes the timer and audio; Stop resets everything to the initial state.

Notes are played using real **piano sample playback** (not synthesized oscillator tones), via a small JS sample-player library that loads recorded piano notes and pitch-maps them on demand.

## 2. UI Spec

```
┌─────────────────────────────┐
│                               │
│            00                │  <- large numeric display
│                               │
│   [ Start ]  [ Pause ]  [Stop]│
│                               │
│   Status: Ready               │  <- small status line
└─────────────────────────────┘
```

- **Display**: large monospace number. Shows `5,4,3,2,1,0` during prep, then `1,2,3,4…` while counting up.
- **Start**: begins a fresh run from `PREP` if timer is idle/stopped; **resumes** from the current value if paused.
- **Pause**: freezes the display and silences audio at the current second; state is retained.
- **Stop**: halts everything, cancels any scheduled audio, and resets display to a blank/`00` idle state.
- **Auto-stop at 10 minutes**: if count-up reaches 600 seconds without manual Stop, the app stops itself the same way and the status line reflects this (e.g. "Stopped — 10 minute limit reached") rather than the generic "Ready" idle message, so the user understands why it stopped.
- Button enabled/disabled states:
  | State | Start | Pause | Stop |
  |---|---|---|---|
  | Idle | enabled | disabled | disabled |
  | Prep/Running | disabled | enabled | enabled |
  | Paused | enabled ("Resume") | disabled | enabled |

## 3. Timer State Machine

States: `IDLE → PREP → COUNTING_UP`, with `PAUSED` reachable from `PREP` or `COUNTING_UP`, and `STOP` returning to `IDLE` from anywhere.

```
IDLE --start--> PREP (5..0) --auto after 0--> COUNTING_UP (1,2,3,...)
PREP --pause--> PAUSED(prep, n) --start--> PREP resumes at n
COUNTING_UP --pause--> PAUSED(count, n) --start--> COUNTING_UP resumes at n
COUNTING_UP --600s count-up elapsed, no stop--> auto-stop --> IDLE
(any state) --stop--> IDLE (display reset, audio cancelled)
```

**Design assumption:** Pause does not reset progress. Resuming continues the countdown/count-up and the music sequence exactly where it left off, so audio phase and displayed number never desync.

**Auto-termination:** if the count-up phase reaches 600 seconds (10 minutes) of elapsed *running* time without Stop being clicked, the app automatically performs the same action as Stop — halts the timer, cancels all scheduled/playing notes, and returns to `IDLE`. Time spent paused does not count toward the 10 minutes (the countdown is based on `elapsedSoFar` accumulated only while `COUNTING_UP`, consistent with the elapsed-time clock model in §4). The prep countdown (5→0) is not counted toward this limit — the 10 minutes starts once count-up begins.

## 4. Timing Implementation Notes

`setInterval` alone drifts over time and is not accurate enough to keep visuals and audio in sync. Recommended approach:

- Use a single `AudioContext` as the authoritative clock (`audioContext.currentTime`).
- On Start, record a `startTime = audioContext.currentTime - elapsedSoFar` (elapsedSoFar = 0 on fresh start, or the paused offset on resume).
- Drive the visible number from `Math.floor(audioContext.currentTime - startTime)` on each animation frame (`requestAnimationFrame`) or a 100–250ms polling interval — not from counting `setInterval` ticks.
- Schedule notes ahead of time (a "look-ahead scheduler," e.g. 100ms window, standard pattern for Web Audio timing) rather than firing an oscillator inside a `setInterval` callback, to avoid jitter.
- On Pause, store `elapsedSoFar = audioContext.currentTime - startTime`, suspend the `AudioContext` (or just stop scheduling and cancel pending oscillators), and stop the RAF loop.
- On Stop, cancel all scheduled/playing oscillators, reset `elapsedSoFar = 0`, and return to `IDLE`.
- **10-minute auto-stop**: on each display update (or scheduler tick) while `COUNTING_UP`, check `elapsedSoFar >= 600`; if so, trigger the same cleanup as a Stop click (cancel scheduled notes, reset `elapsedSoFar = 0`, transition to `IDLE`). Simpler alternative: schedule a single `setTimeout`/timer for 600s at the moment count-up begins, and clear/reschedule it appropriately on Pause/Resume so it reflects only running time.

## 5. Audio Design

Notes are played via **piano sample playback** rather than oscillator synthesis. Recommended library: [`smplr`](https://github.com/danigb/smplr) (lightweight, no build-step friction) using its built-in `Piano` instrument (Salamander Grand Piano samples, CC-BY licensed, loaded from a CDN). Alternatives: `soundfont-player` or Tone.js `Sampler` with a soundfont — same integration shape.

```js
import { Piano } from 'smplr';

const audioContext = new AudioContext();
const piano = new Piano(audioContext);
await piano.load; // resolves once samples are fetched/decoded

// Play a single note:
piano.start({ note: 'C4', time: scheduledTime, duration: 0.9 });

// Play a chord: call start() once per note at the same `time`
['C4', 'E4', 'G4'].forEach(note => piano.start({ note, time: scheduledTime, duration: 1.2 }));
```

Key implementation points:
- **Loading**: `piano.load` must resolve before the first Start click plays anything — show a brief "loading sound…" state if needed (usually well under a second on a normal connection).
- **Creating the `AudioContext`**: still must happen inside a user-gesture handler (the Start button click) to satisfy browser autoplay policy — same as before.
- **Scheduling**: unchanged from §4 — the look-ahead scheduler still computes *when* each note/chord/rest happens from `audioContext.currentTime`; it now calls `piano.start({ note, time })` instead of building an `OscillatorNode` at that time.
- **Chords**: `smplr` has no built-in multi-note "chord" call — a chord is just several `piano.start()` calls sharing the same `time`, one per note (see §5.1/§5.2 chord rows).
- **Note duration**: pass `duration` a bit shorter than the 1-second slot (e.g. `0.9`s) so notes don't bleed into the next second; chords held at the ends of phrases (§5.1 second 0, §5.2 offsets 9/19) can ring slightly longer (e.g. `1.2–1.5`s) since nothing else plays underneath them.
- **Asset size/dependency tradeoff**: this replaces the "zero external assets" property from the oscillator approach — the app now fetches piano sample files (a few MB, cached by the browser after first load) from smplr's CDN, or these can be self-hosted if offline use is required.

### 5.1 Prep Countdown (seconds 5 → 0)

One event per second:

| Second | Sound |
|---|---|
| 5 | C5 |
| 4 | G4 |
| 3 | E4 |
| 2 | C4 |
| 1 | C major chord: C4 + E4 + G4 + C5 (played together) |
| 0 | Rest — no note. Marks the transition into count-up; the first count-up sound (segment 1, offset 1 → C4) plays at count-up second 1. |

### 5.2 Count-Up (seconds 1, 2, 3, … — after prep ends)

Structured as a **120-second master loop**, made of six **20-second segments**, each keyed to one scale:

| Segment | Count-up seconds (within loop) | Scale |
|---|---|---|
| 1 | 1–20 | C major |
| 2 | 21–40 | D major |
| 3 | 41–60 | E major |
| 4 | 61–80 | E minor |
| 5 | 81–100 | D minor |
| 6 | 101–120 | C minor |

After second 120, the loop repeats from segment 1 (C major).

**Each 20-second segment has two symmetric 10-second halves:**

*First half (seconds 1–10 of the segment) — ascending:*
| Offset (sec) | Event |
|---|---|
| 1–8 | Scale ascending, one note per second (e.g. C major: C4 D4 E4 F4 G4 A4 B4 C5) |
| 9 | First-inversion triad of the scale's tonic chord (upper 3 notes of root-position 4-note chord, e.g. C major: E4 G4 C5) |
| 10 | Rest |

*Second half (seconds 11–20 of the segment) — descending:*
| Offset (sec) | Event |
|---|---|
| 11–18 | Same scale descending, one note per second (e.g. C major: C5 B4 A4 G4 F4 E4 D4 C4) |
| 19 | Root-position triad (lower 3 notes of the 4-note chord, e.g. C major: C4 E4 G4) |
| 20 | Rest |

This pattern (8 scale notes + 1 chord + 1 rest = 10 seconds, twice per segment) applies to every scale in the table above — major scales use the major-scale/major-triad pattern; the three minor segments (61–120) use the natural minor scale and its minor triad in the same up/inversion/rest, down/root-triad/rest shape.

### 5.3 Example note tables for reference

**C major segment (1–20):**
- Up (1–8): C4 D4 E4 F4 G4 A4 B4 C5
- Inversion chord (9): E4 G4 C5
- Rest (10)
- Down (11–18): C5 B4 A4 G4 F4 E4 D4 C4
- Root chord (19): C4 E4 G4
- Rest (20)

**D major segment (21–40):** scale D4 E4 F#4 G4 A4 B4 C#5 D5, tonic triad D-F#-A (inversion: F#4 A4 D5; root: D4 F#4 A4).

**E major segment (41–60):** scale E4 F#4 G#4 A4 B4 C#5 D#5 E5, tonic triad E-G#-B.

**E minor segment (61–80):** natural minor scale E4 F#4 G4 A4 B4 C5 D5 E5, tonic triad E-G-B.

**D minor segment (81–100):** scale D4 E4 F4 G4 A4 Bb4 C5 D5, tonic triad D-F-A.

**C minor segment (101–120):** scale C4 D4 Eb4 F4 G4 Ab4 Bb4 C5, tonic triad C-Eb-G.

*(Implementation should generate these programmatically from a scale-interval table plus a root note, rather than hardcoding each segment, to keep the code maintainable.)*

## 6. Data Model (suggested)

```js
const SCALE_INTERVALS = {
  major: [0, 2, 4, 5, 7, 9, 11, 12],
  minor: [0, 2, 3, 5, 7, 8, 10, 12], // natural minor
};

const SEGMENTS = [
  { root: 'C4', type: 'major' },
  { root: 'D4', type: 'major' },
  { root: 'E4', type: 'major' },
  { root: 'E4', type: 'minor' },
  { root: 'D4', type: 'minor' },
  { root: 'C4', type: 'minor' },
];
// segment index = Math.floor(((countUpSecond - 1) % 120) / 20)
// offset within segment = ((countUpSecond - 1) % 20) + 1
```

Given `countUpSecond`, derive: which of the 6 segments (via `% 120`), which half (first 10 vs second 10), and the offset within that half — then look up the note name(s) to play. Note names are generated as strings like `"C4"` / `"F#4"` from the interval table + root, which is exactly the format `piano.start({ note })` expects — no frequency conversion needed since the sample player handles note-name-to-sample mapping internally.

## 7. Edge Cases & Behaviors to Confirm

- **Resuming mid-chord/mid-rest**: since audio is derived purely from elapsed time, resuming after pause naturally lands back on the correct note/chord/rest — no special-casing needed.
- **Rapid Start/Stop/Pause clicks**: debounce or disable buttons per the state table in §2 to prevent overlapping schedulers.
- **Tab backgrounding**: browsers throttle `setInterval`/`requestAnimationFrame` in background tabs; since the audio clock (`AudioContext.currentTime`) keeps running independently, the display should re-sync correctly when the tab regains focus.
- **Audio autoplay policy**: browsers require a user gesture before audio can play; the Start button click satisfies this (create/resume the `AudioContext` inside the Start click handler).
- **Sample load time**: first Start click should wait on `piano.load` before scheduling notes; consider pre-loading the piano as soon as the page loads (not gated on user gesture — just don't call `piano.start()` until a gesture has occurred) so it's ready by the time Start is clicked.
  - **Restrict loaded notes**: by default `SplendidGrandPiano` loads the full 88-key range across multiple velocity layers, which is a slow download on mobile networks. Since the whole piece only ever touches ~16 distinct pitches (roughly C4–E5), pass `notesToLoad: { notes: [...], velocityRange: [1, 127] }` with exactly the MIDI numbers actually used (computed from the segment/scale tables, not hardcoded) — this cuts the download to a small fraction of the default.
  - **Preload on page load**: fetching/decoding audio doesn't require a user gesture (only playback does), so the piano can start loading as soon as the page opens rather than waiting for the Start click. A shared in-flight promise should be used so a Start click that arrives mid-preload awaits the same load rather than kicking off a redundant one.
- **Mobile screen lock / backgrounding**: iOS Safari (and most mobile browsers) suspend JavaScript execution and audio for any tab that isn't the active, foreground tab — this includes the screen auto-locking from inactivity, the user manually locking it, or switching apps. This is a platform-level restriction with no full workaround from a plain web page. The app requests a **Screen Wake Lock** (`navigator.wakeLock`) while running, which prevents the screen from auto-dimming/locking from inactivity — covering the most common case. It does **not** prevent suspension if the user manually presses the lock button or switches apps; there is no web API that overrides that. A fully background-capable version would require packaging the app as a native app or PWA with background-audio entitlements (e.g. via Capacitor), which is a materially larger project than a static page.

## 8. Tech Stack

Plain HTML/CSS/JS is sufficient — no framework or build step needed:
- **HTML**: display + 3 buttons.
- **CSS**: large monospace digit styling, simple button states.
- **JS**: Web Audio API + `smplr` (piano sample playback) for audio, `requestAnimationFrame` for display updates, a small state machine per §3.
- **Dependency**: `smplr` via CDN (e.g. `<script type="module">import { Piano } from 'https://esm.sh/smplr'</script>`) or npm if using a bundler — no other build tooling required.
