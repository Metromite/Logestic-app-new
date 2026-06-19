# Logistics AI Planner — Native Architecture

Migration target: Streamlit/Python → React + TypeScript + Vite + Tauri v2 + SQLite + Firebase.
Source analyzed: `LogisticsAIPlanner/app.py` (3,797 lines), `main.py`, `streamlit_launcher.py`.

## 1. What was ported, and where it lives now

| Original (app.py) | Ported to |
|---|---|
| `unify_text`, `parse_date_safe` | `src/core/dateUtils.ts` |
| `build_vacation_cache`, `build_experience_cache`, `is_on_vacation`, `vacation_within_3_months`, `get_vac_status`, `get_total_exp`, `validate_experience` | `src/core/rotationEngine.ts` |
| `calculate_candidate_score`, `get_best_candidate`, scoring constants | `src/core/scoringEngine.ts` |
| `check_route_requirements` | `src/core/fleetValidation.ts` |
| `init_sqlite_db()` (CREATE TABLE + ALTER chain) | `src-tauri/migrations/0001_init.sql` (flattened — no incremental ALTERs needed for a fresh schema) |
| `run_query` / `load_table` (+ `_sync_log` outbox) | `src-tauri/src/db.rs` (`run_mutation`, `load_table`) + `src/store/dbBridge.ts` |
| `process_sync_log` / `sync_down_from_cloud` | `src-tauri/src/sync_worker.rs` (push) + `src/firebase/pullSync.ts` (pull) |
| Streamlit tabs (`st.data_editor` per entity) | `src/components/entities/*View.tsx` |
| "Generate Route Plan" tab incl. anchor/vehicle/health-card logic | `src/components/routing/RoutePlannerView.tsx` |

No business rule was altered: scoring weights (`ANCHOR_MATCH_BONUS = 50000`, `NEVER_WORKED_BONUS = 10000`,
`RECENT_AREA_PENALTY = -3000`, etc.), the 14-day minimum assignment rule, the Consumer-sector
health-card quota (`hcAssigned < 3`), and the anchor matching logic (substring match against
area name / sector / vehicle / area code) are copied verbatim into TypeScript.

## 2. Layered structure

```
src-tauri/                  Rust shell (Tauri v2)
  src/db.rs                 SQLite command layer (load_table, run_mutation)
  src/sync_worker.rs         Outbox → Firestore batched push
  migrations/0001_init.sql  Full schema (drivers, helpers, areas, vehicles, history,
                             vacations, active_routes, draft_routes, route_plan_reasons,
                             vacation_predictions, default_*, _sync_log)

src/
  core/                     Pure TS business logic (no I/O) — unit-testable 1:1 port
  store/dbBridge.ts         Thin invoke() wrapper matching the Rust commands
  firebase/                 client.ts (auth/init), pullSync.ts (lazy delta pulls)
  components/
    layout/                 Sidebar + TopBar (Liquid Glass shell)
    routing/RoutePlannerView.tsx   Route generation UI bound to core/scoringEngine
    entities/*View.tsx       Drivers/Helpers/Areas/Vehicles/Vacations grids
    assistant/                Floating mascot AI widget + pluggable backend
    ui/liquidGlass.css        Design tokens (glass blur, accent, shadows)
```

## 3. Why this preserves correctness

The Python app's `run_query()` always does two things atomically: (1) write to the
operational SQLite table, (2) append a row to `_sync_log` if the table is in
`SYNC_TABLES`. `db.rs::run_mutation` reproduces this exactly inside a single
`rusqlite` transaction, so a crash between the two writes is impossible — same
guarantee as the original, now enforced at the Rust layer instead of Python's
try/except + manual commit.

## 4. AI Assistant

The City Pharmaceutical mascot (`src/assets/mascot.jpeg`) is rendered as a floating
FAB (`AssistantWidget.tsx`) with a Liquid Glass chat panel. The brain is pluggable via
`AssistantBackend` (`AssistantProvider.tsx`) — drop in an Anthropic, OpenAI, or local
Whisper/STT key with zero UI changes. The assistant is grounded with live SQLite
context (driver/helper/area counts, vacation cache, experience cache) on every turn.

## 5. Next implementation steps (not yet wired, by design — structure only)

- `upsert_synced_rows` Tauri command (referenced in `pullSync.ts`) — bulk UPSERT from
  Firestore docs into SQLite, keyed by `fb_id`.
- Auth gate (Firebase email/password or anonymous) before first sync.
- Excel export (`generate_excel_with_sn` in app.py) → port to a Rust `xlsx` writer
  (e.g. `rust_xlsxwriter`) exposed as a Tauri command, or use SheetJS client-side.
- Vacation prediction heuristics (lines ~3300+ of app.py) — port into
  `core/vacationPredictor.ts` following the same pattern as `scoringEngine.ts`.
