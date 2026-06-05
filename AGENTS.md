# AGENTS.md — FreeDisplay Project Harness Configuration

> This file defines the harness rules for AI agents working on this project.
> AI agents MUST read this file before making any changes.
> For deeper context, read CLAUDE.md first.

## Project Overview

- **Name**: FreeDisplay
- **Language**: Swift 6.0 (concurrency checking set to minimal)
- **Framework**: SwiftUI (MenuBarExtra) + AppKit
- **Package Manager**: None (zero third-party dependencies, all system frameworks)
- **Build Tool**: XcodeGen (`project.yml`) + xcodebuild
- **Minimum OS**: macOS 14.0
- **Architecture**: MVVM — View → ViewModel → Service

## Architecture Rules

### Layer Separation

```
Views/          → UI presentation, reads ViewModel or Service directly (simple cases)
ViewModels/     → State management, bridges View and Service
Services/       → Business logic, interacts with system frameworks (IOKit/CoreGraphics/DDC)
Models/         → Pure data structures (DisplayInfo, DisplayMode, DisplayPreset)
```

- View layer must NOT call CoreGraphics / IOKit / CGSet* series APIs directly
- Gamma table writes must go through GammaService, never bypass it
- BrightnessService (software brightness) writes through GammaService, does not call CGSetDisplayTransferByTable directly

### Module Boundaries

| Directory | Responsibility |
|-----------|---------------|
| `FreeDisplay/Services/` | All system-level operations (DDC, brightness, resolution, HiDPI, arrangement, etc.) |
| `FreeDisplay/Views/` | SwiftUI views, filename format: `XxxView.swift` or `XxxRow.swift` |
| `FreeDisplay/ViewModels/` | State management, 1:1 with View or shared across multiple Views |
| `FreeDisplay/Models/` | Data structures, no side effects |
| `FreeDisplay/Utilities/` | General utility functions |
| `FreeDisplay/Resources/` | Static resources |
| `docs/` | Project documentation (do not change structure) |
| `scripts/` | Build and release scripts |

### Protected Files

The following files require explicit justification before modification:

- `FreeDisplay/FreeDisplay.entitlements` — Entitlements declaration, changes affect signing and App Sandbox
- `project.yml` — XcodeGen configuration, must re-run `xcodegen generate` after changes
- `ExportOptions.plist` — Release signing configuration
- `docs/roadmap/` — Planning documents, only update `[x]` progress markers, do not change structure

## Coding Standards

### Style Guide

- Swift 6.0 syntax, do not downgrade to compatibility syntax
- SwiftUI views preferred, use AppKit only when necessary
- Private frameworks (CoreDisplay etc.) must use `dlopen` + `dlsym` runtime loading, prohibit `@_silgen_name`

### Naming Conventions

- Service classes: `XxxService.swift`, singleton via `static let shared`
- View files: `XxxView.swift`
- Reusable row components: `XxxRow` (struct, supports `@State`)
- UserDefaults keys: must have `fd.` prefix (e.g. `fd.launchAtLogin`)
- Prohibit bare keys (e.g. `"launchAtLogin"`)

### SwiftUI Component Rules

- Row components requiring local state (`isHovered`, `isLoading`) → must be standalone `struct`
- Prohibit using `@ViewBuilder` functions to host components with `@State`

### Concurrency Rules

- Swift 6 concurrency errors → prefer `@MainActor` or `@unchecked Sendable`
- Long-lived C callbacks (e.g. CGDisplayRegisterReconfigurationCallback) → use `Unmanaged.passRetained(self)`, `release()` on unregister
- Prohibit `passUnretained` (dangling pointer risk)
- Only async truly slow operations (filesystem scanning, network requests); microsecond-level IOKit calls stay synchronous

## Testing Requirements

### Verification Commands

```bash
# Build check (Debug) — run after every change
cd ~/Desktop/FreeDisplay && xcodebuild -scheme FreeDisplay -configuration Debug build 2>&1 | tail -20

# Cross-reference check (when changing interfaces/models)
grep -r "DisplayInfo\|DisplayManager\|DDCService" FreeDisplay/ --include="*.swift" | grep -v "^Binary"

# Regenerate xcodeproj (required after changing project.yml)
cd ~/Desktop/FreeDisplay && xcodegen generate

# Release build + DMG packaging
cd ~/Desktop/FreeDisplay && ./build.sh
```

### Test Coverage

- This project has no automated test suite (heavily hardware-dependent, manual testing primary)
- New DDC features must be manually verified on an actual external display
- New HiDPI features must be verified after reconnecting the display

## Git Discipline

### Branch Naming

- `feature/xxx` — new feature
- `fix/xxx` — bug fix
- `refactor/xxx` — refactoring
- `phase-N/xxx` — corresponds to ROADMAP Phase changes

### Commit Message Format

Follow Conventional Commits:

```
feat: add auto brightness adjustment
fix: fix HiDPI plist write permission issue
refactor: extract GammaService for unified transfer function management
```

### Co-Author Line

```
Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

## Forbidden Operations

- Do NOT: Call `CGSetDisplayTransferByTable` / `CGSetDisplayTransferByFormula` directly (bypassing GammaService)
- Do NOT: Call `CGDisplayRestoreColorSyncSettings()` (global reset) — use `GammaService.resetSingleDisplay(displayID)`
- Do NOT: Use `CGConfigureDisplayMirrorOfDisplay` for HiDPI (triggers hardware mirroring + mouse stutter on Apple Silicon)
- Do NOT: Use `@_silgen_name` for private framework symbols (linker undefined symbol)
- Do NOT: Use `CGDisplayVendorNumber/ModelNumber` to match IOKit services (unreliable for some displays)
- Do NOT: Set `DisplayProductName` in plist (overrides system display name)
- Do NOT: Modify `docs/roadmap/` directory structure
- Do NOT: Add third-party dependencies (project policy: zero dependencies)
- Do NOT: Access system framework low-level APIs directly from the View layer

## Agent-Specific Notes

### Startup Checklist

1. Read `docs/BLOCKING.md` first, resolve any P0/P1 items
2. Read `docs/roadmap/CLAUDE.md` to understand the current Phase
3. Read `docs/codemap/CLAUDE.md` → `docs/codemap/file-tree.md` to locate relevant files

### Post-Change Cross-Reference Checks

- Changed `DisplayInfo` properties → grep all reference points and update them
- Changed `project.yml` → must run `xcodegen generate`
- Added new Service/View files → update `docs/codemap/file-tree.md`
- Phase task completed → mark `[x]` in both `docs/roadmap/phase-N.md` and `docs/ROADMAP.md`
- Hit a gotcha → write to `docs/lessons/{topic}.md` and update the index
- Sleep/wake related changes → confirm Service responds to `NSWorkspace.didWakeNotification`

### When to Stop and Ask the User

- Need to use a new private API (CoreDisplay etc.)
- Need SIP disabled or special system permissions
- Architecture direction change (MVVM → other pattern)

### Core Framework Quick Reference

| Framework | Purpose |
|-----------|---------|
| CoreGraphics | Display enumeration, resolution, arrangement |
| IOKit | DDC/CI I2C communication, brightness/contrast |
| ColorSync | ICC Profile management |
| CGVirtualDisplay (private) | Virtual displays, vendorID must be non-zero, call on main thread |
| CoreDisplay (dlsym) | Built-in display brightness reading |

<!-- Generated by Harness Engineering system on 2026-03-12. Review and customize. -->
