# DragController Lifecycle and Navigation Interaction

## Overview

The `DragController` manages drag and drop operations across the page. Understanding its lifecycle and interaction with navigation is crucial for diagnosing the drag-across-navigation bug.

## DragController Architecture

### Location and Purpose
- **File**: `third_party/blink/renderer/core/page/drag_controller.h/.cc`
- **Purpose**: Centralized management of drag and drop operations
- **Scope**: Page-level (not frame-level) - one DragController per Page

### Key Characteristics
- **Inheritance**: `ExecutionContextLifecycleObserver` - gets notified when execution context is destroyed
- **Lifetime**: Tied to Page lifetime, survives frame navigations
- **State Management**: Maintains drag state across the entire page

## Key Members and State

### Core State Variables
```cpp
class DragController : public ExecutionContextLifecycleObserver {
  Member<Page> page_;                    // The page this controller serves
  Member<Document> document_under_mouse_; // Current document being dragged over
  Member<LocalDOMWindow> drag_initiator_; // Window that started the drag
  Member<DragState> drag_state_;         // Current drag operation state
  bool document_is_handling_drag_;       // Whether document handles drag events
  bool did_initiate_drag_;              // Whether this controller initiated drag
  // ... other members
};
```

### Lifecycle Methods
```cpp
void ContextDestroyed() final;  // Called when execution context is destroyed
```

## The Page-Level Scope Issue

### Key Insight: Frame vs Page Scope
- **DragController**: Page-scoped (survives frame navigations)
- **LocalFrame**: Frame-scoped (may be detached/replaced during navigation)
- **EventHandler**: Frame-scoped (one per LocalFrame)

### Problem Analysis
1. **Drag Initiated**: User starts drag in Frame A
2. **Navigation Triggered**: Frame A navigates via `window.location`
3. **Frame Transition**: Frame A's document changes, but same Frame object may be reused
4. **DragController Persists**: Page-level DragController maintains drag state
5. **EventHandler Confusion**: Frame A's EventHandler continues processing events for transitioning frame

## Critical Methods

### DragController::ContextDestroyed()
```cpp
void DragController::ContextDestroyed() {
  drag_state_ = nullptr;
}
```

**Analysis**:
- Only clears drag state when the **entire execution context** is destroyed
- Does NOT get called during normal frame navigation (same execution context persists)
- This means drag state persists across `window.location` navigation

### State Management During Navigation
The controller maintains several pieces of state that may become stale:
- `document_under_mouse_` - May point to old/detaching document
- `drag_initiator_` - May reference old window context
- `drag_state_` - Contains references to old DOM elements

## Navigation Interaction Patterns

### Same-Page Navigation (`window.location`)
1. **Before Navigation**: DragController has active drag state
2. **Navigation Start**: Document begins detaching
3. **Frame Reuse**: Same LocalFrame object, new Document
4. **Continued Processing**: EventHandler continues calling DragController
5. **Stale State**: DragController state references old document/elements

### Cross-Page Navigation
1. **Before Navigation**: DragController has active drag state
2. **Page Replacement**: Entire Page object replaced
3. **Controller Destruction**: Old DragController destroyed, new one created
4. **Clean State**: No stale drag state

## The Root Cause

### Frame-Page State Mismatch
```
LocalFrame (Frame-scoped)        Page (Page-scoped)
├── EventHandler                ├── DragController
│   └── Processes mouse events  │   └── Maintains drag state
├── Document (being detached)   └── Persists across navigation
└── New Document (loading)
```

### Problematic Sequence
1. **EventHandler::UpdateDragAndDrop** called during navigation
2. **Hit testing** performed on frame with detaching document
3. **DragController** called with stale state references
4. **Inconsistent state** between frame (transitioning) and page (stable)

## Potential Solutions

### 1. Navigation-Aware Drag Cancellation
Monitor navigation events and cancel drag operations:
```cpp
// In DragController or EventHandler
if (frame->GetDocument()->IsDetaching() ||
    frame->Loader().HasProvisionalDocumentLoader()) {
  DragEnded();  // Cancel current drag operation
  return;
}
```

### 2. Document Lifecycle Synchronization
Update drag state when document lifecycle changes:
```cpp
void DragController::DocumentDetaching(Document* document) {
  if (document_under_mouse_ == document) {
    DragEnded();
  }
}
```

### 3. Frame-Level Drag State Isolation
Consider moving drag state to frame level or adding frame-level hooks:
```cpp
// In LocalFrame
void LocalFrame::DetachDocument() {
  // Cancel any active drag operations for this frame
  if (GetPage()->GetDragController().IsFrameDragging(this)) {
    GetPage()->GetDragController().CancelFrameDrag(this);
  }
  return Loader().DetachDocument();
}
```

## Investigation Points

### 1. State References During Navigation
- What DOM elements does `drag_state_` reference during navigation?
- Are these references becoming invalid/stale?

### 2. Event Timing Analysis
- When exactly does `EventHandler::UpdateDragAndDrop` get called during navigation?
- What is the document/frame state at those moments?

### 3. Cross-Frame Interactions
- Does drag state properly handle multiple frames?
- How does navigation in one frame affect drag operations in others?

## Recommended Next Steps

1. **Add Navigation Hooks**: Instrument DragController to log navigation events
2. **State Validation**: Add checks for stale references in drag state
3. **Lifecycle Integration**: Consider integrating with document lifecycle observers
4. **Frame Isolation**: Investigate frame-level drag state management

## Test Scenarios

### Critical Test Cases
1. **Single Frame Navigation**: Drag during `window.location` in main frame
2. **Iframe Navigation**: Drag during navigation in iframe
3. **Parent-Child Interaction**: Drag from parent, navigate iframe
4. **Multiple Navigation**: Multiple rapid `window.location` calls during drag

Each scenario should validate:
- Drag state consistency
- Hit testing validity
- Event propagation correctness
- Resource cleanup
