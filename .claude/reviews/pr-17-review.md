# PR Review: #17 — Fix sync, Overview numbers, Reminders tab and overlapping audio

**Reviewed**: 2026-09-14
**Branch**: fix/sync-scores-reminders-audio → main
**Decision**: APPROVE with comments (findings below were fixed in 9cfffad)

## Summary
The sync fix is correct and verified against the real Supabase project. Review found two HIGH logic bugs next to the changed code (reminder time zone, alerts that never cleared) plus two small issues; all fixed and covered by tests.

## Findings

### CRITICAL
None. `/api/sync` runs under the caller's JWT, so RLS already limits writes; pinning each row's `patient_id` to the verified patient adds defence in depth.

### HIGH (fixed)
1. `src/lib/engine/adherence.ts` — reminder `time_of_day` (patient wall-clock) was compared with the UTC clock and UTC dates. In IST a 10 am reminder counted as due only from 3:30 pm; "done today" and the ack's scheduled date used UTC too. Now local calendar throughout.
2. `src/app/api/sync/route.ts` `checkMissedSessionsAlert` / `checkLowAdherenceAlert` — missed-days alert fired for a patient added the same day and ignored a game played today; neither alert ever resolved, keeping active patients on "Needs attention". Low adherence fired when nothing had been due (0%). Now self-resolving, new patients exempt, nothing raised with zero due reminders.

### MEDIUM (fixed)
3. `src/app/api/sync/route.ts` — `syncedEventCount` counted rows dropped for invalid ids.

### LOW
4. (fixed) `src/lib/db/sync.ts` — syncQueue entries for schedules that no longer exist were never removed.
5. (not changed, pre-existing) Two phones playing the same game on the same day each upsert their own daily totals; the last sync wins for that game-day.
6. (not changed, pre-existing) `syncQueue` entries for tables that sync via `synced` flags are never drained, so the table grows slowly.

## Validation Results

| Check | Result |
|---|---|
| Type check | Pass |
| Lint | Pass (56 pre-existing errors, 0 warnings, unchanged) |
| Tests | Pass (652, incl. live Supabase integration) |
| Build | Pass |

## Files Reviewed
43 files in 9abe7f6 plus 8 in 9cfffad: sync route/client/wire, patients API, dashboard and patient pages, CognitiveTab, GameBreakdownChart, RemindersTab, SyncStatus, useSync, useLocalScoreRows, audio channel/player/speech/narrate, AudioRouteReset, BigButton, N-Back, adherence engine/adapter, reminders engine/page, schema, starter reminders, DESIGN.md, tests.
