# Windows active-window capture from the Screenshot Window shortcut

**Status:** implementation plan only. No feature code is included in this pull request.

Tracks [upstream issue #369](https://github.com/jamesmontemagno/tiny-clips/issues/369). The
dedicated `ScreenshotWindow` hotkey currently opens the window selector, which prevents a
hands-free workflow such as invoking it with AutoHotkey bound to Alt+PrintScreen and immediately
capturing the foreground window.

## Scope and acceptance criteria

- Add an opt-in Screenshot setting, default off, labeled **Capture active window directly from
  the Screenshot Window shortcut**.
- When enabled, the dedicated window-screenshot hotkey captures the eligible foreground window
  directly without showing `WindowPickerWindow`.
- Resolve the foreground HWND before activating any TinyClips UI, then validate it using the same
  eligibility rules as the existing window picker.
- When no eligible foreground window exists, cancel without creating a file or changing the
  clipboard. Emit a capture trace and a concise user-visible notification so the no-op can be
  diagnosed.
- When disabled, preserve the current window-selector flow exactly.
- Do not change the default hotkey, the general capture picker's Window option, or video/GIF
  behavior.

The feature is Windows-only: the request is specifically for the Windows `ScreenshotWindow`
shortcut and Alt+PrintScreen automation scenario. No macOS changes are proposed.

## Proposed design

`CaptureScreenshotWindowAsync` routes the dedicated hotkey into
`BeginCaptureAsync(CaptureType.Screenshot, CapturePickerMode.Window)`. On that route only, check
the new preference and, when enabled, capture the foreground HWND before any TinyClips overlay or
picker is activated. Validate the handle against the same candidate eligibility used by
`WindowEnumerator`—visible, non-minimized, non-cloaked, not TinyClips, sufficiently large, and
selectable—then pass it to the existing `CaptureTarget.Window` screenshot and save flow.

Keep the setting off by default. If the foreground window is missing or ineligible, stop the flow
without falling back to the selector and without saving/copying anything; report the no-op through
the normal capture trace plus a short notification. A genuine capture failure should continue
through the existing screenshot error path. Do not bind Alt+PrintScreen by default; users may bind
the dedicated shortcut or invoke it through automation.

## Likely implementation surfaces

| Area | Files/modules | Responsibility |
|---|---|---|
| Shortcut routing | `windows/src/TinyClips.App/App.xaml.cs` | Gate only the dedicated `ScreenshotWindow` route, resolve the foreground HWND before UI activation, and reuse the screenshot pipeline for a valid target. |
| Window eligibility | `windows/src/TinyClips.App/Infrastructure/WindowEnumerator.cs` and a small testable resolver in `windows/src/TinyClips.Core/Capture` | Reuse existing selectable-window rules; isolate handle-to-candidate resolution so eligibility can be unit tested without Win32 calls. |
| Preference | `windows/src/TinyClips.Core/Services/ICaptureSettings.cs`, `CaptureSettings.cs` | Persist the opt-in preference with a false default and reset behavior consistent with other capture settings. |
| Settings UI | `windows/src/TinyClips.App/ViewModels/SettingsViewModel.cs`, `Controls/Settings/ScreenshotSettingsSection.xaml` | Load/persist the toggle and give it an accessible name and concise explanation. |
| Tests | `windows/tests/TinyClips.Core.Tests` and relevant app-level seams | Cover default/disabled routing, enabled direct routing, and missing/ineligible foreground handles. |

These are expected surfaces; implementation should reuse existing settings persistence, capture
error reporting, and window picker eligibility helpers rather than duplicate their logic.

## Edge cases and decisions

- The foreground handle may be zero, become invalid, or cease to be eligible before capture starts;
  treat this as a no-op and do not produce a partial file or clipboard update.
- TinyClips windows, including settings and the screenshot editor, must never be directly captured
  through this shortcut.
- Minimized, cloaked, invisible, untitled, tiny, or otherwise non-selectable windows must not
  bypass the existing window picker rules.
- Elevated windows can be foreground but may not be capturable from a non-elevated TinyClips
  process. Preserve the current error behavior and ensure no partial output is left behind.
- Do not reopen the selector on an invalid target; this is the issue owner's proposed behavior.
- The setting must affect only the dedicated `ScreenshotWindow` shortcut; selecting Window from
  the ordinary capture picker must continue to show the selector.
- If platform APIs can fail after validation, report the capture failure explicitly rather than
  silently treating it as success.

## Validation

Automated tests should verify that the preference defaults off, disabled mode uses the existing
selector route, enabled mode selects only an eligible foreground candidate, and invalid/absent
handles cancel before save or clipboard work. Keep platform API interactions behind a narrow seam
so the eligibility decision remains deterministic and unit-testable.

Manual Windows validation should cover:

- The dedicated Screenshot Window shortcut with the setting off and on.
- Alt+PrintScreen invoked through AutoHotkey without TinyClips UI taking focus first.
- No foreground window, a minimized/invisible window, and TinyClips itself in the foreground.
- Windows on secondary monitors and elevated windows.
- No file or clipboard changes after an ineligible-window no-op or a cancelled flow.
- The general picker Window option remains unchanged.

Run the Windows app build and `TinyClips.Core.Tests` suite following
`.github/instructions/windows-winui.instructions.md` when implementation is undertaken. The
plan-only Markdown PR does not require platform builds.
