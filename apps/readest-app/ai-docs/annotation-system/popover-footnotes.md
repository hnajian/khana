# Popover Footnotes

**Added**: December 2024 (commits 0cbb950c, b57fd8bd)

## Overview

The Popover Footnotes feature displays footnotes in an inline popup without navigating away from the main text. It supports both horizontal and vertical writing modes, making it ideal for CJK languages.

## Key Components

**Primary Files:**
- **`src/app/reader/components/FootnotePopup.tsx`** - Main footnote popup component
- **`src/components/Popup.tsx`** - Reusable popup container with directional triangles
- **`src/utils/sel.ts`** - Position calculation utilities (getPosition, getPopupPosition)
- **`src/libs/document.ts`** - Direction detection (getDirection function)

## Footnote Flow

1. **User clicks footnote link** (e.g., `<a href="#fn1">¹</a>`)
2. **Event captured** in `useFoliateEvents` hook (`link` event)
3. **FootnoteHandler** from foliate-js renders footnote content
4. **FootnotePopup** component displays popup with:
   - Nested FoliateView rendering footnote HTML
   - Calculated position based on anchor location
   - Directional triangle pointing to source
   - Responsive sizing based on content
5. **User dismisses** popup by clicking outside or pressing Esc

## Vertical Writing Mode Support

**Detection:**
```typescript
// From document.ts
const getDirection = (doc: Document) => {
  const { writingMode, direction } = defaultView!.getComputedStyle(doc.body);
  const vertical = writingMode === 'vertical-rl' || writingMode === 'vertical-lr';
  const rtl = direction === 'rtl' || doc.body.dir === 'rtl';
  return { vertical, rtl };
};
```

**Position Calculation for Vertical Mode:**

For vertical text (common in Japanese, Chinese books):
- Popup positioned **left or right** of the footnote anchor (not up/down)
- Triangle points **left** (if popup on right) or **right** (if popup on left)
- Vertical centering at midpoint of anchor element

```typescript
// From sel.ts
function getPopupPosition(anchorRect, isVertical) {
  if (isVertical) {
    // Calculate left vs right space
    const spaceLeft = anchorRect.left;
    const spaceRight = window.innerWidth - anchorRect.right;

    // Place on side with more space
    const dir = spaceLeft > spaceRight ? 'left' : 'right';

    // Vertically center at anchor midpoint
    const top = anchorRect.top + anchorRect.height / 2;

    return { dir, top, left: dir === 'left' ? anchorRect.left - popupWidth - 6 : anchorRect.right + 6 };
  } else {
    // Standard horizontal positioning (up/down)
    // ...
  }
}
```

## Popup Directions

**Four directional triangles** (defined in `Popup.tsx`):
- **'up'**: Triangle points upward (horizontal mode, popup below text)
- **'down'**: Triangle points downward (horizontal mode, popup above text)
- **'left'**: Triangle points left (vertical mode, popup on right)
- **'right'**: Triangle points right (vertical mode, popup on left)

## Responsive Sizing

**FootnotePopup sizing logic:**
```typescript
function getResponsivePopupSize(isVertical, orientation) {
  if (isVertical) {
    // Vertical writing mode
    return {
      width: Math.min(window.innerHeight / 2, 600),
      height: 360
    };
  } else {
    // Horizontal writing mode
    return {
      width: 360,
      height: Math.min(window.innerHeight / 3, 400)
    };
  }
}
```

- Adapts to viewport size
- Different constraints for vertical vs horizontal modes
- Respects safe area margins (10px padding from edges)

## Footnote Rendering

**Nested FoliateView:**
- Footnote content rendered using same engine as main book
- Inherits theme colors and fonts from main reader
- Custom CSS applied for compact display
- Supports HTML formatting in footnotes
- Handles links within footnotes

