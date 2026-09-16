# PR Review: #10 — Encrypt personal and health data stored on the device

**Reviewed**: 2026-09-17
**Author**: prathameshfuke
**Branch**: feat/secure-storage-impl → main (smriti-backup, already merged as fd300df)
**Decision**: REQUEST CHANGES (follow-up PR)

## Summary
Solid design (sync AES-GCM in a DBCore middleware, key sealed by a non-extractable CryptoKey, AAD per field). Main risk is silent local data loss when the key is missing or replaced.

## Findings

### CRITICAL
None

### HIGH
1. **Two tabs on first open can create two different keys** — `src/lib/db/crypto/keys.ts:44-62`. Both tabs run `ready`, both see no keyring row, both `put` a new key; the last write wins. Rows the other tab wrote are then unreadable, and the next migration pass clears those whole tables (including `syncQueue`). Fix: create the key inside a `rw` transaction on `keyring` that re-reads before writing (or use `add` and reload on ConstraintError), and have other tabs reload the key on `versionchange`/storage events.
2. **Any unseal failure replaces the key and wipes tables** — `keys.ts:47-49` + `migrateToEncrypted` (`keys.ts:120-127`). A fresh keyring entry has no `migratedVersion`, so migration runs, reading an old row throws `StorageDecryptError`, and the table is cleared. This drops unsynced `syncQueue` rows (game progress, reminder acks) without telling the user. Fix: do not overwrite an existing keyring row automatically; surface an error/telemetry event, and at least flush sync before clearing, or skip clearing `syncQueue`.

### MEDIUM
3. **`rawKey` fallback stores the data key in plain text** next to the ciphertext — `keys.ts:59,67`. In that mode encryption only hides data from casual devtools browsing. Log it once and document that it gives no at-rest protection.
4. **Logout clears chats/messages/quizzes but not `memoryBankEntries`, `patientPhotos`, `telemetryEvents`** — `src/app/caregiver/settings/page.tsx:205-215`. They are encrypted but still readable by the next caregiver who signs in on the device if queries ever stop filtering by patient. Decide explicitly and add a test.

### LOW
5. AAD is `table.field` only, not the row id, so a ciphertext can be swapped between rows of the same table undetected — `cipher.ts:48`. Include the primary key in the AAD where available.

## Validation Results

| Check | Result |
|---|---|
| Type check | Pass (after `next build` generates types) |
| Lint | Fail — 132 errors, same count as before PR #9/#10 (pre-existing) |
| Tests | 748 pass / 11 fail — same 11 fail before these PRs (Supabase integration + login page) |
| Build | Pass |

## Files Reviewed
Added: crypto/cipher.ts, crypto/fields.ts, crypto/keys.ts, crypto/middleware.ts, tests/data-layer/secure-storage.test.ts
Modified: lib/db/schema.ts, app/caregiver/settings/page.tsx, package.json, package-lock.json, tests/data-layer/db.test.ts
