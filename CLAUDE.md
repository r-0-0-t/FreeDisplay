# FreeDisplay — Claude Context Entry

> Version: 2026-03-05 | Status: Phase 22 complete

## What This Project Is

A free, open-source alternative to BetterDisplay. A macOS menu bar app for display management: DDC brightness/contrast control, resolution/HiDPI management, display arrangement, color management, virtual displays.
Tech stack: Swift 6 + SwiftUI (MenuBarExtra) + IOKit + CoreGraphics, zero third-party dependencies.

## Quick Navigation (on-demand loading)

| What you need to do | Which file to read |
|---------------------|--------------------|
| **Check blocking issues before starting** | `docs/BLOCKING.md` |
| Understand code structure, find files | `docs/codemap/CLAUDE.md` (index) → `docs/codemap/file-tree.md` (file tree) → `docs/codemap/relationships.md` (relationship diagram) |
| View project plan, current progress | `docs/roadmap/CLAUDE.md` (overview) → `docs/roadmap/phase-N.md` (details) |
| View work habits/preferences | `docs/habits.md` |
| View lessons learned / gotchas | `docs/lessons/CLAUDE.md` (index) → `docs/lessons/{topic}.md` (details) |

## Current Focus

- **Current phase**: Phase 22 complete. Feature pruning (removed rotation/streaming/PiP/mirror/protection etc.) + auto-brightness rewrite + HiDPI plist override implementation.
- **Don't touch**: `docs/roadmap/` structure (planner output), only update `[x]` progress markers
- **Recent changes**: Phase 21 feature pruning (deleted 15+ files), Phase 22 auto-brightness rewrite (CoreDisplay dlsym), HiDPI switched from mirror to plist override, arrangement center-alignment fix, HiDPI presets

## Autonomous Decision Rules

> **Blocking first (first thing when starting):**
- Every session → read `docs/BLOCKING.md` first → resolve any P0/P1 items → only then work on ROADMAP
- If stuck on a problem → add it to `docs/BLOCKING.md`

> **Cross-reference rules:**
- Changed a `DisplayInfo` property → grep all references and update them
- Changed `project.yml` → must run `xcodegen generate` to regenerate xcodeproj
- Added new Service/View files → update `docs/codemap/file-tree.md`

> **Fix/development:**
- Build failure → fix until it passes, don't skip
- Swift 6 concurrency errors → use `@MainActor` or `@unchecked Sendable` (project has `SWIFT_STRICT_CONCURRENCY: minimal`)
- New files don't need project.yml changes (xcodegen auto-includes all source files under FreeDisplay/)

