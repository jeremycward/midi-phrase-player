# Decisions

Append-only log. Status: Accepted / Proposed / Superseded.

| ID | Decision | Status |
|---|---|---|
| D-001 | TypeScript for app logic, esbuild bundle to a single JS file for Max `v8`. | Accepted |
| D-002 | Max handles MIDI I/O, transport sync, scheduling; TS holds logic. | Accepted |
| D-003 | Events are beat-relative (quarter note = 1 beat), never ms. | Accepted |
| D-004 | Scribbletune bundled into `v8`; fallback `node.script` for generation only. Do not assume Node APIs in `v8`. | Accepted (spike in milestone 3) |
| D-005 | Playback is a pure, clock-agnostic state machine driven by `tick <beat>` from Max; it owns note-off tracking and flushes on stop. | Proposed |
| D-006 | Playback starts on the next tick after key press in milestone 2; quantisation configurable later. | Proposed |
| D-007 | Use Max `v8` (not legacy `js`) as the script object; tests with Vitest. | Proposed |
| D-008 | `dist/` bundle is committed so the device works from a fresh clone; revisit at milestone 6 (freeze device). | Proposed |
| D-009 | Tick source: patcher sends beat position at a fixed resolution (e.g. 120 ticks/beat) derived from Live's transport. Needs verification in Max. | Proposed |
