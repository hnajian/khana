# 1. Continuous Scroll Option

**Added**: v0.9.20 (Commit c25f2a7a, #522, #516, #373)

## Overview

The Continuous Scroll option enables smooth, browser-like scrolling in scrolled mode, as opposed to discrete page-by-page scrolling. This provides a more natural reading experience for users who prefer fluid scrolling.

## Settings Location

**Path**: Settings Dialog → Behavior (Control) Panel → Scroll section → Continuous Scroll toggle

**File**: `src/components/settings/ControlPanel.tsx` (lines 168-176)

## Implementation

**Type Definition** (`src/types/book.ts`):
```typescript
export interface ViewSettings {
  scrolled: boolean;           // Enable scrolled mode
  continuousScroll: boolean;   // Enable continuous scroll (default: false)
  // ... other settings
}
```

**Default Value** (`src/services/constants.ts`):
```typescript
export const DEFAULT_VIEW_SETTINGS: ViewSettings = {
  scrolled: false,
  continuousScroll: false,  // Disabled by default
  // ...
};
```

## Behavior

**When Enabled** (`continuousScroll: true`):
- Mouse wheel/trackpad scrolling is smooth and continuous
- Scrolling moves pixel-by-pixel like a web browser
- No discrete page jumps
- Scroll velocity is preserved (momentum scrolling)

**When Disabled** (`continuousScroll: false`):
- Scrolling is discrete and paginated
- Each scroll event advances by one page
- Clear page boundaries
- More controlled navigation

## Technical Details

**Scroll Handling** (`src/app/reader/hooks/useIframeEvents.ts`):
```typescript
const handleWheel = (event: WheelEvent) => {
  if (!viewSettings.continuousScroll) {
    event.preventDefault();

    // Discrete page-by-page scrolling
    if (event.deltaY > 0) {
      goToNextPage();
    } else if (event.deltaY < 0) {
      goToPreviousPage();
    }
  }
  // If continuousScroll is true, allow default browser scrolling
};
```

**Debounced Handling**:
- Scroll events are debounced with 500ms delay
- Prevents excessive page flipping on rapid scrolling
- Threshold: 30px for touch events

## Use Cases

**Continuous Scroll Enabled**:
- Long-form reading (novels, articles)
- Users who prefer smooth scrolling
- Touchpad/trackpad users
- Reading on web browsers

**Continuous Scroll Disabled**:
- Precise page navigation
- Users who prefer discrete page turns
- Mimicking physical book experience
- Better control over reading position

## Related Settings

- **Scrolled Mode** (`viewSettings.scrolled`): Must be enabled for continuous scroll to have effect
- **Paginated Mode**: Continuous scroll has no effect when paginated mode is active

---