> **SwiftUI component rules:**
- Row components that need local state (isHovered, isLoading) → must be standalone `struct`, ❌ cannot be `@ViewBuilder` functions (@ViewBuilder functions don't support @State)
- Reusable row components named consistently: `XxxRow` (e.g. DetailRow, ExpandableRow, ProtectionRowView)

> **UserDefaults key naming convention:**
- All UserDefaults keys must have `fd.` prefix (e.g. `fd.launchAtLogin`, `fd.AutoBrightnessEnabled`)
- ❌ Bare keys (e.g. `"launchAtLogin"`) may conflict with system or third-party keys

> **Cross-Service coordination for shared resources:**
- Two Services cannot independently write to the same CoreGraphics resource (e.g. gamma table) → designate one Service as the owner responsible for final writes
- BrightnessService (software brightness) writes through GammaService, does not call CGSetDisplayTransferByTable directly
- ❌ View layer calling `CGSetDisplayTransferByFormula/Table` directly (bypassing GammaService) → ✅ Use `GammaService.apply()` or `GammaService.resetSingleDisplay()` indirectly
- ❌ `CGDisplayRestoreColorSyncSettings()` (global) → ✅ `GammaService.resetSingleDisplay(displayID)` resets only a single display

> **Sleep/wake handling (required):**
- Services that write display hardware state (gamma, software brightness) must respond to `NSWorkspace.didWakeNotification` to re-apply
- Already registered: AppDelegate listens for wake → GammaService.reapplyIfNeeded + BrightnessService.reapplySoftwareBrightnessIfNeeded

> **C callback Unmanaged rules:**
- Long-lived C callbacks (CGDisplayRegisterReconfigurationCallback etc.) → must use `Unmanaged.passRetained(self)`, call `release()` on unregister
- ❌ `passUnretained` (dangling pointer risk)

> **IOKit display matching:**
- Don't use CGDisplayVendorNumber/ModelNumber to match IOKit services (unreliable for some displays)
- Display names via `NSScreen.localizedName`
- IOKit service lookup via IOServiceGetMatchingServices enumerating IODisplayConnect (❌ CGDisplayIOServicePort is deprecated)
- DDC external brightness control may not be available → UI should degrade gracefully (prompt user on detection failure)

> **Async principle:**
- Only make truly slow operations async (filesystem scanning, network requests)
- Microsecond-level IOKit calls (name lookup, property reads) stay synchronous, not worth async overhead

> **CGVirtualDisplay private API (must follow):**
- `vendorID` must be non-zero (e.g. `0xEEEE`), passing 0 causes `CGVirtualDisplay(descriptor:)` to return nil
- `CGVirtualDisplay(descriptor:)` must be called on the main thread (returns nil from background), `apply(settings)` can be background
- Bridging header property names follow Chromium's `virtual_display_mac_util.mm` (`maxPixelsWide`/`maxPixelsHigh` not `maxPixelSize`)

> **HiDPI implementation (must follow):**
- ❌ `CGConfigureDisplayMirrorOfDisplay` for HiDPI — triggers hardware mirroring + mouse stutter on Apple Silicon
- ✅ Plist override writing to `/Library/Displays/Contents/Resources/Overrides/` — same approach as BetterDisplay
- Writing plist requires admin privileges → use `NSAppleScript("do shell script ... with administrator privileges")`
- ❌ Setting `DisplayProductName` in plist — overrides system display name
- Requires display reconnect to take effect (IOServiceRequestProbe may not be reliable)

> **Private framework dynamic loading:**
- ❌ `@_silgen_name` for private framework symbols (linker undefined symbol)
- ✅ `dlopen` + `dlsym` runtime loading (e.g. CoreDisplay_Display_GetUserBrightness)

> **Stop and ask the user:**
- Need to use private API (CoreDisplay etc.)
- Need SIP disabled or special system permissions
- Architecture direction change (MVVM → something else)

> **Self-maintenance rules:**
- Added/removed/changed files → update `docs/codemap/file-tree.md`
- Phase task completed → mark `[x]` in `docs/roadmap/phase-N.md` **and also in `docs/ROADMAP.md`** (autopilot tracks progress via the latter)
- Hit a gotcha → write to `docs/lessons/{topic}.md` (also update `docs/lessons/CLAUDE.md` index)
- Discovered a preference/pattern → write to `docs/habits.md`
- Stuck on something → add to `docs/BLOCKING.md`
- Resolved a BLOCKING item → move to resolved section

## Verification Chain (run after every change)

```bash
# 1. Build check
cd ~/Desktop/FreeDisplay && xcodebuild -scheme FreeDisplay -configuration Debug build 2>&1 | tail -5

# 2. Cross-reference check (when changing interfaces/models)
grep -r "DisplayInfo\|DisplayManager\|DDCService" FreeDisplay/ --include="*.swift" | grep -v "^Binary"
```

## Common Operations Playbook

### Adding a Feature Section (most common operation for Phase 2-12)
1. Create new Service in `Services/` (e.g. `BrightnessService.swift`)
2. Create matching View in `Views/` (e.g. `BrightnessSliderView.swift`)
3. If state management needed, create ViewModel in `ViewModels/`
4. Embed the new View in `MenuBarView.swift`
5. Update `DisplayInfo.swift` with needed properties
6. Run verification chain

### Implementing DDC Features
1. Implement IOKit I2C communication in `DDCService.swift`
2. Look up VCP code in DDC/CI standard for the feature (e.g. 0x10 = brightness)
3. Call `DDCService.shared.read/write` from the Service
4. Run verification chain + manually test on an actual external display

### Fixing Bugs
1. Confirm whether it's a compile error or runtime error
2. Compile error → look at xcodebuild output to locate
3. Runtime error → check Console.app logs or Xcode debugger
4. Fix → run verification chain

## Design Resources

- **App icon design**: Use [Nano Banana](https://nano-banana.ai/) (Google Gemini-powered AI image generator) to generate high-quality icons
  - Supports text descriptions for generating icons, logos, UI elements
  - After generation, use Python PIL to crop/scale to macOS multi-resolution PNGs (16/32/64/128/256/512/1024)
  - Icon files located at `FreeDisplay/Assets.xcassets/AppIcon.appiconset/`

## Key Conventions

- **Language**: Swift 6.0 (concurrency checking set to minimal)
- **Minimum OS**: macOS 14.0
- **Architecture**: MVVM (View → ViewModel → Service)
- **Build**: `xcodegen generate && xcodebuild -scheme FreeDisplay -configuration Debug build`
- **No Sandbox**: entitlements have App Sandbox disabled (required for DDC/IOKit)
- **No third-party dependencies**: all system frameworks

## Core Frameworks

| Framework | Purpose | Phase |
|-----------|---------|-------|
| CoreGraphics | Display enumeration, resolution, arrangement | 1-4 |
| IOKit | DDC/CI I2C communication, brightness/contrast | 2 |
| ColorSync | ICC Profile management | 5 |
| CoreGraphics (CGVirtualDisplay) | Virtual displays | 10 |
| CoreDisplay (dlsym) | Built-in display brightness reading | 22 |
