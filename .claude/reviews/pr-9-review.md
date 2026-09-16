# PR Review: #9 — Add appointment details and day-before/day-of prompts to appointment reminders

**Reviewed**: 2026-09-17
**Author**: prathameshfuke
**Branch**: feat/appointment-reminders → main (smriti-backup, already merged as a464350)
**Decision**: REQUEST CHANGES (follow-up PR)

## Summary
Occurrence logic, adherence counting and sanitizing are careful and well tested. One sync bug when a reminder changes type, plus unrelated binary assets in the PR.

## Findings

### CRITICAL
None

### HIGH
1. **Changing an appointment to another type fails to sync** — `src/lib/db/wire.ts:128` and `src/lib/db/serverProfile.ts:77` only send appointment columns for appointment rows. The edit form lets the type change (`src/app/reminders/page.tsx:288`), so the upsert leaves the old `appointment_date` etc. on the server row, and MIGRATION 013's `reminder_schedules_appointment_fields_check` rejects it. Fix: when the type is not `appointment`, send the six columns as explicit `null` (only once the migration is applied everywhere), or block type changes on existing appointment rows.
2. **MIGRATION 013 must be applied before this build ships** — appointment rows send the new columns; without the migration every appointment upsert fails. Confirm it ran on the Supabase project.

### MEDIUM
3. **24 unrelated PNGs (3.6 MB) in `smriti/appassests/`** were added in this PR. Not part of the feature, and the folder name is misspelled. Move them to their own commit/PR or `public/`.
4. **Past appointments stay in "active" lists forever** — `reminders/page.tsx` `activeSchedules` does not filter out dated appointments whose date has passed. Hide or auto-deactivate them.

### LOW
5. Server-side `computeAdherence` uses the server clock for `nowStamp` (`lib/engine/adherence.ts:68-92`); on a UTC server an Indian patient's day-before window counts as closed/open up to 5.5 h off. Same as existing weekday logic, so only worth fixing together.
6. Extra blank line added in `src/lib/db/sync.ts:236`.

## Validation Results

| Check | Result |
|---|---|
| Type check | Pass |
| Lint | Fail — pre-existing (132 errors before and after) |
| Tests | Same 11 pre-existing failures; new appointment tests pass |
| Build | Pass |

## Files Reviewed
Source: engine/appointments.ts (added), engine/reminders.ts, engine/adherence.ts, db/wire.ts, db/sync.ts, db/serverProfile.ts, db/schema.ts, hooks/useReminders.ts, app/reminders/page.tsx, app/app/page.tsx, api/sync, api/patients/[id]/adherence, api/ai/generate-digest, dashboard/adherenceAdapter.ts
Other: i18n locales, docs/03_DATABASE.md, tests (added), 24 PNGs (added)
