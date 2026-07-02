# Count-Up Timer with Piano Sound

A single-page timer: a 5-second countdown ("prep"), then an indefinite count-up, each second paired with a real piano note — scales and chords that cycle through major and minor keys. Auto-stops after 10 minutes of running time if you forget to hit Stop.

See [`docs/design-doc.md`](docs/design-doc.md) for the full spec: UI behavior, the timer state machine, the exact note/chord/rest sequence for every second, and the audio scheduling approach.

## Running it

No build step or install required — it's a single static HTML file.

```
open index.html
```

or serve it locally, e.g.:

```
python3 -m http.server 8000
```

then visit `http://localhost:8000`.

Piano samples are loaded on first Start click (via [`smplr`](https://github.com/danigb/smplr), CDN-hosted, no local assets needed) — an internet connection is required the first time.

## Known limitation: mobile screen lock

On iOS Safari (and most mobile browsers), the timer stops if the screen locks or you switch apps — this is a browser platform restriction, not something a web page can fully override. The app requests a Screen Wake Lock to keep the screen from auto-dimming while running, which covers the most common case, but manually locking the phone or switching apps will still suspend it. See the design doc's Edge Cases section for details.

## How it works

- **Start** — begins the prep countdown (5→0), then counts up. If paused, resumes exactly where it left off.
- **Pause** — freezes the display and audio in place.
- **Stop** — resets to idle. Also fires automatically at the 10-minute mark.

Implementation notes (why it doesn't drift, how notes are scheduled, etc.) are in the design doc.
