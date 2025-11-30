# Reader UI and Interaction Settings

**Category**: User Interface, User Experience, Settings
**Added**: v0.9.19-0.9.31
**Related Commits**: #522, #580, #596, #620, #646, #719, #727, #750

## Overview

This document covers advanced reader UI and interaction settings added in versions 0.9.19-0.9.31, providing users with fine-grained control over the reading interface, navigation behavior, and settings preview experience.

## 1. Continuous Scroll Option

**Added**: v0.9.20 (Commit c25f2a7a, #522, #516, #373)

### Overview

The Continuous Scroll option enables smooth, browser-like scrolling in scrolled mode, as opposed to discrete page-by-page scrolling. This provides a more natural reading experience for users who prefer fluid scrolling.

### Settings Location

**Path**: Settings Dialog → Behavior (Control) Panel → Scroll section → Continuous Scroll toggle

**File**: `src/components/settings/ControlPanel.tsx` (lines 168-176)

### Implementation

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

### Behavior

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

### Technical Details

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

### Use Cases

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

### Related Settings

- **Scrolled Mode** (`viewSettings.scrolled`): Must be enabled for continuous scroll to have effect
- **Paginated Mode**: Continuous scroll has no effect when paginated mode is active

---

## 2. Settings Preview Snap Dialog

**Added**: v0.9.25 (Commit accf6fb5, #646)

### Overview

The Settings Preview Snap Dialog is a mobile-optimized bottom sheet that allows users to preview font, layout, and color changes in real-time while keeping the book content partially visible. The dialog can be dragged to different heights and "snaps" to predefined positions.

### Implementation

**File**: `src/components/Dialog.tsx`

### Snap Behavior

The dialog supports three states:

1. **Full Height** (100%): Dialog fills entire screen
2. **Snapped Height** (70%): Dialog occupies 70% of screen, allowing book preview below
3. **Dismissed** (0%): Dialog is fully closed

### Drag Mechanics

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

### Touch Event Handling

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

### Haptic Feedback

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

### Real-Time Preview

**How It Works**:
1. User opens Settings Dialog on mobile
2. User drags dialog to 70% snap position
3. Book content is visible in bottom 30% of screen
4. User adjusts font size slider
5. Font changes are immediately reflected in visible book content
6. User can compare settings without fully closing dialog

### Platform Availability

- **Mobile (width < 640px)**: Full snap dialog functionality
- **Desktop**: Standard full-screen modal (no snapping)

### CSS

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

## 3. Global Fulltext Search Shortcut

**Added**: v0.9.27 (Commit 01ad18ca, #750)

### Overview

The Global Fulltext Search Shortcut provides a standard `Ctrl+F` (Windows/Linux) or `Cmd+F` (macOS) keyboard shortcut to open the search interface, making Readest more intuitive for users familiar with browser search functionality.

### Keyboard Shortcut

- **Windows/Linux**: `Ctrl + F`
- **macOS**: `Cmd + F`

### Implementation

**File**: `src/app/reader/hooks/useBookShortcuts.ts`

```typescript
import { useShortcuts } from '@/hooks/useShortcuts';

export const useBookShortcuts = () => {
  const shortcuts = useShortcuts();

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      // Ctrl+F or Cmd+F
      if ((e.ctrlKey || e.metaKey) && e.key === 'f') {
        e.preventDefault();
        shortcuts.onToggleSearchBar();
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, []);
};
```

**Shortcut Handler** (`src/hooks/useShortcuts.ts`):
```typescript
const showSearchBar = () => {
  eventDispatcher.dispatch('search', { term: '' });
};

export const useShortcuts = () => ({
  onToggleSearchBar: showSearchBar,
  // ... other shortcuts
});
```

### Event Flow

1. User presses `Ctrl+F` / `Cmd+F`
2. Event is captured by `useBookShortcuts`
3. Browser's default find-in-page is prevented
4. `showSearchBar()` is called
5. 'search' event is dispatched
6. Search sidebar tab is activated
7. Search input receives focus

### Context-Aware Behavior

**When Focus Is On**:
- **Book content**: Opens search sidebar
- **Input/textarea**: Allows browser default (except note editor)
- **Note editor**: Opens search sidebar (special case)

```typescript
const isInputFocused = () => {
  const activeElement = document.activeElement;
  const tagName = activeElement?.tagName.toLowerCase();

  if (tagName === 'input' || tagName === 'textarea') {
    // Exception: note editor
    if (activeElement?.classList.contains('note-editor')) {
      return false;
    }
    return true;
  }
  return false;
};

const handleKeyDown = (e: KeyboardEvent) => {
  if (isInputFocused()) return;  // Don't interfere with input fields

  if ((e.ctrlKey || e.metaKey) && e.key === 'f') {
    e.preventDefault();
    showSearchBar();
  }
};
```

### Cross-Platform Support

**Modifier Key Detection**:
```typescript
const isMac = navigator.platform.toUpperCase().indexOf('MAC') >= 0;
const modifierKey = isMac ? 'Cmd' : 'Ctrl';

// Display in UI
<kbd>{modifierKey}+F</kbd>
```

### Accessibility

- **Keyboard-only users**: Can quickly access search without mouse
- **Screen readers**: Announces "Search activated" when triggered
- **Focus management**: Search input receives focus immediately

### Related Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl/Cmd + F` | Open search |
| `Ctrl/Cmd + G` | Find next (browser default) |
| `Ctrl/Cmd + Shift + G` | Find previous (browser default) |
| `Esc` | Close search sidebar |

---

## 4. Click-to-Flip Area Swap

**Added**: v0.9.26 (Commit 3e388960, #727, #719)

### Overview

The Click-to-Flip Area Swap option allows users to reverse the default click/tap navigation zones, making the left side advance forward and the right side go backward. This is particularly useful for left-handed users or those who prefer non-standard navigation.

### Settings Location

**Path**: Settings Dialog → Behavior (Control) Panel → Pagination section → "Swap Click Sides" (or "Swap Tap Sides" on mobile)

**File**: `src/components/settings/ControlPanel.tsx` (lines 217-228)

### Default Behavior

**For LTR (Left-to-Right) Books**:
- **Left Click**: Previous page
- **Right Click**: Next page

**For RTL (Right-to-Left) Books**:
- **Left Click**: Next page (reversed)
- **Right Click**: Previous page (reversed)

### Swapped Behavior

**For LTR Books** (when `swapClickArea: true`):
- **Left Click**: Next page
- **Right Click**: Previous page

**For RTL Books** (when `swapClickArea: true`):
- **Left Click**: Previous page (reversed)
- **Right Click**: Next page (reversed)

### Implementation

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

### Disabled Conditions

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

### UI Representation

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

### Related Settings

| Setting | Description | Interaction with Swap |
|---------|-------------|----------------------|
| **Tap/Click to Paginate** | Enable/disable click navigation | Must be enabled for swap to work |
| **Tap/Click Both Sides** | Both sides advance forward | Overrides swap setting |
| **Disable Double Tap/Click** | Prevent zoom on double-tap | Independent, works with swap |

### Use Cases

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

## 5. Show/Hide Header and Footer Widgets

**Added**: v0.9.23 (Commit 48074f0f, #620, #602)

### Overview

Provides granular control over reader interface elements, allowing users to show or hide header and footer widgets, customize progress indicators, and choose between distraction-free or information-rich reading experiences.

### Settings Location

**Path**: Settings Dialog → Layout Panel → Header & Footer section

**File**: `src/components/settings/LayoutPanel.tsx` (lines 603-692)

### Available Settings

#### 1. Show Header (Toggle)

**Default**: Enabled in paginated mode, disabled in scrolled mode

```typescript
<input
  type="checkbox"
  checked={viewSettings.showHeader}
  onChange={(e) => updateSetting('showHeader', e.target.checked)}
/>
```

**When Enabled**:
- Section/chapter title displayed at top
- Fixed position header bar
- Minimum top margin enforced

**When Disabled**:
- No header bar
- More vertical space for content
- Cleaner reading interface

#### 2. Show Footer (Toggle)

**Default**: Enabled in paginated mode, disabled in scrolled mode

```typescript
<input
  type="checkbox"
  checked={viewSettings.showFooter}
  onChange={(e) => updateSetting('showFooter', e.target.checked)}
/>
```

**When Enabled**:
- Progress indicators at bottom
- Page numbers or percentage
- Remaining time/pages (if enabled)
- Minimum bottom margin enforced

**When Disabled**:
- No footer bar
- Maximum vertical space
- Immersive reading experience

#### 3. Show Remaining Time (Toggle)

**Default**: Enabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showRemainingTime}
  disabled={!viewSettings.showFooter}
  onChange={(e) => updateSetting('showRemainingTime', e.target.checked)}
/>
```

**Calculation** (`src/app/reader/components/FooterBar.tsx`):
```typescript
const calculateRemainingTime = (
  remainingPages: number,
  averageReadingSpeed: number  // pages per minute
): string => {
  const minutes = Math.round(remainingPages / averageReadingSpeed);

  if (minutes < 60) {
    return `${minutes} min left`;
  } else {
    const hours = Math.floor(minutes / 60);
    const mins = minutes % 60;
    return `${hours}h ${mins}m left`;
  }
};
```

**Display Format**:
- **Less than 1 hour**: "42 min left"
- **More than 1 hour**: "2h 15m left"
- **Scope**: Current chapter or entire book (configurable)

#### 4. Show Remaining Pages (Toggle)

**Default**: Enabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showRemainingPages}
  disabled={!viewSettings.showFooter}
  onChange={(e) => updateSetting('showRemainingPages', e.target.checked)}
/>
```

**Display Format**:
- "123 pages left in chapter"
- "456 pages left in book"

#### 5. Show Reading Progress (Toggle)

**Default**: Enabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showProgress}
  disabled={!viewSettings.showFooter}
  onChange={(e) => updateSetting('showProgress', e.target.checked)}
/>
```

**Components**:
- Progress bar (visual indicator)
- Numeric progress (page number or percentage)

#### 6. Reading Progress Style (Radio/Select)

**Default**: "Page Number"

**Options**:
- **Page Number**: "12 / 345" (current page / total pages)
- **Percentage**: "3.5%" (percentage through book/chapter)

```typescript
<select
  value={viewSettings.progressStyle}
  disabled={!viewSettings.showProgress}
  onChange={(e) => updateSetting('progressStyle', e.target.value)}
>
  <option value="pageNumber">{_('Page Number')}</option>
  <option value="percentage">{_('Percentage')}</option>
</select>
```

**Implementation**:
```typescript
const getProgressDisplay = () => {
  if (viewSettings.progressStyle === 'percentage') {
    const percent = ((currentPage / totalPages) * 100).toFixed(1);
    return `${percent}%`;
  } else {
    return `${currentPage} / ${totalPages}`;
  }
};
```

#### 7. Apply Also in Scrolled Mode (Toggle)

**Default**: Disabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showWidgetsInScrolledMode}
  onChange={(e) => updateSetting('showWidgetsInScrolledMode', e.target.checked)}
/>
```

**Purpose**: By default, header/footer are hidden in scrolled mode for distraction-free reading. This option forces them to display even in scrolled mode.

### Margin Behavior

**Header Margin Enforcement**:
```typescript
const getMinTopMargin = (): number => {
  if (!viewSettings.showHeader) return 0;

  const headerHeight = 44; // px
  const safeAreaTop = gridInsets.top;
  const requiredMargin = headerHeight - safeAreaTop;

  return Math.max(0, Math.round(requiredMargin / 4) * 4); // Round to nearest 4px
};
```

**Footer Margin Enforcement**:
```typescript
const getMinBottomMargin = (): number => {
  if (!viewSettings.showFooter) return 0;

  const footerHeight = 44; // px
  const safeAreaBottom = gridInsets.bottom;
  const requiredMargin = footerHeight - safeAreaBottom;

  return Math.max(0, Math.round(requiredMargin / 4) * 4);
};
```

### UI Components

**Header Component** (`src/app/reader/components/SectionInfo.tsx`):
```typescript
const SectionInfo = () => {
  const { viewSettings } = useReaderStore();

  if (!viewSettings.showHeader) return null;
  if (viewSettings.scrolled && !viewSettings.showWidgetsInScrolledMode) {
    return null;
  }

  return (
    <div className="section-info">
      {currentSection.title}
    </div>
  );
};
```

**Footer Component** (`src/app/reader/components/PageInfo.tsx`):
```typescript
const PageInfo = () => {
  const { viewSettings } = useReaderStore();

  if (!viewSettings.showFooter) return null;
  if (viewSettings.scrolled && !viewSettings.showWidgetsInScrolledMode) {
    return null;
  }

  return (
    <div className="page-info">
      {viewSettings.showProgress && <Progress />}
      {viewSettings.showRemainingTime && <RemainingTime />}
      {viewSettings.showRemainingPages && <RemainingPages />}
    </div>
  );
};
```

### Preset Configurations

**Minimal (Distraction-Free)**:
```typescript
{
  showHeader: false,
  showFooter: false,
  showWidgetsInScrolledMode: false
}
```

**Information-Rich**:
```typescript
{
  showHeader: true,
  showFooter: true,
  showRemainingTime: true,
  showRemainingPages: true,
  showProgress: true,
  progressStyle: 'pageNumber',
  showWidgetsInScrolledMode: true
}
```

**Balanced (Default)**:
```typescript
{
  showHeader: true,
  showFooter: true,
  showRemainingTime: true,
  showRemainingPages: false,
  showProgress: true,
  progressStyle: 'pageNumber',
  showWidgetsInScrolledMode: false
}
```

---

## 6. Scrolled Mode Toggler in Layout Panel

**Added**: v0.9.21 (Commit b7dc880a, #596, #580)

### Overview

The Scrolled Mode Toggler was added to the Layout Panel to streamline the workflow for switching between paginated and scrolled modes while adjusting layout settings. Previously, users had to switch between Behavior and Layout panels to toggle modes and adjust settings.

### Settings Location

**Path**: Settings Dialog → Layout Panel → Top of panel (first option)

**File**: `src/components/settings/LayoutPanel.tsx`

### Implementation

```typescript
const LayoutPanel = () => {
  const { viewSettings, updateViewSettings } = useReaderStore();
  const [isScrolledMode, setScrolledMode] = useState(viewSettings.scrolled);

  const handleScrolledToggle = (enabled: boolean) => {
    setScrolledMode(enabled);
    updateViewSettings({ scrolled: enabled });

    // Update renderer
    const view = getView(bookKey);
    view?.renderer.setAttribute('flow', enabled ? 'scrolled' : 'paginated');

    // Update max-inline-size for scrolled mode
    const maxInlineSize = getMaxInlineSize(viewSettings);
    view?.renderer.setAttribute('max-inline-size', `${maxInlineSize}px`);

    // Reapply styles
    view?.renderer.setStyles?.(getStyles(viewSettings));
  };

  return (
    <div className="layout-panel">
      {/* Scrolled Mode Toggle - First Option */}
      <div className="setting-row">
        <label htmlFor="scrolledMode">{_('Scrolled Mode')}</label>
        <input
          type="checkbox"
          id="scrolledMode"
          checked={isScrolledMode}
          onChange={(e) => handleScrolledToggle(e.target.checked)}
        />
        <p className="setting-description">
          {_('Enable continuous scrolling instead of page-by-page navigation')}
        </p>
      </div>

      {/* Rest of layout settings... */}
      {/* Margins, Columns, etc. */}
    </div>
  );
};
```

### Benefits

**Before** (v0.9.20 and earlier):
1. Open Settings Dialog
2. Go to Behavior (Control) Panel
3. Toggle Scrolled Mode
4. Switch to Layout Panel
5. Adjust margins/columns
6. Switch back to Behavior Panel to toggle mode again
7. Switch to Layout Panel to see changes

**After** (v0.9.21+):
1. Open Settings Dialog
2. Go to Layout Panel
3. Toggle Scrolled Mode (in same panel)
4. Adjust margins/columns immediately
5. Compare paginated vs scrolled settings without panel switching

### Mode-Specific Layout Behavior

**Paginated Mode**:
- `maxColumnCount`: Controls number of columns per page
- `maxInlineSize`: Maximum width per column
- Margins affect page boundaries

**Scrolled Mode**:
- `maxColumnCount`: Usually 1 (single column scrolling)
- `maxInlineSize`: Maximum content width
- Margins create padding around scrollable content

### Real-Time Updates

```typescript
useEffect(() => {
  if (!bookView) return;

  // Apply flow attribute
  bookView.renderer.setAttribute('flow', isScrolledMode ? 'scrolled' : 'paginated');

  // Update column configuration
  if (isScrolledMode) {
    bookView.renderer.setAttribute('max-column-count', '1');
  } else {
    bookView.renderer.setAttribute('max-column-count', String(viewSettings.maxColumnCount || 2));
  }

  // Update max-inline-size
  const maxInlineSize = isScrolledMode
    ? viewSettings.maxInlineSize || 720
    : (viewSettings.maxInlineSize || 720) / (viewSettings.maxColumnCount || 1);

  bookView.renderer.setAttribute('max-inline-size', `${maxInlineSize}px`);

  // Reflow layout
  bookView.renderer.render();
}, [isScrolledMode, viewSettings]);
```

### Visual Feedback

**Toggle State Indicator**:
```css
.scrolled-mode-toggle {
  position: relative;
}

.scrolled-mode-toggle::after {
  content: attr(data-mode);
  position: absolute;
  right: 0;
  font-size: 12px;
  color: var(--text-muted);
}

.scrolled-mode-toggle[data-mode="paginated"]::after {
  content: "📖 Paginated";
}

.scrolled-mode-toggle[data-mode="scrolled"]::after {
  content: "📜 Scrolled";
}
```

### Interaction with Other Settings

**Affected Settings**:
- **Column Width/Height Labels**: Swap based on mode
  - Paginated: "Maximum Column Width"
  - Scrolled: "Maximum Content Width"

- **Header/Footer Widgets**: Different defaults
  - Paginated: Shown by default
  - Scrolled: Hidden by default (unless "Apply also in scrolled mode" is enabled)

- **Continuous Scroll**: Only available in scrolled mode

**Setting Label Updates**:
```typescript
const getColumnWidthLabel = () => {
  return isScrolledMode
    ? _('Maximum Content Width')
    : bookLayout.vertical
      ? _('Maximum Column Height')
      : _('Maximum Column Width');
};
```

### Keyboard Shortcut (Optional Enhancement)

```typescript
// Not implemented by default, but recommended
const SHORTCUT_TOGGLE_SCROLLED = 'Ctrl+Shift+S';

useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    if (e.ctrlKey && e.shiftKey && e.key === 's') {
      e.preventDefault();
      handleScrolledToggle(!isScrolledMode);
    }
  };

  document.addEventListener('keydown', handleKeyDown);
  return () => document.removeEventListener('keydown', handleKeyDown);
}, [isScrolledMode]);
```

---

## Summary

These 6 features collectively enhance the reader UI and user experience:

| Feature | User Benefit | Technical Impact |
|---------|--------------|------------------|
| **Continuous Scroll** | Natural browser-like scrolling | Smooth scroll vs discrete pagination |
| **Snap Dialog** | Real-time settings preview on mobile | Better UX for font/layout adjustments |
| **Search Shortcut** | Familiar Ctrl/Cmd+F workflow | Standard keyboard navigation |
| **Click Area Swap** | Left-handed navigation, accessibility | Reversed click zones |
| **Header/Footer Control** | Customizable reading interface | Granular widget visibility |
| **Scrolled Toggler** | Streamlined mode switching | Faster layout optimization |

All features follow Readest's design philosophy of providing power users with extensive customization while maintaining sensible defaults for casual users.

---

---

## 7. Prev/Next Section Navigation

**Added**: v0.9.43 (Commit 7c464b9a, #1195, #1125)

### Overview

The Prev/Next Section Navigation feature adds dedicated buttons in the footer bar for quick navigation between book sections/chapters. This complements existing page navigation by allowing users to jump directly to the previous or next section without using the table of contents.

### UI Location

**Position**: Footer bar (reader interface)
**Buttons**:
- Previous Section (⏮ icon)
- Next Section (⏭ icon)

### Implementation

**File**: `src/app/reader/components/FooterBar.tsx`

```typescript
const FooterBar = () => {
  const { sections, currentSectionIndex } = useBookNavigation();
  const hasPrevSection = currentSectionIndex > 0;
  const hasNextSection = currentSectionIndex < sections.length - 1;

  const goToPrevSection = () => {
    if (hasPrevSection) {
      const prevSection = sections[currentSectionIndex - 1];
      navigateToSection(prevSection.href);
    }
  };

  const goToNextSection = () => {
    if (hasNextSection) {
      const nextSection = sections[currentSectionIndex + 1];
      navigateToSection(nextSection.href);
    }
  };

  return (
    <div className="footer-bar">
      <button
        onClick={goToPrevSection}
        disabled={!hasPrevSection}
        aria-label="Previous section"
      >
        <FaStepBackward />
      </button>

      {/* Page info and progress */}

      <button
        onClick={goToNextSection}
        disabled={!hasNextSection}
        aria-label="Next section"
      >
        <FaStepForward />
      </button>
    </div>
  );
};
```

### Button Behavior

**Previous Section Button**:
- Navigates to the start of the previous chapter/section
- Disabled when on first section
- Gray/dimmed appearance when disabled

**Next Section Button**:
- Navigates to the start of the next chapter/section
- Disabled when on last section
- Gray/dimmed appearance when disabled

### Section Detection

**TOC-Based** (`src/app/reader/hooks/useBookNavigation.ts`):
```typescript
const getCurrentSectionIndex = (currentCfi: string): number => {
  const toc = book.getTOC();

  // Find current section based on CFI
  for (let i = 0; i < toc.length; i++) {
    const section = toc[i];
    if (isCfiBefore(currentCfi, section.cfi)) {
      return Math.max(0, i - 1);
    }
  }

  return toc.length - 1;
};
```

### Keyboard Shortcuts

While not part of the initial implementation, suggested shortcuts:
- `Alt + ←`: Previous section
- `Alt + →`: Next section

### Use Cases

**Quick Chapter Navigation**:
- Finish reading a chapter and immediately jump to next
- Return to previous chapter for reference
- Skip to specific parts without opening TOC

**Linear Reading Flow**:
- Continue reading without interruption
- Natural progression through book structure
- Maintain reading momentum

### Visual Design

**Button Styling**:
```css
.section-nav-button {
  padding: 8px 12px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.section-nav-button:hover:not(:disabled) {
  background: var(--hover-bg);
  border-color: var(--hover-border);
}

.section-nav-button:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
```

---

## 8. Separate Header/Footer Visibility for Reading Modes

**Added**: v0.9.32 (Commit ccd467eb, #859)

### Overview

This feature allows independent control of header and footer visibility for paginated and scrolled modes, enabling users to have different UI configurations for each reading mode.

### Settings Structure

**Type Definition** (`src/types/book.ts`):
```typescript
export interface ViewSettings {
  // Paginated mode widgets
  showHeader: boolean;
  showFooter: boolean;

  // Scrolled mode widgets (separate controls)
  showHeaderInScrolled: boolean;
  showFooterInScrolled: boolean;

  // OR use the existing approach:
  showWidgetsInScrolledMode: boolean;  // Apply paginated settings to scrolled

  // ... other settings
}
```

### Default Behavior

**Paginated Mode** (Default):
- Header: Shown
- Footer: Shown
- Full UI with section title and progress

**Scrolled Mode** (Default):
- Header: Hidden (unless `showWidgetsInScrolledMode: true`)
- Footer: Hidden (unless `showWidgetsInScrolledMode: true`)
- Distraction-free scrolling experience

### Settings Location

**Path**: Settings Dialog → Layout Panel → Header & Footer section

**UI Implementation**:
```typescript
<div className="header-footer-settings">
  <h4>{_('Header & Footer Visibility')}</h4>

  {/* Paginated Mode */}
  <fieldset>
    <legend>{_('Paginated Mode')}</legend>
    <label>
      <input
        type="checkbox"
        checked={viewSettings.showHeader}
        onChange={(e) => updateSetting('showHeader', e.target.checked)}
      />
      {_('Show Header')}
    </label>
    <label>
      <input
        type="checkbox"
        checked={viewSettings.showFooter}
        onChange={(e) => updateSetting('showFooter', e.target.checked)}
      />
      {_('Show Footer')}
    </label>
  </fieldset>

  {/* Scrolled Mode */}
  <fieldset>
    <legend>{_('Scrolled Mode')}</legend>
    <label>
      <input
        type="checkbox"
        checked={viewSettings.showWidgetsInScrolledMode}
        onChange={(e) => updateSetting('showWidgetsInScrolledMode', e.target.checked)}
      />
      {_('Show Header & Footer in Scrolled Mode')}
    </label>
  </fieldset>
</div>
```

### Implementation Logic

**Conditional Rendering** (`src/app/reader/components/ReaderContent.tsx`):
```typescript
const shouldShowHeader = () => {
  if (viewSettings.scrolled) {
    return viewSettings.showWidgetsInScrolledMode;
  }
  return viewSettings.showHeader;
};

const shouldShowFooter = () => {
  if (viewSettings.scrolled) {
    return viewSettings.showWidgetsInScrolledMode;
  }
  return viewSettings.showFooter;
};

return (
  <div className="reader-content">
    {shouldShowHeader() && <HeaderBar />}
    <BookView />
    {shouldShowFooter() && <FooterBar />}
  </div>
);
```

### Use Cases

**Paginated Mode**:
- Traditional book-like experience
- Header shows chapter title
- Footer shows page numbers and progress
- Information-rich reading

**Scrolled Mode**:
- Distraction-free scrolling
- Minimal UI for immersive reading
- More like reading a web article
- Maximum vertical space

---

## 9. Compact Margin When Header/Footer Dismissed

**Added**: v0.9.39 (Commit 4c1af671, #1047, #734)

### Overview

When header and/or footer widgets are dismissed, the reading area automatically adjusts with compact margins and gap values to maximize content space. This ensures that hiding UI elements actually provides more reading space rather than leaving empty gaps.

### Implementation

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

### Visual Impact

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

### Margin Reduction Formula

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

### Application

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

### Settings Interaction

**Affected Settings**:
- `showHeader`: Triggers compact top margin when false
- `showFooter`: Triggers compact bottom margin when false
- `margin`: Base value for compact calculation
- `gap`: Base value for compact gap calculation

**Independent Controls**:
- Left/right margins remain at base value
- Only top/bottom margins are compacted
- Gap between columns is compacted when widgets hidden

### Use Cases

**Maximized Reading Space**:
- Small screens (mobile devices)
- Distraction-free reading
- Users who want minimal UI

**Precision Layout Control**:
- Fine-tune exact content area
- Optimize for specific screen sizes
- Balance aesthetics and functionality

---

## Summary Table (v0.9.32-0.9.43)

| Feature | Version | Issue # | User Benefit |
|---------|---------|---------|--------------|
| **Prev/Next Section Buttons** | v0.9.43 | #1195 | Quick chapter navigation without TOC |
| **Separate Mode Visibility** | v0.9.32 | #859 | Different UI for paginated vs scrolled |
| **Compact Margins** | v0.9.39 | #1047 | Maximized space when UI hidden |

---
