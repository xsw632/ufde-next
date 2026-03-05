---
status: awaiting_human_verify
trigger: "Top-right close button cannot close window after installing a new package, and sending open project causes severe freeze."
created: 2026-03-05T19:14:19+08:00
updated: 2026-03-05T19:22:18+08:00
---

## Current Focus

hypothesis: Fix is implemented and build checks pass; final validation requires runtime confirmation in packaged workflow.
test: User verifies top-right close and open-project behavior in actual desktop app package.
expecting: Close button exits immediately; open-project no longer causes lockup/freeze.
next_action: wait for human verification feedback from reproduced workflow.

## Symptoms

expected: Clicking top-right close should close app window immediately; open-project command should open project without system freeze.
actual: Close button does nothing. Triggering open project causes severe freeze (perceived whole-system lockup).
errors: Not provided yet.
reproduction: 1) Install latest package build. 2) Launch app. 3) Click top-right close: window does not close. 4) Send/open project action: machine freezes.
started: Still reproducible after installing newer package; likely persistent across recent builds.

## Eliminated

<!-- Append disproved hypotheses here -->

## Evidence

- timestamp: 2026-03-05T19:14:58+08:00
  checked: repository-wide symbol search for close and open-project flows
  found: open-project logic is centralized in `src/model/project.ts` and called from `src/HeaderBar.tsx` and `src/pages/StartupPage.tsx`; close-window interception exists in `src/App.tsx` (`onCloseRequested`).
  implication: investigation should focus on `App.tsx` close-request behavior and project opening implementation path.
- timestamp: 2026-03-05T19:15:40+08:00
  checked: full implementations in `src/App.tsx`, `src/model/project.ts`, `src/HeaderBar.tsx`, and `src/pages/StartupPage.tsx`
  found: `openProject` relies on dialog + fs plugins (`open` and `readTextFile`), while close behavior depends on Tauri window events with a close-request interception guard.
  implication: if packaged permission scopes are wrong, both open-project and close behavior can fail despite correct-looking frontend logic.
- timestamp: 2026-03-05T19:16:55+08:00
  checked: git history and latest commit diff for `src/App.tsx`
  found: commit `d45377a` removed explicit close path in `onCloseRequested` when project is not modified; only modified-project path now calls `appWindow.close()`.
  implication: if Tauri close-request listener requires explicit close behavior with a registered handler, top-right close becomes a no-op in common cases.
- timestamp: 2026-03-05T19:17:35+08:00
  checked: recent history and `ProjectPage` rendering path
  found: `ProjectPage` itself has no obvious heavy loops after opening a project; latest regression is concentrated in close-request handling changed today.
  implication: close-event recursion/no-op is now the strongest unifying cause candidate for both symptoms.
- timestamp: 2026-03-05T19:18:31+08:00
  checked: `@tauri-apps/api` source (`window.js`, `webviewWindow.js`) and changelog
  found: `onCloseRequested` auto-calls `destroy()` if not prevented; `close()` emits another close-requested event; `destroy()` is the force-close path.
  implication: guarded code paths that call `close()` from within close-request handlers can recurse, so fix should use `destroy()` after explicit guard bypass.
- timestamp: 2026-03-05T19:20:32+08:00
  checked: capability file + current close guard implementation
  found: `src-tauri/capabilities/migrated.json` grants `core:window:allow-close` but not `core:window:allow-destroy`; `src/App.tsx` guarded path calls `appWindow.close()`.
  implication: close can no-op when auto-destroy permission is missing; guarded close path is vulnerable to close-request recursion patterns.
- timestamp: 2026-03-05T19:21:01+08:00
  checked: applied fix in source files
  found: `src/App.tsx` guarded close now calls `appWindow.destroy()` (no `useRef` bypass path), and capability list now includes `core:window:allow-destroy`.
  implication: close path aligns with Tauri v2 semantics and avoids re-emitting close-request during guarded quit flow.
- timestamp: 2026-03-05T19:22:18+08:00
  checked: `pnpm build`
  found: build succeeded (`tsc && vite build`), no TypeScript errors.
  implication: frontend changes compile and bundle correctly.
- timestamp: 2026-03-05T19:22:18+08:00
  checked: `cargo check` in `src-tauri`
  found: check succeeded; only pre-existing dead-code warnings in `src/vlfd/*`.
  implication: backend/capability changes are accepted by Rust/Tauri build pipeline.

## Resolution

root_cause:
root_cause: Tauri v2 close flow now depends on `destroy` inside `onCloseRequested`, but capability permissions omitted `core:window:allow-destroy`; the guarded modified-project path also used `close()` which can re-enter close-request logic and trigger loop-like freezes.
fix: Added `core:window:allow-destroy` permission and replaced guarded close calls from `appWindow.close()` to `appWindow.destroy()` in `src/App.tsx`.
verification:
verification: `pnpm build` passed; `cargo check` passed (warnings only). Runtime verification in packaged app pending human confirmation.
files_changed: ["src/App.tsx", "src-tauri/capabilities/migrated.json", "src-tauri/gen/schemas/capabilities.json"]
