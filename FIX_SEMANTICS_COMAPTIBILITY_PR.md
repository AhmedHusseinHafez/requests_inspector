# Fix semantics compatibility for QA/Maestro by opening inspector as a route

## Summary

In our project, we enable `requests_inspector` in the QA build variant so the team can debug network requests during testing. That setup broke **Flutter semantics**, which our QA team relies on to write integration tests with **Maestro Studio**.

The root cause was the inspector being embedded in a `PageView` alongside the app (with a nested `MaterialApp`), which kept an extra widget subtree in the tree and interfered with the semantics/accessibility layer Maestro uses to find and interact with UI elements.

This PR fixes that by:

- Showing the inspector via `Navigator.push` instead of a `PageView` page swap, so the main app widget tree stays clean when the inspector is closed
- Replacing the nested `MaterialApp` in `Inspector` with a `Theme` + `Scaffold`, avoiding a second app root in the semantics tree
- Updating `hideInspector` to use `Navigator.pop` for a consistent open/close flow

With these changes, semantics work correctly in QA builds while the inspector remains available for debugging.

**Full thanks to the package team** for maintaining this tool and for accepting this improvement — it unblocks our QA automation workflow with Maestro.

## Problem

When `RequestsInspector` was enabled in our QA flavor:

- Maestro Studio could not reliably locate elements via semantics
- Integration tests that depend on semantic identifiers/labels failed or behaved inconsistently
- The issue traced to the inspector's nested `MaterialApp` + always-present `PageView` structure

## Solution

| Before | After |
|--------|--------|
| Inspector always in `PageView` (page 0 = app, page 1 = inspector) | Inspector pushed only when opened |
| Nested `MaterialApp` inside inspector | `Theme` + `Scaffold` only |
| `hideInspector()` via `pageController.jumpToPage(0)` | `hideInspector(context)` via `Navigator.pop` |

## Test plan

- [ ] Run the example app with inspector enabled; long-press opens the inspector
- [ ] Close button dismisses the inspector and returns to the app
- [ ] Verify semantics tree in QA build (e.g. Flutter DevTools → Semantics, or Maestro Studio element picker)
- [ ] Confirm Maestro flows can find and tap app widgets when inspector is **closed**
- [ ] Confirm inspector still captures and displays network requests as before
- [ ] Smoke test on iOS and Android QA variants
