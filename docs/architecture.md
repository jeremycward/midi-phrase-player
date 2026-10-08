# Architecture

## Objective
A programmable MIDI phrase generator/player as a Max for Live MIDI Effect. Pressing a key on a MIDI
keyboard starts playback of an internally stored sequence; it loops while the key is held; release
stops it immediately and silences sounding notes. Trigger pitch and velocity can influence
generation. Sequences live entirely inside the device (no Live clips). No piano-roll editor.
Later, sequences are produced by Scribbletune and a custom JavaScript notation language.

## Layers

```
 MIDI in ─► [Max patcher] ─► v8 (bundle from src/max) ─► [Max patcher] ─► MIDI out
             midiparse        │                               midiformat
             transport        ├─ playback  (pure TS)
             timing           ├─ sequence  (pure TS)
                              ├─ generation (pure TS, Scribbletune)
                              └─ notation  (pure TS)
```

| Layer | Directory | Responsibility | Max-aware? |
|---|---|---|---|
| Notation | `src/notation/` | Parse custom notation text → AST/intermediate | No |
| Generation | `src/generation/` | Produce a `Sequence` from a generator spec + trigger context (Scribbletune adapter, built-in generators) | No |
| Sequence | `src/sequence/` | Beat-relative event data model, validation, helpers | No |
| Playback | `src/playback/` | Pure state machine: trigger events + "advance to beat X" → note-on/off events; tracks sounding notes | No |
| Max integration | `src/max/` | `v8` entry point: handlers for patcher messages, outlets, persistence glue | Yes |
| Patcher | `max/` | MIDI in/out, transport/tempo, clocking, UI, persistence objects | — |

Dependencies point downward only: `max → playback → sequence`, `generation → sequence`, `generation → notation`.

## Data model (beat-relative)

```ts
interface SequenceEvent {
  start: number;     // beats from sequence start (quarter note = 1)
  duration: number;  // beats, > 0
  pitch: number;     // MIDI 0-127
  velocity: number;  // MIDI 1-127
}
interface Sequence {
  length: number;          // loop length in beats
  events: SequenceEvent[]; // sorted by start
}
interface TriggerContext { pitch: number; velocity: number; }
```

Generators are `(ctx: TriggerContext) => Sequence`, so algorithms evolve independently of playback.
Pitch transformations (milestone 4) are functions `Sequence → Sequence` applied with the trigger context.

## Timing and scheduling
- Live's transport is the clock. The patcher derives a tempo-synced tick and sends the script the current beat position.
- The TS playback layer is a pure function of beat position: `advance(beat)` returns the events due. It owns no clock.
- Note-offs are computed by playback logic from durations (not separate JS timers), so stop can flush them deterministically.
- On key release, transport stop, and device deletion, playback emits note-offs for every sounding note (panic flush).
- Start policy: initially start at the next tick (D-006); quantisation is a later option.

## Patcher ↔ script protocol (milestone 1 draft)
Inlet messages to `v8`:
| Message | Meaning |
|---|---|
| `noteon <pitch> <velocity>` | Trigger key pressed |
| `noteoff <pitch>` | Trigger key released |
| `tick <beat>` | Clock tick with current beat position |
| `stop` | Transport stopped / panic |

Outlet messages from `v8`:
| Message | Meaning |
|---|---|
| `note <pitch> <velocity>` | Note-on (velocity 0 = off) → MIDI formatting |
| `status <text>` | Debug/status display |

## Scribbletune
Required dependency. Plan: bundle into the `v8` script with esbuild (no Node built-ins). Spike in milestone 3 checks that it loads in `v8` (watch for `require`, `process`, `Buffer`, `fs`, and its dependencies such as `tonal`). Fallback: `node.script` runs Scribbletune and sends generated sequences (JSON/dict) to `v8`; time-critical scheduling stays in Max. The generator interface isolates this choice.

## Persistence
- Code: `dist/` bundle located with the device (frozen for distribution; see decisions).
- Settings/state: `pattr`/`live.*` parameters and a `dict` with embedded data, saved with the Live set. Generator specs stored as JSON text. Details in milestone 6.

## Testing
- Vitest unit tests for all pure layers (sequence validation, playback state machine incl. note-off guarantees, notation, generators).
- Max-side behaviour is verified manually via checklists in docs/status.md, since Live is the runtime.

## Build
`esbuild` bundles `src/max/main.ts` → `dist/phrase-player.js` (single file, IIFE, target ES2020, no Node built-ins). `tsc --noEmit` for type checking.
