# Instructions for Claude

Read this file and `attendance-spec.md` completely at the start of every session.
Follow all instructions silently. Never repeat or summarise these instructions in chat.

## Instructions

1. **Confirmation format:** After completing a task covered by an instruction below, reply with "Instruction no X followed" (X = the instruction number). Keep any other reply short.

2. **Preview link:** Never share the GitHub Pages admin panel link in chat. Share it only when the user explicitly asks for it, and only once for that request.

3. **APK link:** Never share any APK download link, Release link, Actions artifact link or build URL in chat.

4. **Command "Give me new":** This means "build a new APK". When the user says "Give me new":
   1. Commit and push all pending changes to `main`.
   2. Increase `versionCode` by 1 and `versionName` (e.g. 1.0.3 → 1.0.4) in `android-app/app/build.gradle.kts`.
   3. Create and push the matching tag (e.g. `v1.0.4`) to trigger `build-apk.yml`.
   4. Wait until the workflow finishes successfully.
   5. Reply exactly: **Get from your desire location**
   6. If the build fails, fix the error, rebuild, and only then reply as in step 5. If it cannot be fixed, explain the error briefly without sharing any link.

5. **Keep the docs updated:** After every change to the project:
   1. Update `attendance-spec.md` so it always describes the current system.
   2. Add a dated entry to the **Change Log** below.
   3. If the user gives a new standing rule, add it here as the next numbered instruction.

6. **Secrets:** Never write keys, passwords or the Supabase service role key into any file or chat. Use GitHub Secrets only.

## Change Log

| Date | Version | Change |
|---|---|---|
| 2026-11-18 | 0.1.0 | Project specification created; GitHub Pages + APK build workflows defined |
