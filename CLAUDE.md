# Widget Hub — notes for Claude Code

- Read `docs/design.md` before any app work. Follow its architecture and build order.
- The Android project lives in `android/` (Kotlin, Jetpack Glance, min SDK 26).
- Every widget = a Glance template + a JSON data file in a fixed format. Plans/data
  live in `plans/`, never hard-coded.
- Keep logic (week status, rotation) in plain Kotlin classes with unit tests.
- The user is learning Kotlin/Android: after each step, briefly explain what each
  new file does and why.
- Workout planning is done by the `coach-steve` skill. Only Coach Steve edits
  `plans/workout.json`.
- `coach-private/` holds personal health notes: read it when needed, never commit
  it, never copy its contents into public files.
- Dev machine: Windows laptop, 8 GB RAM — test on the real phone over USB, no emulator.
