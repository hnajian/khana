# 6. Scrolled Mode Toggler in Layout Panel

**Added**: v0.9.21 (Commit b7dc880a, #596, #580)

## Overview

The Scrolled Mode Toggler was added to the Layout Panel to streamline the workflow for switching between paginated and scrolled modes while adjusting layout settings. Previously, users had to switch between Behavior and Layout panels to toggle modes and adjust settings.

## Settings Location

**Path**: Settings Dialog → Layout Panel → Top of panel (first option)

**File**: `src/components/settings/LayoutPanel.tsx`

## Implementation

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

## Benefits

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

## Mode-Specific Layout Behavior

**Paginated Mode**:
- `maxColumnCount`: Controls number of columns per page
- `maxInlineSize`: Maximum width per column
- Margins affect page boundaries

**Scrolled Mode**:
- `maxColumnCount`: Usually 1 (single column scrolling)
- `maxInlineSize`: Maximum content width
- Margins create padding around scrollable content

## Real-Time Updates

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

## Visual Feedback

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

## Interaction with Other Settings

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

## Keyboard Shortcut (Optional Enhancement)

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
