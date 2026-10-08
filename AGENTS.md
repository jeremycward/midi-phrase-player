# AGENTS.md — Persistent development instructions

Project: **midi-phrase-player**, a Max for Live MIDI Effect (Ableton Live, Max 9.0.8) that plays
internally stored, programmatically generated MIDI phrases while a keyboard note is held.

## Environment
- Windows 11, PowerShell, VS Code. Use Windows paths in commands.
- TypeScript source, bundled with esbuild into `dist/` for Max's `v8` object.
- npm for dependencies, Git for version control.
- Max 9.0.8 runtime. **Do not assume Node.js APIs exist in `v8`** (no `require` of core modules, `fs`, `process`, `Buffer`, timers other than Max's `Task`/`setTimeout` equivalents, etc.).

## Architecture rules (see docs/architecture.md)
1. Most application logic lives in TypeScript; the patcher stays small.
2. Max objects handle MIDI I/O, transport sync and event scheduling.
3. Keep separate: notation parsing, sequence generation, sequence model, playback.
4. Events are beat-relative (`SequenceEvent`); never store wall-clock ms in the model.
5. Timing derives from Live's transport/tempo (`transport`, `metro`/`timepoint` in tempo-relative ticks), not an independent clock.
6. Every started note must be guaranteed an off. Stop = flush all sounding notes.
7. Pure modules (`sequence`, `notation`, `generation`, `playback` logic) must not import from `src/max/` or touch Max globals. Only `src/max/` may use Max APIs (`Max`, `Task`, `outlet`, `post`, ...).
8. State and external dependencies must survive save/reopen of the device.

## Workflow
- Work in small milestones (see docs/status.md). Each milestone must leave a working device.
- Before finishing a task: `npm run typecheck`, `npm test`, `npm run build` must pass.
- Do not commit without asking the user first. Commit messages: imperative summary; include the Co-authored-by trailer.
- Update `docs/status.md` after each milestone, and `docs/decisions.md` for any non-trivial design choice (append; don't rewrite history).
- Prefer small surgical changes; don't implement beyond the agreed milestone.

## Max patcher rules
- `.maxpat` files are JSON; generate valid ones that open in Max 9 / Live. Keep them hand-readable: few objects, explicit comments, presentation mode set up for the device UI.
- MIDI effect devices need `"classnamespace": "box"`, `midiin`/`midiout` (or `midiparse`/`midiformat`) and device dimensions set (`"devicewidth"`).
- Reference the bundle via a path relative to the device/project (`dist/…`); the file must be in the search path / frozen with the device.
- Document every message the `v8` object receives/sends in docs/architecture.md (the "patcher ↔ script protocol").
- Any change to a patcher must be verified by the user opening it in Max; say what to look for.

## Code conventions
- TypeScript strict mode, ES modules in source; esbuild output is a single IIFE/CJS-free script for `v8`.
- Tests with Vitest in `tests/`, covering pure modules.
- No comments unless they clarify non-obvious logic.

## Layout
`max/` patchers · `src/generation/` · `src/notation/` · `src/sequence/` · `src/playback/` · `src/max/` · `tests/` · `examples/` · `docs/` · `dist/` (generated; committed only if needed for the device to run — see decisions.md)