**Style Application:**
```typescript
// From FootnotePopup.tsx
function handleBeforeRender(doc) {
  // Apply reader theme colors
  doc.body.style.color = theme.foregroundColor;
  doc.body.style.backgroundColor = theme.backgroundColor;

  // Apply custom fonts (if set)
  if (customFont) {
    doc.body.style.fontFamily = customFont;
  }

  // Compact margins for popup
  doc.body.style.margin = '8px';
}
```

## Event Handling

**Footnote Link Detection:**

Links are detected as footnote links if:
- `href` contains `#` (internal anchor)
- Link element is `<a>` or `<sup>`
- Target ID exists in document

**Custom Footnote Events:**
```typescript
// Alternative to foliate-js built-in footnotes
eventDispatcher.on('footnote-popup', handleFootnotePopupEvent);
```

Allows custom footnote implementations to trigger the popup.

## Integration with Annotator

Both `Annotator` and `FootnotePopup` use shared positioning utilities:
- `getPosition(range)` - Get bounding rect for selection/anchor
- `getPopupPosition(rect, isVertical)` - Calculate popup placement
- Both respect vertical writing mode
- Both use `Popup` component for consistent triangles

## Adding Custom Footnote Formatting

**Example: Styling footnote content**

1. **Modify `FootnotePopup.tsx` render handler**:
```typescript
function handleBeforeRender(doc) {
  // Add custom CSS for footnote styling
  const style = doc.createElement('style');
  style.textContent = `
    .footnote-number {
      font-weight: bold;
      color: var(--primary-color);
    }
    .footnote-text {
      font-size: 0.9em;
      line-height: 1.4;
    }
  `;
  doc.head.appendChild(style);
}
```

2. **Test with various footnote formats**:
   - Standard EPUB footnotes (`<aside>` or `<section epub:type="footnote">`)
   - Inline HTML footnotes
   - Reference-style footnotes (calibre format)

## Closing Popup

**Dismiss triggers:**
- Click outside popup
- Press Escape key
- Navigate to different page
- Close sidebar (if footnote was in sidebar TOC)
- Window resize event

**Cleanup:**
```typescript
function closePopup() {
  // Destroy nested FoliateView
  footnoteView?.destroy();

  // Clear state
  setShowPopup(false);
  setFootnoteContent(null);
}
```

## Accessibility

**Keyboard Support:**
- Footnote link clickable via keyboard (Tab + Enter)
- Popup dismissible via Escape key
- Focus remains on trigger link after closing

**Screen Reader Support:**
- Popup has `role="dialog"`
- Footnote content is readable
- ARIA label describes popup purpose

## Common Issues

**Issue: Popup positioned incorrectly in vertical mode**
- Check `getDirection()` correctly detects vertical writing mode
- Verify `isVertical` prop passed to position calculation
- Inspect CSS `writing-mode` on book body

**Issue: Footnote content empty**
- Verify footnote target ID exists in document
- Check `FootnoteHandler` in foliate-js is initialized
- Inspect network tab for footnote content loading
- Ensure footnote is in same spine item (EPUB) or same document

**Issue: Popup too small for content**
- Adjust responsive size calculation in `getResponsivePopupSize()`
- Check viewport constraints
- Increase max width/height limits

**Issue: Triangle pointing wrong direction**
- Verify position calculation in `getPopupPosition()`
- Check `dir` parameter matches popup placement
- Ensure CSS for triangle borders is correct

## Related Files

| File | Purpose |
|------|---------|
| `FootnotePopup.tsx:handleFootnotePopupEvent` | Custom footnote event handling |
| `Popup.tsx:styles` | Triangle CSS (lines 103-147) |
| `sel.ts:getPopupPosition` | Position calculation (lines 108-125) |
| `document.ts:getDirection` | Vertical mode detection (lines 227-233) |
| `useFoliateEvents.ts:link` | Footnote link event capture |
| `types/book.ts:BookLayout.vertical` | Vertical mode flag |

---

**Related**: [index.md](./index.md) (Main annotation system documentation)
