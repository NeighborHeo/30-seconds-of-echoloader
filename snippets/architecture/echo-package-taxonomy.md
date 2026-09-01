---
title: "Echo Package Taxonomy: The ~50 Bounded Subsystems"
category: architecture
tags: [packages, taxonomy, echoroot, echoscanner, gcviewer, msbuild]
difficulty: beginner
---

`src/packages/` contains ~50 `Echo*` packages, each a bounded subsystem compiled as a separate MSBuild `.vcxproj` — knowing which package owns what prevents cross-boundary coupling.

## Structure

```
src/packages/
├── EchoScanner/        ← acquisition engine, SWE/UGAP mode mgmt (Layer A lives here)
├── EchoRoot/           ← MFC shell, boot/auth, ApplicationTouchPanel
├── EchoWorksheet/      ← reporting, measurements, worksheet UI
├── EchoArchive/        ← DICOM/image storage, clipboard
├── EchoReport/         ← report generation, RepDesigner
├── EchoUserManager/    ← auth, user roles, audit log
└── ...~44 more Echo* packages

src/GcViewer/           ← NOT an Echo* package; ObjectViewer rendering layer (Layer B lives here)
```

## Why It Exists

Each package is independently buildable and has a clear ownership boundary — this is how a ~40k-file C++ codebase stays navigable. `GcViewer` is intentionally outside the `Echo*` namespace because it is a reusable rendering framework, not a scanner-specific subsystem.

## Key Points

- **No CMake** — every package is an MSBuild `.vcxproj`; project references define the dependency graph
- `EchoScanner` depends on `GcViewer`, never the reverse — the rendering layer is downstream
- `EchoRoot` is the MFC application entry point; it bootstraps all other packages at startup
- When adding code, ask which package boundary it belongs to before creating a new file — most things already have a home
