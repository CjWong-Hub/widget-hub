# Widget Hub — Design Doc

One Android app ("Widget Hub") that hosts all my custom home-screen widgets.
Goal: **functional** widgets that are cheap to change. Looks don't matter much.

First widget: **Workout widget**, fed by **Coach Steve** (a Claude Code skill).

---

## 1. Architecture

```
Coach Steve (Claude Code skill, runs on laptop in this repo)
      │ writes + commits
      ▼
plans/workout.json  ──push──►  GitHub (public repo)
                                    │ raw URL, fetched by the app
                                    ▼
                    Widget Hub app (Kotlin + Jetpack Glance) on phone
```

Core rule: **the plan lives in data, not code.** Changing the routine = new
`workout.json`, never a rebuild. Every future widget follows the same pattern:
a display template + a data file in a fixed format.

## 2. Repo layout

```
widget-hub/
├── CLAUDE.md                        conventions for Claude Code
├── docs/design.md                   this file
├── plans/workout.json               PUBLIC — fetched by the widget
├── .claude/skills/coach-steve/      Coach Steve skill (public, no personal info)
├── coach-private/                   GITIGNORED — never pushed
│   ├── profile.md                   goals, injury history, equipment
│   ├── state.md                     current phase, start date, latest feedback
│   └── log.md                       phase changes + feedback history
└── android/                         the Android Studio project
```

The repo is public so the app can fetch the plan without a login.
Personal / health notes stay in `coach-private/` (gitignored, laptop only —
back it up somewhere private, e.g. a private repo or cloud folder).

## 3. Plan format — `plans/workout.json`

```json
{
  "version": 1,
  "updated": "2026-12-01",
  "phase": { "number": 1, "name": "Recovery" },
  "weekly_target": 2,
  "rotation": [
    {
      "id": "A",
      "name": "Full body – knee focus",
      "duration_min": 60,
      "exercises": [
        { "name": "Warm-up + scap prep", "sets": 1, "reps": "10 min" },
        { "name": "Spanish squat hold", "sets": 4, "reps": "30-45s", "note": "pain ≤3/10" }
      ]
    }
  ]
}
```

Rules (Coach Steve must obey; the app should validate):
- `version` increases by 1 on every new plan.
- `rotation`: 2–4 routines, ids unique (`A`, `B`, `C`…).
- Each routine: **max 7 exercise lines** (fits a 4×3 widget).
- `name` ≤ 32 chars, `reps` is free text ≤ 12 chars, `note` optional ≤ 40 chars.

## 4. Workout widget behaviour

**Shows:**
- Phase label: `Phase 1 · Recovery`
- Week progress: `This week: 1 / 2`
- Next routine name + its exercise list (`3 × 10/leg  Split squat`)
- Buttons: **✓ Done** (log session), **↶ Undo**, **⟳ Refresh plan**

**Rotation:** next routine = the one after the last logged routine id, looping.
If a new plan no longer contains that id, restart at the first routine.

**Log:** ✓ Done saves `{date, routineId, planVersion}` and advances the rotation.
Undo removes the most recent log entry (only within 24 h).

**Week = Monday–Sunday.** Status from `need = weekly_target − sessions this week`:

| Condition | Status |
|---|---|
| need ≤ 0 | 🟢 green "Week done ✓" |
| Mon–Thu, need > 0 | normal |
| Fri–Sat, need = 1 | 🟡 yellow |
| Fri, need = 2 | 🟡 yellow |
| Sat, need = 2 | 🔴 red |
| Sun, need ≥ 1 | 🔴 red (whole widget background) |

**Refresh:**
- On every tap (Done / Undo / Refresh) → redraw immediately.
- Daily job just after midnight (WorkManager) → recompute week status + fetch plan.
- Fetch = GET `https://raw.githubusercontent.com/<user>/widget-hub/main/plans/workout.json`
  (GitHub may cache ~5 min). On failure keep the last good plan.

## 5. App components (Kotlin)

| Part | Tech |
|---|---|
| Widget UI | Jetpack Glance (`GlanceAppWidget`), actions via `ActionCallback` |
| Storage | DataStore — cached plan JSON + session log list |
| Plan parsing | kotlinx.serialization |
| Networking | simple HTTP GET (HttpURLConnection or OkHttp) |
| Background | WorkManager daily job |
| App screen | minimal: plan URL setting, session history, "fetch now" |
| Logic | pure Kotlin `WeekStatus` + `Rotation` classes, **unit tested** |

Phone notes (Vivo V25e): set app battery usage to **unrestricted**, else the
midnight job can be killed.

## 6. Build order

1. Glance hello-world widget on the phone (proves toolchain).
2. Parse a bundled `workout.json`, show next routine.
3. ✓ Done / Undo + DataStore log + rotation.
4. Week status + colours (+ unit tests).
5. Fetch from GitHub + daily WorkManager job.

## 7. Worth putting on GitHub / learning

- `WeekStatus` / `Rotation` logic with unit tests (date edge cases).
- Data-driven widget architecture (Glance + DataStore + WorkManager).
- LLM skill → structured JSON → device pipeline (Coach Steve).

## 8. Out of scope (for now)

Phase feedback on the widget (done in chat with Coach Steve), per-set logging,
RPE, charts, theming.
