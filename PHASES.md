# OnTime – Development Phases

Step-by-step plan to finish the Staff Attendance System described in `attendance-spec.md`.
Tick each box (`- [x]`) as it is completed. A phase is finished only when every item under **Done when** is true.

| Phase | Name | Depends on | Status |
|---|---|---|---|
| 0 | Setup & accounts | — | ⬜ Not started |
| 1 | Supabase backend | 0 | ⬜ Not started |
| 2 | Android – login & device binding | 1 | ⬜ Not started |
| 3 | Android – attendance flow & offline sync | 2 | ⬜ Not started |
| 4 | Admin web – core | 1 | ⬜ Not started |
| 5 | Reports & exports | 3, 4 | ⬜ Not started |
| 6 | Data removal & backup | 1, 5 | ⬜ Not started |
| 7 | Testing, pilot & v1.0 release | all | ⬜ Not started |
| 8 | Optional improvements | 7 | ⬜ Later |

Phases 2–3 (Android) and 4 (Admin web) can run in parallel once Phase 1 is done.

---

## Phase 0 – Setup & accounts

- [ ] Create the Supabase project (free plan), region closest to Pakistan (e.g. Mumbai / Singapore).
- [ ] Create the folder structure: `android-app/`, `admin-web/`, `supabase/`, `.github/workflows/`.
- [ ] Add `.gitignore` (Android, Node, `*.jks`, `.env*`, `local.properties`).
- [ ] Install Supabase CLI locally; `supabase init` inside `supabase/`; link to the project.
- [ ] Generate the release keystore with `keytool` and store the original file safely offline (two copies).
- [ ] Generate the **QR signing key pair** (Ed25519). Private key → Supabase Edge Function secret only. Public key → bundled in the Android app (it is not secret).
- [ ] Add GitHub Secrets: `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`, `SUPABASE_URL`, `SUPABASE_ANON_KEY` (+ later `SUPABASE_DB_URL`, `GDRIVE_*` for backup).
- [ ] Settings → Pages → Source = "GitHub Actions".
- [ ] Add workflow files `deploy-admin.yml` and `build-apk.yml` (from the spec).
- [ ] In `admin-web/vite.config.js` set `base: '/OnTime/'` (the real repo name, not `attendance-system`).

**Done when:** an empty Vite page deploys to GitHub Pages and an empty Android app builds a signed APK in Actions.

---

## Phase 1 – Supabase backend

### 1.1 Database tables (`supabase/migrations/`)
- [ ] `staff` – id (T001), auth_user_id, full name, role (admin/staff), group, active flag, created_at.
- [ ] `devices` – staff_id, public_key, model, attestation result, rooted flag, status (active/revoked/pending), registered_at, revoked_at.
- [ ] `qr_codes` – version, token, signature, active flag, generated_by, generated_at.
- [ ] `settings` – academy lat/long, radius (default 100 m), school start/end time, late/early rules.
- [ ] `attendance` – transaction_id (unique), staff_id, device_id, event_type, device_time, server_time, estimated_true_time, uptime, lat/long/accuracy, distance, range_status, mock_location, qr_version, photo paths + hashes, signature, flags[], sync_status, corrected_by, correction_reason.
- [ ] `monthly_summary` – staff_id, month, days present, lates, early departures, absences, total hours (kept permanently).
- [ ] `audit_log` – actor, action, table, record id, old/new values, timestamp.
- [ ] Unique constraint: one check-in and one check-out per staff per day.

### 1.2 Security
- [ ] Row Level Security ON for **every** table.
- [ ] Staff: read only their own attendance/devices; cannot insert directly (insert goes through Edge Function).
- [ ] Admin: read all; changes only through Edge Functions.
- [ ] Storage bucket `attendance-photos` (private), per-staff folders, signed URLs for viewing.
- [ ] Audit triggers on `attendance` (corrections), `devices`, `qr_codes`, `settings`, `staff`.

