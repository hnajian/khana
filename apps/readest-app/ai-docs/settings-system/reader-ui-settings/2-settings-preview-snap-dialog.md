# 2. Settings Preview Snap Dialog

**Added**: v0.9.25 (Commit accf6fb5, #646)

## Overview

The Settings Preview Snap Dialog is a mobile-optimized bottom sheet that allows users to preview font, layout, and color changes in real-time while keeping the book content partially visible. The dialog can be dragged to different heights and "snaps" to predefined positions.

## Implementation

**File**: `src/components/Dialog.tsx`

## Snap Behavior

The dialog supports three states:

1. **Full Height** (100%): Dialog fills entire screen
2. **Snapped Height** (70%): Dialog occupies 70% of screen, allowing book preview below
3. **Dismissed** (0%): Dialog is fully closed

## Drag Mechanics

**Snap Thresholds**:
```typescript
const SNAP_THRESHOLD = 0.2;        // 20% tolerance zone
const VELOCITY_THRESHOLD = 0.5;    // Swipe velocity for dismissal
const SNAP_HEIGHT = 0.7;           // 70% of screen height
```

**Drag Zones**:
- **Upper Zone**: `position > 1 - snapHeight - 0.2` → Snaps to full screen
- **Snap Zone**: `position ≈ 1 - snapHeight ± 0.2` → Snaps to 70%
- **Lower Zone**: `position < 1 - snapHeight + 0.2` → Dismisses dialog

**Velocity-Based Dismissal**:
- If user swipes down with velocity > 0.5, dialog dismisses regardless of position
- Provides quick gesture-based closing

## Touch Event Handling

```typescript
const handleTouchStart = (e: TouchEvent) => {
  const touch = e.touches[0];
  setStartY(touch.clientY);
  setStartTime(Date.now());
};

const handleTouchMove = (e: TouchEvent) => {
  const touch = e.touches[0];
  const currentY = touch.clientY;
  const deltaY = currentY - startY;

  // Update dialog position
  const newPosition = Math.max(0, Math.min(1, deltaY / screenHeight));
  setDialogPosition(newPosition);
};

const handleTouchEnd = () => {
  const velocity = (currentY - startY) / (Date.now() - startTime);

  if (velocity > VELOCITY_THRESHOLD) {
    // Fast swipe down - dismiss
    dismissDialog();
  } else if (Math.abs(dialogPosition - SNAP_HEIGHT) < SNAP_THRESHOLD) {
    // Near snap zone - snap to 70%
    snapToHeight(SNAP_HEIGHT);
  } else if (dialogPosition > 0.5) {
    // Upper half - snap to full
    snapToHeight(1.0);
  } else {
    // Lower half - dismiss
    dismissDialog();
  }
};
```

## Haptic Feedback

**On Snap**:
```typescript
if (isSnapping) {
  performHapticFeedback('light');
}
```

**On Dismiss**:
```typescript
if (isDismissing) {
  performHapticFeedback('medium');
}
```

## Real-Time Preview

**How It Works**:
1. User opens Settings Dialog on mobile
2. User drags dialog to 70% snap position
3. Book content is visible in bottom 30% of screen
4. User adjusts font size slider
5. Font changes are immediately reflected in visible book content
6. User can compare settings without fully closing dialog

## Platform Availability

- **Mobile (width < 640px)**: Full snap dialog functionality
- **Desktop**: Standard full-screen modal (no snapping)

## CSS

```css
.snap-dialog {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  transform: translateY(var(--dialog-position));
  transition: transform 0.3s cubic-bezier(0.4, 0.0, 0.2, 1);
}

.snap-dialog.dragging {
  transition: none;  /* Disable transition during drag */
}

.snap-dialog-handle {
  width: 36px;
  height: 4px;
  background: rgba(128, 128, 128, 0.5);
  border-radius: 2px;
  margin: 8px auto;
  cursor: grab;
}

.snap-dialog-handle:active {
  cursor: grabbing;
}
```

---
