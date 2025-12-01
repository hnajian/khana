# 9. Compact Margin When Header/Footer Dismissed

**Added**: v0.9.39 (Commit 4c1af671, #1047, #734)

## Overview

When header and/or footer widgets are dismissed, the reading area automatically adjusts with compact margins and gap values to maximize content space. This ensures that hiding UI elements actually provides more reading space rather than leaving empty gaps.

## Implementation

**File**: `src/app/reader/components/FoliateViewer.tsx`

**Margin Calculation**:
```typescript
const getEffectiveMargins = (viewSettings: ViewSettings) => {
  const baseMargin = viewSettings.margin || 20;
  const baseGap = viewSettings.gap || 10;

  // Compact mode when widgets are hidden
  const compactMargin = Math.max(4, Math.round(baseMargin * 0.3));
  const compactGap = Math.max(2, Math.round(baseGap * 0.3));

  return {
    top: viewSettings.showHeader ? baseMargin : compactMargin,
    bottom: viewSettings.showFooter ? baseMargin : compactMargin,
    left: baseMargin,
    right: baseMargin,
    gap: (viewSettings.showHeader || viewSettings.showFooter) ? baseGap : compactGap
  };
};
```

## Visual Impact

**With Header/Footer** (Normal Mode):
```
┌────────────────────────────────┐
│ Header Bar (44px)              │
├────────────────────────────────┤
│ ↕ Top Margin (20px)            │
│                                │
│    Book Content                │
│                                │
│ ↕ Bottom Margin (20px)         │
├────────────────────────────────┤
│ Footer Bar (44px)              │
└────────────────────────────────┘
```

**Without Header/Footer** (Compact Mode):
```
┌────────────────────────────────┐
│ ↕ Top Margin (6px) - Compact   │
│                                │
│                                │
│    Book Content (More Space)   │
│                                │
│                                │
│ ↕ Bottom Margin (6px) - Compact│
└────────────────────────────────┘
```

## Margin Reduction Formula

**Compact Margin**: `max(4px, margin × 0.3)`

**Examples**:
- `margin: 20px` → compact: `6px`
- `margin: 40px` → compact: `12px`
- `margin: 10px` → compact: `4px` (minimum)

**Compact Gap**: `max(2px, gap × 0.3)`

**Examples**:
- `gap: 10px` → compact: `4px`
- `gap: 20px` → compact: `6px`
- `gap: 5px` → compact: `2px` (minimum)

## Application

**CSS Variables**:
```typescript
useEffect(() => {
  const margins = getEffectiveMargins(viewSettings);

  bookView?.renderer.setStyles?.({
    '--margin-top': `${margins.top}px`,
    '--margin-bottom': `${margins.bottom}px`,
    '--margin-left': `${margins.left}px`,
    '--margin-right': `${margins.right}px`,
    '--gap': `${margins.gap}px`
  });
}, [viewSettings.showHeader, viewSettings.showFooter, viewSettings.margin, viewSettings.gap]);
```

## Settings Interaction

**Affected Settings**:
- `showHeader`: Triggers compact top margin when false
- `showFooter`: Triggers compact bottom margin when false
- `margin`: Base value for compact calculation
- `gap`: Base value for compact gap calculation

**Independent Controls**:
- Left/right margins remain at base value
- Only top/bottom margins are compacted
- Gap between columns is compacted when widgets hidden

## Use Cases

**Maximized Reading Space**:
- Small screens (mobile devices)
- Distraction-free reading
- Users who want minimal UI

**Precision Layout Control**:
- Fine-tune exact content area
- Optimize for specific screen sizes
- Balance aesthetics and functionality

---