### 1.3 Edge Functions (`supabase/functions/`)
- [ ] `admin-create-staff` – creates auth user `t001@alfurqan.local` + staff row.
- [ ] `admin-reset-password`, `admin-set-active`.
- [ ] `register-device` – stores public key, model, attestation chain check, root flag.
- [ ] `admin-device` – revoke / approve replacement.
- [ ] `submit-attendance` – verifies device signature, ignores repeated transaction IDs, enforces 1 check-in/1 check-out per day, computes distance & range status server-side, sets flags (Out of Range, Fake GPS, Old QR, Duplicate Photo, Late Sync), records server time.
- [ ] `regenerate-qr` – new token, version +1, signs payload, deactivates old version.
- [ ] `server-time` – returns server time for the app's time-evidence sync.
- [ ] `admin-correct-attendance` – manual correction with required reason.
- [ ] Every admin function checks the caller is an active admin.

**Done when:** all tables, RLS and functions are deployed, and a test script can create a staff member, register a device, submit a signed record and see it rejected when the signature is wrong.

---

## Phase 2 – Android: login & device binding

- [ ] Project setup: Kotlin, minSdk 24+, Jetpack Compose (or XML), Hilt/manual DI, Supabase Kotlin SDK.
- [ ] `SUPABASE_URL` / `SUPABASE_ANON_KEY` injected from env into `BuildConfig` (never hard-coded).
- [ ] Signing config from the spec in `app/build.gradle.kts`; `versionCode` / `versionName` start at 1 / 1.0.0.
- [ ] Login screen: staff types "T001" + password → app converts to `t001@alfurqan.local`.
- [ ] First login must be online; session cached securely afterwards (EncryptedSharedPreferences).
- [ ] BiometricPrompt (fingerprint or device PIN) before each attendance.
- [ ] Android Keystore key pair (EC P-256) generated on first login, with Key Attestation.
- [ ] Root detection (su binaries, test-keys, Magisk paths, etc.).
- [ ] Call `register-device`; show "Device pending approval" if the admin must approve a replacement.
- [ ] Home screen: Check-in, Check-out, My Attendance, Sync Now, pending-records counter.

**Done when:** a staff member logs in once online, then can open the app offline and pass fingerprint/PIN unlock; the device appears in the database.

---

## Phase 3: Android attendance flow & offline sync

### 3.1 Capture flow (Scan → Photo → GPS)
- [ ] CameraX + ML Kit QR scanner; verify QR signature offline with the bundled public key; reject fake QR.
- [ ] Remember the latest known QR version; mark "Old QR" if an older version is scanned.
- [ ] Front camera auto-capture (no gallery, no manual shutter).
- [ ] Save original + overlay copy (school, name, ID, date, "Device time", location status, device status, reference AT-xxxxx).
- [ ] Compress each image to about 100–150 KB; SHA-256 hash of each.
- [ ] Fused location: lat, long, accuracy, location time, distance to academy, mock-location detection.
- [ ] Range label: "In Range – 42 m" / "Out of Range – 350 m" / "Fake GPS suspected". Never reject.

### 3.2 Local storage & duplicate rules
- [ ] Encrypted Room database (SQLCipher), with all fields from spec §6.
- [ ] Transaction ID (UUID) generated on the phone.
- [ ] Record payload signed with the Keystore private key.
- [ ] Repeat scan within 10 minutes → "Already marked", not stored.
- [ ] Only one check-in and one check-out per day locally.

### 3.3 Sync & time evidence
- [ ] WorkManager job: runs when internet returns, retries with backoff, uploads photos then record, resumes interrupted uploads.
- [ ] "Sync Now" button.
- [ ] At each online sync save (server time, `SystemClock.elapsedRealtime()`); compute estimated true time for offline events. Handle phone reboot (uptime resets) by flagging the estimate as unreliable.
- [ ] Records older than 72 hours unsynced → "Late Sync".
- [ ] My Attendance screen: own records with time, distance, range status and sync status.
- [ ] Optional: check latest GitHub Release and prompt to update.

**Done when:** a full check-in and check-out works in airplane mode, syncs automatically when online, and appears on the server with correct flags.

---

## Phase 4 – Admin web: core

