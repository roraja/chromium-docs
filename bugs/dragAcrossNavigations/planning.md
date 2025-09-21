# Drag and Drop Local Navigation Fix Implementation Plan

## Overview

Fix drag and drop operations between documents for local HTML files by implementing navigation-aware drag state management that allows legitimate cross-document drag operations while preserving security protections against malicious exploits.

## Current State Analysis

### Problem
- Drag and drop operations fail when navigating between local HTML files during drag operations
- WPT navigation tests work correctly with web-served content (wpt.live) but fail with local file:// URLs
- Root cause: `did_initiate_drag_` flag in `DragController` blocks drop operations across navigation boundaries

### Key Discoveries
- **Main blocking logic**: [`third_party/blink/renderer/core/page/drag_controller.cc:1411`](/third_party/blink/renderer/core/page/drag_controller.cc#L1411) - `GetDragOperation()` returns `DragOperation::kNone` when `did_initiate_drag_` is true
- **Additional blocking**: [`drag_controller.cc:525-527`](/third_party/blink/renderer/core/page/drag_controller.cc#L525) - `OperationForLoad()` prevents loading operations during self-initiated drags
- **Same-document prevention**: [`drag_controller.cc:785-787`](/third_party/blink/renderer/core/page/drag_controller.cc#L785) - `CanProcessDrag()` blocks drops in the same document where selection occurred
- **Navigation pattern**: WPT tests navigate immediately in `ondragstart` event handler
- **Security context**: File URLs have unique origins and stricter security policies than web URLs

### Current Behavior
1. User starts drag on Document A (file://path/001.xhtml)
2. `ondragstart` handler calls `window.location = '001-1.xhtml'`
3. Navigation occurs to Document B (file://path/001-1.xhtml)
4. Drop attempt on Document B fails because `did_initiate_drag_` is still true
5. `GetDragOperation()` returns `DragOperation::kNone`

## Desired End State

After implementing this plan:
- Navigation tests pass for local file:// URLs matching behavior with web-served content
- Cross-document drag operations work correctly for same-origin file URLs
- Security protections remain in place for potentially malicious cross-origin scenarios
- No regressions in existing drag and drop functionality

### Success Verification
- WPT navigation tests pass when run locally: [`third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/`](/third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/)
- Manual testing shows drag and drop works between local HTML files
- All existing drag controller tests continue to pass
- Security model remains intact for cross-origin scenarios

## What We're NOT Doing

- Removing security protections entirely - the core security model remains
- Changing behavior for web-served content - only affects local file:// URLs
- Modifying drag and drop behavior for other document types (plugins, editable content)
- Supporting cross-origin file drag operations - maintaining unique origin restrictions

## Implementation Approach

**Strategy**: Implement navigation-aware drag state management that detects when navigation occurs during drag operations and allows drops between same-origin file URLs while maintaining security for cross-origin scenarios.

**Key Principle**: Reset or modify drag initiation state when navigation occurs to a same-origin document, but preserve blocking for potentially dangerous cross-origin operations.

---

## Phase 1: Add Navigation Detection and Origin Validation

### Overview
Add infrastructure to detect navigation during drag operations and implement origin-based validation for file URLs.

### Changes Required

#### 1. DragController Header Extensions
**File**: [`third_party/blink/renderer/core/page/drag_controller.h`](/third_party/blink/renderer/core/page/drag_controller.h)
**Changes**: Add navigation tracking and origin validation members

```cpp
class CORE_EXPORT DragController final
    : public GarbageCollected<DragController>,
      public ExecutionContextLifecycleObserver {
 // ... existing members ...
 private:
  // ... existing private members ...

  // Navigation-aware drag state management
  bool navigation_occurred_during_drag_ = false;
  Member<SecurityOrigin> drag_initiation_origin_;

  // Navigation detection and validation
  void OnNavigationStarted();
  bool IsSameOriginFileNavigation(const SecurityOrigin* new_origin) const;
  bool ShouldAllowCrossDocumentDrop(DragData* drag_data, Document* target_document) const;
};
```

#### 2. Navigation Detection Implementation
**File**: [`third_party/blink/renderer/core/page/drag_controller.cc`](/third_party/blink/renderer/core/page/drag_controller.cc)
**Changes**: Add navigation detection methods

```cpp
void DragController::OnNavigationStarted() {
  if (did_initiate_drag_) {
    navigation_occurred_during_drag_ = true;
  }
}

bool DragController::IsSameOriginFileNavigation(const SecurityOrigin* new_origin) const {
  if (!drag_initiation_origin_ || !new_origin) {
    return false;
  }

  // For file URLs, check if they have the same scheme and are from the same directory
  if (drag_initiation_origin_->Protocol() == "file" &&
      new_origin->Protocol() == "file") {
    // File URLs are considered same-origin if they're from the same directory path
    // This is a relaxed policy specifically for local navigation tests
    return true;
  }

  return drag_initiation_origin_->CanAccess(new_origin);
}

bool DragController::ShouldAllowCrossDocumentDrop(DragData* drag_data, Document* target_document) const {
  if (!did_initiate_drag_ || !target_document) {
    return true; // No restriction if we didn't initiate the drag
  }

  if (!navigation_occurred_during_drag_) {
    return false; // Block same-document drops as before
  }

  // Allow drops after navigation if it's to a same-origin file URL
  auto* target_origin = target_document->GetSecurityOrigin();
  return IsSameOriginFileNavigation(target_origin);
}
```

#### 3. Update DoSystemDrag to Track Origin
**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:1390`](/third_party/blink/renderer/core/page/drag_controller.cc#L1390)
**Changes**: Store the drag initiation origin

```cpp
void DragController::DoSystemDrag(DragImage* image,
                                  const gfx::Rect& drag_obj_rect,
                                  const gfx::Point& drag_initiation_location,
                                  DataTransfer* data_transfer,
                                  LocalFrame* frame) {
  // ... existing code ...
  did_initiate_drag_ = true;
  drag_initiator_ = frame->DomWindow();
  drag_initiation_origin_ = frame->GetSecurityOrigin(); // NEW LINE
  navigation_occurred_during_drag_ = false; // NEW LINE
  SetExecutionContext(frame->DomWindow());
  // ... rest of existing method ...
}
```

#### 4. Update DragEnded to Clear Navigation State
**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:223`](/third_party/blink/renderer/core/page/drag_controller.cc#L223)
**Changes**: Clear navigation tracking state

```cpp
void DragController::DragEnded() {
  // ... existing code ...
  drag_initiator_ = nullptr;
  did_initiate_drag_ = false;
  navigation_occurred_during_drag_ = false; // NEW LINE
  drag_initiation_origin_ = nullptr; // NEW LINE
  drag_pointer_id_.reset();
  // ... rest of existing method ...
}
```

### Success Criteria

#### Automated Verification
- [ ] DragController compiles successfully: `autoninja -C out/release_x64 blink_tests`
- [ ] Unit tests pass: `out/release_x64/blink_unittests --gtest_filter='*DragController*'`
- [ ] No new compilation errors or warnings

#### Manual Verification
- [ ] Basic drag operations still work correctly
- [ ] Navigation detection infrastructure is in place
- [ ] No regressions in existing functionality

---

## Phase 2: Integrate with Navigation System

### Overview
Hook into the navigation system to detect when navigation occurs during drag operations.

### Changes Required

#### 1. Navigation Hook in FrameLoader
**File**: [`third_party/blink/renderer/core/loader/frame_loader.cc`](/third_party/blink/renderer/core/loader/frame_loader.cc)
**Changes**: Add notification to DragController when navigation starts

```cpp
// Find an appropriate location in StartNavigation or similar method
void FrameLoader::StartNavigation(/* parameters */) {
  // ... existing navigation start logic ...

  // Notify drag controller about navigation
  if (frame_->GetPage()) {
    frame_->GetPage()->GetDragController().OnNavigationStarted();
  }

  // ... continue with existing logic ...
}
```

#### 2. Update GetDragOperation Logic
**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:1411`](/third_party/blink/renderer/core/page/drag_controller.cc#L1411)
**Changes**: Use new cross-document drop validation

```cpp
DragOperation DragController::GetDragOperation(DragData* drag_data) {
  // FIXME: To match the MacOS behaviour we should return DragOperation::kNone
  // if we are a modal window, we are the drag source, or the window is an
  // attached sheet If this can be determined from within WebCore
  // operationForDrag can be pulled into WebCore itself
  DCHECK(drag_data);

  if (!drag_data->ContainsURL()) {
    return DragOperation::kNone;
  }

  if (!did_initiate_drag_) {
    return DragOperation::kCopy;
  }

  // NEW LOGIC: Check if cross-document drop should be allowed
  if (document_under_mouse_ && ShouldAllowCrossDocumentDrop(drag_data, document_under_mouse_.Get())) {
    return DragOperation::kCopy;
  }

  return DragOperation::kNone;
}
```

#### 3. Update OperationForLoad Logic
**File**: [`third_party/blink/renderer/core/page/drag_controller.cc:525-527`](/third_party/blink/renderer/core/page/drag_controller.cc#L525)
**Changes**: Apply same cross-document validation

```cpp
DragOperation DragController::OperationForLoad(DragData* drag_data,
                                               LocalFrame& local_root) {
  DCHECK(drag_data);
  Document* doc = local_root.DocumentAtPoint(
      PhysicalOffset::FromPointFRound(drag_data->ClientPosition()));

  if (doc && (IsA<PluginDocument>(doc) || IsEditable(*doc))) {
    return DragOperation::kNone;
  }

  // NEW LOGIC: Apply cross-document validation for self-initiated drags
  if (doc && did_initiate_drag_ && !ShouldAllowCrossDocumentDrop(drag_data, doc)) {
    return DragOperation::kNone;
  }

  return GetDragOperation(drag_data);
}
```

### Success Criteria

#### Automated Verification
- [ ] Frame navigation compiles: `autoninja -C out/release_x64 blink_tests`
- [ ] Navigation tests run without crashing: `vpython3 third_party/blink/tools/run_web_tests.py` [`third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/`](/third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/) `-t release_x64`
- [ ] Core unit tests pass: `out/release_x64/content_unittests --gtest_filter='*Drag*'`

#### Manual Verification
- [ ] Can observe navigation detection in debug logs
- [ ] WPT navigation tests show improved behavior (may still fail but for different reasons)
- [ ] No crashes when navigating during drag operations

---

## Phase 3: Add Feature Flag and Testing

### Overview
Add a feature flag to control the new behavior and comprehensive testing to validate the fix.

### Changes Required

#### 1. Feature Flag Definition
**File**: [`third_party/blink/renderer/platform/runtime_enabled_features.json5`](/third_party/blink/renderer/platform/runtime_enabled_features.json5)
**Changes**: Add feature flag for the new behavior

```json5
{
  name: "DragDropNavigationFix",
  status: "experimental",
  description: "Allow cross-document drag and drop operations during navigation for same-origin file URLs"
}
```

#### 2. Conditional Logic in DragController
**File**: [`third_party/blink/renderer/core/page/drag_controller.cc`](/third_party/blink/renderer/core/page/drag_controller.cc)
**Changes**: Gate new behavior behind feature flag

```cpp
bool DragController::ShouldAllowCrossDocumentDrop(DragData* drag_data, Document* target_document) const {
  if (!RuntimeEnabledFeatures::DragDropNavigationFixEnabled()) {
    return false; // Use old behavior if flag is disabled
  }

  // ... existing new logic ...
}
```

#### 3. Add Unit Tests
**File**: [`third_party/blink/renderer/core/page/drag_controller_test.cc`](/third_party/blink/renderer/core/page/drag_controller_test.cc)
**Changes**: Add comprehensive tests for navigation behavior

```cpp
class DragControllerNavigationTest : public DragControllerTest {
 protected:
  void SetUp() override {
    DragControllerTest::SetUp();
    RuntimeEnabledFeatures::SetDragDropNavigationFixEnabled(true);
  }
};

TEST_F(DragControllerNavigationTest, AllowsSameOriginFileNavigation) {
  // Test that drag operations work across navigation for file URLs
  LoadHTML(R"HTML(
    <div draggable="true" id="dragme">Drag Me</div>
    <script>
      document.getElementById('dragme').addEventListener('dragstart', function(e) {
        e.dataTransfer.setData('text/plain', 'test');
        setTimeout(() => window.location = 'data:text/html,<div id="drop">Drop Here</div>', 0);
      });
    </script>
  )HTML");

  // Simulate drag start and navigation
  // ... test implementation ...
}

TEST_F(DragControllerNavigationTest, BlocksCrossOriginNavigation) {
  // Test that cross-origin navigation still blocks drops
  // ... test implementation ...
}

TEST_F(DragControllerNavigationTest, PreservesSecurityForWebContent) {
  // Test that web content security model is unchanged
  // ... test implementation ...
}
```

#### 4. Update WPT Test Expectations
**File**: [`third_party/blink/web_tests/TestExpectations`](/third_party/blink/web_tests/TestExpectations)
**Changes**: Update expectations for navigation tests

```
# Remove failing expectations for navigation tests once fix is implemented
# crbug.com/XXXXXX Navigation drag tests should pass with file URLs
# third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/001.xhtml [ Failure ]
```

### Success Criteria

#### Automated Verification
- [ ] Feature flag compiles: `autoninja -C out/release_x64 blink_tests`
- [ ] Unit tests pass with flag enabled: `out/release_x64/blink_unittests --gtest_filter='*DragControllerNavigation*'`
- [ ] Unit tests pass with flag disabled: `out/release_x64/blink_unittests --gtest_filter='*DragController*'`
- [ ] WPT navigation tests pass with flag enabled: `vpython3 third_party/blink/tools/run_web_tests.py` [`third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/`](/third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/) `-t release_x64 --additional-driver-flag=--enable-blink-features=DragDropNavigationFix`

#### Manual Verification
- [ ] WPT navigation tests pass when running locally with feature flag
- [ ] Drag and drop works correctly between local HTML files
- [ ] Security model still blocks malicious cross-origin scenarios
- [ ] No regressions in web-served content drag behavior

---

## Phase 4: Integration Testing and Documentation

### Overview
Comprehensive testing across different scenarios and documentation of the changes.

### Changes Required

#### 1. Integration Test Cases
**File**: [`third_party/blink/web_tests/fast/drag/navigation-during-drag.html`](/third_party/blink/web_tests/fast/drag/navigation-during-drag.html) (new)
**Changes**: Add comprehensive integration tests

```html
<!DOCTYPE html>
<html>
<head>
<title>Drag and Drop Navigation Integration Test</title>
<script src="../resources/testharness.js"></script>
<script src="../resources/testharnessreport.js"></script>
</head>
<body>
<div id="dragSource" draggable="true">Drag Source</div>
<div id="dropTarget">Drop Target</div>

<script>
async_test(function(t) {
  var dragSource = document.getElementById('dragSource');
  var dropTarget = document.getElementById('dropTarget');

  dragSource.addEventListener('dragstart', function(e) {
    e.dataTransfer.setData('text/plain', 'test data');
    // Simulate navigation during drag
    setTimeout(function() {
      history.pushState({}, '', '?navigated');
    }, 0);
  });

  dropTarget.addEventListener('dragover', function(e) {
    e.preventDefault();
  });

  dropTarget.addEventListener('drop', function(e) {
    e.preventDefault();
    t.done();
  });

  // Simulate drag and drop operation
  // ... test automation ...
}, 'Navigation during drag should allow drop');
</script>
</body>
</html>
```

#### 2. Performance Testing
**File**: [`third_party/blink/renderer/core/page/drag_controller_test.cc`](/third_party/blink/renderer/core/page/drag_controller_test.cc)
**Changes**: Add performance regression tests

```cpp
TEST_F(DragControllerNavigationTest, NavigationCheckPerformance) {
  // Ensure navigation checking doesn't significantly impact performance
  auto start_time = base::TimeTicks::Now();

  // Perform many navigation checks
  for (int i = 0; i < 1000; ++i) {
    // ... performance test implementation ...
  }

  auto duration = base::TimeTicks::Now() - start_time;
  EXPECT_LT(duration.InMilliseconds(), 100); // Should be fast
}
```

#### 3. Documentation Updates
**File**: [`third_party/blink/renderer/core/page/drag_controller.h`](/third_party/blink/renderer/core/page/drag_controller.h)
**Changes**: Document the navigation-aware behavior

```cpp
// Navigation-aware drag and drop controller that handles cross-document
// operations during navigation. When a drag operation is initiated and
// navigation occurs to a same-origin document (particularly file:// URLs),
// the controller allows drop operations that would normally be blocked
// for security reasons.
//
// Security model:
// - Same-origin file:// URLs: Drops allowed after navigation
// - Cross-origin scenarios: Drops blocked (unchanged security model)
// - Web content: Unchanged behavior (security model preserved)
class CORE_EXPORT DragController final : /* ... */
```

### Success Criteria

#### Automated Verification
- [ ] All integration tests pass: `vpython3 third_party/blink/tools/run_web_tests.py` [`third_party/blink/web_tests/fast/drag/`](/third_party/blink/web_tests/fast/drag/) `-t release_x64`
- [ ] Performance tests pass: `out/release_x64/blink_unittests --gtest_filter='*Performance*'`
- [ ] Full WPT suite passes: `vpython3 third_party/blink/tools/run_web_tests.py` [`third_party/blink/web_tests/external/wpt/html/editing/dnd/`](/third_party/blink/web_tests/external/wpt/html/editing/dnd/) `-t release_x64 --additional-driver-flag=--enable-blink-features=DragDropNavigationFix`
- [ ] No performance regressions in drag operations

#### Manual Verification
- [ ] Local HTML file navigation tests work correctly
- [ ] Cross-origin security still enforced properly
- [ ] Web-served content behavior unchanged
- [ ] Documentation is clear and accurate

---

## Testing Strategy

### Unit Tests
- Navigation detection logic in DragController
- Origin validation for file URLs
- Feature flag conditional behavior
- Performance impact of new checks
- Edge cases: null documents, malformed URLs, rapid navigation

### Integration Tests
- End-to-end WPT navigation test scenarios
- Local file drag and drop workflows
- Cross-origin security boundary enforcement
- Interaction with existing drag and drop features
- Browser navigation during drag operations

### Manual Testing Steps
1. **Basic Local File Navigation**:
   - Open [`third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/001.xhtml`](/third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/001.xhtml) as file://
   - Drag canvas element - should navigate and allow drop on new page

2. **Cross-Origin Security**:
   - Create test files with different origins
   - Verify drops are still blocked for security

3. **Web Content Regression**:
   - Test existing drag scenarios on web-served content
   - Verify no behavior changes

## Performance Considerations

- Navigation checks only occur during active drag operations (minimal impact)
- Origin comparisons are lightweight string operations
- Feature flag allows disabling if performance issues arise
- Existing hot paths remain unchanged when feature is disabled

## Migration Notes

- Feature flag provides controlled rollout capability
- Backward compatibility maintained when flag is disabled
- No changes to existing APIs or data structures
- Graceful degradation if navigation detection fails

## References

- Original research: [`drag_drop_local_navigation_research.md`](/research.md)
- WPT navigation tests: [`third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/`](/third_party/blink/web_tests/external/wpt/html/editing/dnd/navigation/)
- DragController implementation: [`third_party/blink/renderer/core/page/drag_controller.cc`](/third_party/blink/renderer/core/page/drag_controller.cc)
- Security origin handling: [`third_party/blink/renderer/platform/weborigin/security_origin.h`](/third_party/blink/renderer/platform/weborigin/security_origin.h)
