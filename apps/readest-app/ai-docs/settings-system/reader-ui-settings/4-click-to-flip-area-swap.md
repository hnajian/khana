# 4. Click-to-Flip Area Swap

**Added**: v0.9.26 (Commit 3e388960, #727, #719)

## Overview

The Click-to-Flip Area Swap option allows users to reverse the default click/tap navigation zones, making the left side advance forward and the right side go backward. This is particularly useful for left-handed users or those who prefer non-standard navigation.

## Settings Location

**Path**: Settings Dialog → Behavior (Control) Panel → Pagination section → "Swap Click Sides" (or "Swap Tap Sides" on mobile)

**File**: `src/components/settings/ControlPanel.tsx` (lines 217-228)

## Default Behavior

**For LTR (Left-to-Right) Books**:
- **Left Click**: Previous page
- **Right Click**: Next page

**For RTL (Right-to-Left) Books**:
- **Left Click**: Next page (reversed)
- **Right Click**: Previous page (reversed)

## Swapped Behavior

**For LTR Books** (when `swapClickArea: true`):
- **Left Click**: Next page
- **Right Click**: Previous page

**For RTL Books** (when `swapClickArea: true`):
- **Left Click**: Previous page (reversed)
- **Right Click**: Next page (reversed)

## Implementation

**Type Definition** (`src/types/book.ts`):
```typescript
export interface ViewSettings {
  swapClickArea: boolean;  // Swap click/tap sides (default: false)
  // ... other settings
}
```

**Click Area Detection** (`src/app/reader/hooks/useIframeEvents.ts`):
```typescript
const handleClick = (event: MouseEvent) => {
  const clickX = event.clientX;
  const screenWidth = window.innerWidth;
  const clickZone = clickX / screenWidth;

  // Determine if click is on left or right side
  const isLeftSide = clickZone < 0.5;

  // Apply swap logic
  const shouldGoForward = viewSettings.swapClickArea
    ? isLeftSide   // When swapped, left = next
    : !isLeftSide; // When normal, right = next

  // Apply RTL reversal
  if (bookLayout.rtl) {
    shouldGoForward = !shouldGoForward;
  }

  if (shouldGoForward) {
    goToNextPage();
  } else {
    goToPreviousPage();
  }
};
```

## Disabled Conditions

The swap setting is **disabled** when:

1. **Click-to-Flip is OFF** (`disableClick: true`):
   ```typescript
   if (viewSettings.disableClick) {
     // Swap option is grayed out
   }
   ```

2. **Both Sides Advance Forward** (`fullscreenClickArea: true`):
   ```typescript
   if (viewSettings.fullscreenClickArea) {
     // Both sides go to next page, swap has no effect
   }
   ```

## UI Representation

```typescript
<div className="setting-row">
  <label htmlFor="swapClickArea">
    {isMobile ? _('Swap Tap Sides') : _('Swap Click Sides')}
  </label>
  <input
    type="checkbox"
    id="swapClickArea"
    checked={viewSettings.swapClickArea}
    disabled={viewSettings.disableClick || viewSettings.fullscreenClickArea}
    onChange={(e) => updateSetting('swapClickArea', e.target.checked)}
  />
  <p className="setting-description">
    {_('Reverse the tap/click areas for page navigation')}
  </p>
</div>
```

## Related Settings

| Setting | Description | Interaction with Swap |
|---------|-------------|----------------------|
| **Tap/Click to Paginate** | Enable/disable click navigation | Must be enabled for swap to work |
| **Tap/Click Both Sides** | Both sides advance forward | Overrides swap setting |
| **Disable Double Tap/Click** | Prevent zoom on double-tap | Independent, works with swap |

## Use Cases

**Left-Handed Users**:
- Hold device in left hand
- Use left thumb to advance pages
- More ergonomic navigation

**Tablet/E-Reader Holders**:
- Holding device with left hand
- Right hand is occupied
- Left-thumb navigation preferred

**Personal Preference**:
- Users who find reversed navigation more intuitive
- Coming from apps with reversed navigation

---