- [ ] Vite + React scaffold, `HashRouter`, Supabase JS client (anon key only).
- [ ] Admin login; non-admin accounts are refused.
- [ ] **Dashboard:** total staff, present, absent, late, currently on campus, pending offline records, flagged records by type.
- [ ] **Staff:** add, edit, deactivate, reset password.
- [ ] **Devices:** list, history, revoke, approve replacement.
- [ ] **Attendance:** daily and monthly views, filters, photo viewer (original | overlay side by side), map pin (Leaflet + OpenStreetMap, free), manual correction with reason.
- [ ] **QR management:** Regenerate button, A4 PDF download (large QR, school name, "Staff Attendance – Scan Here", version, date), version history.
- [ ] **Settings:** academy location (pick on map) and radius; school start/end times.
- [ ] **Audit log** viewer.
- [ ] Mobile-friendly layout (admin may use a phone).

**Done when:** the admin can do every action listed in spec §9 from the GitHub Pages site without any service-role key in the code.

---

## Phase 5 – Reports & exports

- [ ] Status engine (SQL views or functions): late arrival, early departure, working hours, missing check-in, missing check-out, absence.
- [ ] Individual staff report.
- [ ] Daily attendance sheet.
- [ ] Monthly summary.
- [ ] Late-arrival report.
- [ ] Working-hours report.
- [ ] GPS distance and status shown in every report.
- [ ] Export every report to Excel (SheetJS) and PDF (jsPDF + autotable).

**Done when:** each report matches hand-checked numbers for a test week and exports cleanly to both formats.

---

## Phase 6 – Data removal & backup

- [ ] Nightly scheduled job (pg_cron or scheduled Edge Function):
  - [ ] Build/refresh `monthly_summary` first.
  - [ ] Delete photos older than 30 days from Storage.
  - [ ] Delete detailed attendance rows older than 30 days.
- [ ] Before monthly deletion: auto-generate the month's report as Excel + PDF and store it (Storage or Google Drive).
- [ ] `weekly-backup.yml`: `pg_dump` via `SUPABASE_DB_URL` secret → upload to the admin's Google Drive (rclone or Drive API with a service account stored in secrets).
- [ ] "Download full backup" button in the admin web.
- [ ] Reminder note on the dashboard for vacations (open once a week so the free project doesn't pause).

**Done when:** a test record dated 31+ days ago is removed overnight, its summary survives, and a backup file appears in Google Drive.

---

## Phase 7 – Testing, pilot & v1.0 release

- [ ] Security tests: wrong device signature, fake QR, old QR, mock GPS app, rooted phone, changed phone clock, replayed transaction ID, duplicate photo.
- [ ] RLS tests: staff cannot read others' data or write directly.
- [ ] Offline tests: 3 days offline, reboot while offline, interrupted upload.
- [ ] Test on 3–5 real staff phones (different brands / Android versions).
- [ ] Storage check against the free 1 GB limit.
- [ ] Pilot with a small staff group for 1–2 weeks; fix issues.
- [ ] Staff guide (Urdu/English): install APK, first login, daily use.
- [ ] Admin guide.
- [ ] Tag `v1.0.0` → APK published on Releases.

**Done when:** pilot runs a full week with no lost records and v1.0.0 is released.

---

## Phase 8 – Optional improvements (after v1.0)

- [ ] Grace period before "Late" (e.g. 10 minutes).
- [ ] Friday timings / separate staff groups.
- [ ] Holiday calendar (Sundays & holidays not counted as absent).
- [ ] Leave requests from the app, approved by admin.
- [ ] Closing-time alert to admin listing missing check-outs.
- [ ] Play Integrity via Play Console internal testing ($25 one-time).

---

## Points to decide / fix in the spec

1. **Spec file name:** CLAUDE.md refers to `attendance-spec.md`; the spec has been renamed to match.
2. **Vite base path:** must be `/OnTime/`, not `/attendance-system/`.
3. **Example report times:** departures like "01:45" should be written "13:45" (or "01:45 PM") so working-hours maths is unambiguous.
4. **Storage estimate:** 30 staff × 2 events × 2 photos × 150 KB ≈ 18 MB/day; 30 calendar days ≈ 540 MB, about 400 MB if only ~22 working days. Still within 1 GB.
5. **Key attestation on the server:** fully verifying the attestation certificate chain inside a Deno Edge Function is complex; v1 can store the chain and do a basic check, with full verification later.
6. **Uptime counter** resets when the phone restarts, so offline time estimates after a reboot must be flagged.
