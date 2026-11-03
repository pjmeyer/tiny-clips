# Windows region screenshot countdown before selection

**Status:** implementation plan only. No feature code is included in this pull request.

Tracks [upstream issue #424](https://github.com/jamesmontemagno/tiny-clips/issues/424). Today,
the Windows screenshot flow opens the region selector first and runs the countdown only after the
user commits a region. This prevents users from preparing a transient popup or returning to a
full-screen remote desktop session during the countdown.

## Scope and acceptance criteria

- Add an opt-in Screenshot setting, off by default, to run the screenshot countdown before region
  selection. Suggested label: **Start region countdown before selecting the area**.
- With the setting off, keep the current order exactly: select the region, then count down, then
  capture.
- With it on and a screenshot countdown enabled with a positive duration, count down before
  showing any region-selection overlay. Once the timer expires, immediately capture a fresh
  backdrop and open the selector.
- If the countdown is disabled or its duration is zero, go directly to region selection.
- Cancelling during the countdown or region selection produces no capture, saved file, or
  clipboard change.
- Limit the change to Windows screenshot Region captures, whether started from the capture picker
  or the dedicated region-screenshot shortcut. Keep Screen, Window, OCR, video, and GIF ordering
  unchanged.
- Exclude TinyClips' countdown and selector overlays from the resulting screenshot.

## Proposed design

`BeginCaptureAsync` currently resolves every target before the countdown. For the enabled Region
route, split that sequence so the app knows the selected mode and countdown settings before calling
`ResolveTargetAsync`. Run the countdown without a region indicator (the region is not yet known),
discard any backdrop captured while the picker was open, and invoke region selection with a fresh
backdrop after the countdown. Preserve cancellation-token handling and the existing post-capture
save, clipboard, and editor path.

The new persisted preference should default to `false` so existing users retain the current
behavior. Add it to the capture-settings interface and implementation, load and persist it through
the settings view model, and expose an accessible toggle in the Screenshot settings section. Its
description should explain that the selection overlay appears after the timer.

## Likely implementation surfaces

| Area | Files/modules | Responsibility |
|---|---|---|
| Flow orchestration | `windows/src/TinyClips.App/App.xaml.cs` | Defer Region target resolution until after the countdown only when the preference is on. |
| Region overlay | `RegionSelectController` and its backdrop capture path | Ensure the selector uses a fresh post-countdown frame, not an early picker backdrop. |
| Preference | `windows/src/TinyClips.Core/Services/ICaptureSettings.cs`, `CaptureSettings.cs` | Persist the opt-in setting with a false default. |
| Settings UI | `windows/src/TinyClips.App/ViewModels/SettingsViewModel.cs`, `Controls/Settings/ScreenshotSettingsSection.xaml` | Bind the preference and expose accessible, localized-quality copy consistent with existing settings cards. |
| Tests | `windows/tests/TinyClips.Core.Tests` and relevant app-level test seams | Verify route decisions, default behavior, cancellation, and unrelated capture-mode invariants. |

This is a Windows-only request; no macOS files or behavior need to change.

## Edge cases and decisions

- The countdown currently anchors to the selected monitor, but the monitor is unknown before a
  region is chosen. Decide whether to show the timer on the cursor monitor, the preferred/primary
  monitor, or another stable location.
- For multi-monitor mode, ensure the countdown does not open a selector early and that the fresh
  backdrop is captured across the intended monitors.
- A transient popup may appear or disappear between the timer expiring and the selector becoming
  visible; minimize that interval and do not reuse a pre-countdown frame.
- Full-screen RDP preparation should remain possible without TinyClips overlays taking focus before
  the timer expires.
- Cancellation at either stage must release capture overlays and leave no partial output.
- Existing screenshot countdown behavior for Screen and Window, and all video/GIF flows, must be
  unchanged.

## Validation

Automated coverage should verify both preference states, route ordering with and without a
countdown, and that cancellation never reaches screenshot save/clipboard work. Use deterministic
logic tests where possible rather than tests requiring capture permissions or hardware.

Manual Windows validation should cover:

- The setting off (selection then countdown) and on (countdown then selection).
- Countdown disabled, countdown cancelled, and region selection cancelled.
- A transient popup and a full-screen RDP session prepared during the countdown.
- Multi-monitor and mixed-DPI layouts, including the chosen countdown-window placement.
- TinyClips countdown/selector overlays not appearing in the saved capture.
- Screen, Window, OCR, video, and GIF behavior remaining unchanged.

Run the Windows app build and `TinyClips.Core.Tests` suite following
`.github/instructions/windows-winui.instructions.md` when implementation is undertaken. The
plan-only Markdown PR does not require platform builds.
