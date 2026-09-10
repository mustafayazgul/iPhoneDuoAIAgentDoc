# iPhone Duo AI Agent Doc

A knowledge pack that teaches AI coding agents how to adapt existing iOS apps for **iPhone Duo**, Apple's first foldable, dual-display iPhone (announced September 9, 2026, shipping October 23, 2026).

## Why this repo exists

Every LLM has a training cutoff. Models trained before September 2026 know nothing about iPhone Duo: its dual displays, the fold, vertical toolbars, reserved regions, arrangements, hinge APIs, or the iOS 27.1 SDK. Until models trained on this material are released, AI coding agents (Claude Code, Cursor, Copilot, and others) will hallucinate or simply miss what an iPhone Duo migration requires.

This repo closes that gap: drop the guide into your agent's context and it can plan and execute an iPhone Duo adaptation with the correct APIs, design rules, and test matrix.

## What's inside

| File | Description |
|---|---|
| `IPHONE_DUO_MIGRATION.md` | The full migration guide: device facts, SDK behavior tiers, an 8-phase migration workflow (audit, layout, bars, fold-aware layout, multitasking, camera, design rules, verification matrix), and a quick API reference for SwiftUI, UIKit, and AVFoundation |

Key topics covered:

- Size classes on the outer (compact) and inner (regular/regular) displays, and why idiom, orientation, and `UIScreen.main` checks break
- Asymmetric safe areas, layout margins, and Concentricity APIs
- Vertical navigation, toolbar, and tab bars: `axisBehavior`, visibility priorities, overflow menus, opt-out
- Reserved regions (`.division` for the fold, `.occlusion` for cameras) and displacement patterns
- The new `ArrangementView` / `UIArrangementViewController` layout containers (`.split` and `.overlay`)
- Hinge APIs (`onHingeChange`, `UIHingeInteraction`) for interactions, not layout
- Split View multitasking, multiple scenes, scene accessories, and the camera capture accessory
- Camera adaptation: Virtual Front Camera, `AVCaptureDeviceDirectionCoordinator`, dynamic aspect ratios
- Touch ID instead of Face ID: `LAContext.biometryType` handling
- A 14-configuration verification matrix for the iPhone Duo simulator (Device Hub, Xcode 27.1)

## How to use it

**Claude Code:** copy `IPHONE_DUO_MIGRATION.md` into your project and reference it from `CLAUDE.md`:

```markdown
# CLAUDE.md
When adapting this app for iPhone Duo, follow IPHONE_DUO_MIGRATION.md.
```

Then prompt something like: "Adapt this app for iPhone Duo following IPHONE_DUO_MIGRATION.md, starting with the Phase 0 audit."

**Cursor / other agents:** add the file to your rules directory (for example `.cursor/rules/`) or attach it to the conversation context before asking for the migration.

**Any chat LLM:** paste the guide into the conversation before asking iPhone Duo questions.

## Sources

Compiled from material Apple published on September 9, 2026:

- Six Tech Talks sessions: Prepare your app for iPhone Duo, Design for iPhone Duo, Strike a pose with adaptive layouts, Leverage multiple displays and scenes, Raise the bar with iPhone Duo, Build a great camera experience for iPhone Duo
- Apple Developer News: "Get ready for iPhone Duo"
- Human Interface Guidelines: "Designing for iPhone Duo"
- Apple Newsroom: "Apple unveils iPhone Duo"

Full links are listed at the bottom of the guide.

## Accuracy notes

- API names were normalized from session transcripts and session pages; verify exact signatures against the iOS 27.1 SDK headers in Xcode 27.1 before shipping.
- Logical point dimensions and some bar-ordering details come from secondary reporting and are flagged as such in the guide.
- This is an unofficial community resource, not affiliated with or endorsed by Apple.

Last updated: September 10, 2026
