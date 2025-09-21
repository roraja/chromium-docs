---
date: 2025-01-09T14:30:00-08:00
researcher: GitHub Copilot
git_commit: PLACEHOLDER
branch: PLACEHOLDER
repository: chromium
topic: "Drag and drop between documents doesn't work with local HTML files"
tags: [research, codebase, drag-drop, navigation, security, file-urls, cross-document]
status: complete
last_updated: 2025-01-09
last_updated_by: GitHub Copilot
---

# Research: Drag and drop between documents doesn't work with local HTML files

**Date**: 2025-01-09T14:30:00-08:00
**Researcher**: GitHub Copilot
**Git Commit**: PLACEHOLDER
**Branch**: PLACEHOLDER
**Repository**: chromium

## Research Question
Why do drag and drop operations between documents fail when opening local HTML files, specifically navigation tests that redirect during drag operations, but work correctly when served via wpt.live?

## Summary
The issue is caused by a security feature in Chromium's drag controller that prevents cross-document drag operations when the drag was initiated by the same document. The `did_initiate_drag_` flag in `DragController` blocks drop operations to prevent potential security issues with navigation during drag operations. This affects local file:// URLs more severely than web-served content due to different origin policies and navigation behaviors.

## Detailed Findings

### Core Issue: Self-Initiated Drag Blocking

The primary mechanism blocking these drag operations is found in `DragController::GetDragOperation()`:

**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:1411`](/third_party/blink/renderer/core/page/drag_controller.cc#L1411)
```cpp
return drag_data->ContainsURL() && !did_initiate_drag_ ? DragOperation::kCopy
                                                       : DragOperation::kNone;
```

This logic specifically returns `DragOperation::kNone` when `did_initiate_drag_` is true, effectively preventing any drag operation initiated by the current document from being accepted as a drop.

### Drag Initiation Tracking

The `did_initiate_drag_` flag is set when a drag operation starts:

**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:1390`](/third_party/blink/renderer/core/page/drag_controller.cc#L1390)
```cpp
void DragController::DoSystemDrag(...) {
  did_initiate_drag_ = true;
  drag_initiator_ = frame->DomWindow();
  // ...
}
```

And reset when drag ends:

**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:223`](/third_party/blink/renderer/core/page/drag_controller.cc#L223)
```cpp
void DragController::DragEnded() {
  did_initiate_drag_ = false;
  // ...
}
```

### Navigation During Drag Problem

The WPT navigation tests (e.g., `001.xhtml`) use this pattern:
1. Start drag with `ondragstart`
2. Immediately navigate with `window.location = '001-1.xhtml'`
3. Attempt to drop on the new document

This creates a situation where:
- The drag is initiated in document A
- Navigation occurs to document B
- The drop attempt happens in document B
- But the `DragController` still considers this a self-initiated drag

### Additional Load Operation Blocking

There's also specific blocking in `OperationForLoad()`:

**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:525-527`](/third_party/blink/renderer/core/page/drag_controller.cc#L525)
```cpp
if (doc &&
    (did_initiate_drag_ || IsA<PluginDocument>(doc) || IsEditable(*doc)))
  return DragOperation::kNone;
```

This prevents loading operations (which would include navigation-based drops) when the drag was self-initiated.

### Same-Document Check in CanProcessDrag

There's also logic to prevent dropping in the same document where selection occurred:

**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:785-787`](/third_party/blink/renderer/core/page/drag_controller.cc#L785)
```cpp
if (did_initiate_drag_ &&
    document_under_mouse_ ==
        (drag_initiator_ ? drag_initiator_->document() : nullptr)) {
  // Prevent dropping in the same location where selection was made
  return !result.IsSelected(HitTestLocation(point_in_frame));
}
```

### File URL vs Web URL Behavior Differences

The behavior difference between local files and web-served content likely stems from:

1. **Origin Policies**: File URLs have unique origins, making cross-document operations more restrictive
2. **Navigation Timing**: Local file navigation may be synchronous vs asynchronous for web content
3. **Security Context**: File URLs may maintain security context differently during navigation

### Test Examples Analysis

The navigation tests show the problematic pattern:

**File**: [`third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/001.xhtml`](/third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/001.xhtml)
```javascript
function start(event) {
  event.dataTransfer.effectAllowed = 'copy';
  event.dataTransfer.setData('text/uri-list', document.querySelector('canvas').toDataURL('image/png'));
  window.location = '001-1.xhtml';  // Navigation during dragstart
}
```

This immediately navigates during the drag start event, which likely maintains the `did_initiate_drag_` state across the navigation boundary.

## Architecture Insights

- **Security-First Design**: The drag controller prioritizes security by preventing potentially dangerous cross-document drag operations
- **State Persistence**: Drag state persists across navigation, which is intentional for security but causes issues with legitimate test cases
- **Origin-Based Restrictions**: File URLs are treated with additional security restrictions compared to web URLs

## Related Code References

- [`third_party/blink/renderer/core/page/drag_controller.cc:1411`](/third_party/blink/renderer/core/page/drag_controller.cc#L1411) - Main blocking logic in GetDragOperation
- [`third_party/blink/renderer/core/page/drag_controller.cc:1390`](/third_party/blink/renderer/core/page/drag_controller.cc#L1390) - Drag initiation flag setting
- [`third_party/blink/renderer/core/page/drag_controller.cc:223`](/third_party/blink/renderer/core/page/drag_controller.cc#L223) - Drag end flag reset
- [`third_party/blink/renderer/core/page/drag_controller.cc:525-527`](/third_party/blink/renderer/core/page/drag_controller.cc#L525) - Load operation blocking
- [`third_party/blink/renderer/core/page/drag_controller.cc:785-787`](/third_party/blink/renderer/core/page/drag_controller.cc#L785) - Same document prevention

## Open Questions

1. **Intentional Security Feature**: Is this blocking behavior intentional security protection or an unintended side effect?
2. **Navigation State Management**: Should drag state be reset or maintained across navigations?
3. **File URL Special Handling**: Should file URLs have different drag/drop policies than web URLs?
4. **WPT Test Validity**: Are these navigation-during-drag tests testing valid web platform behavior?

## Potential Solutions

1. **Reset drag state on navigation**: Clear `did_initiate_drag_` when navigating to a new document
2. **Origin-based checks**: Allow drops between file URLs with specific origin relationships
3. **Navigation-aware drag handling**: Special handling for drags that cross navigation boundaries
4. **Test environment differences**: Different handling for test environments vs production

## Historical Context

This appears to be intentional security hardening to prevent malicious websites from exploiting drag operations during navigation. The behavior difference between local files and web content suggests this security model may be more restrictive for file:// URLs as they're considered a higher security risk context.
