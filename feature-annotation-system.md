# Feature: Annotation System

## Overview

The Annotation System enables users to highlight text, add notes, and interact with selected text through various tools (translation, dictionary lookup). This feature is central to Readest's goal of providing an immersive deep-reading experience.

## Key Components

### Primary Files

- **`src/app/reader/components/annotator/Annotator.tsx`** - Main annotator component managing selection and annotation UI
- **`src/app/reader/components/annotator/AnnotationPopup.tsx`** - Popup showing existing annotation details
- **`src/app/reader/components/annotator/HighlightOptions.tsx`** - Color picker for highlighting
- **`src/app/reader/hooks/useFoliateEvents.ts`** - Bridges foliate-js events to React

### Related Files

- **`src/app/reader/components/annotator/DeepLPopup.tsx`** - Translation popup
- **`src/app/reader/components/annotator/WikipediaPopup.tsx`** - Wikipedia lookup popup
- **`src/app/reader/components/annotator/WiktionaryPopup.tsx`** - Dictionary lookup popup
- **`src/app/reader/components/annotator/PopupButton.tsx`** - Reusable button component for popups
- **`src/app/reader/utils/iframeEventHandlers.ts`** - Iframe event handling utilities
- **`src/store/notebookStore.ts`** - Stores annotations and notes

## Architecture

### Annotation Flow

1. **User selects text** in the book viewer (foliate-js iframe)
2. **`useFoliateEvents` hook** catches the `draw-annotation` event
3. **`Annotator` component** displays highlight options popup
4. **User chooses action**:
   - **Highlight**: Color is selected, annotation saved
   - **Translate**: Opens DeepL popup with translation
   - **Dictionary**: Opens Wikipedia/Wiktionary popup
   - **Note**: Opens note editor in notebook
5. **Annotation stored** in `notebookStore` and persisted via `appService`
6. **Overlays drawn** on all views of the same book via foliate-js API

### Component Hierarchy

```
ReaderContent
└── Annotator
    ├── HighlightOptions
    ├── AnnotationPopup
    ├── DeepLPopup
    ├── WikipediaPopup
    └── WiktionaryPopup
```

### State Management

**Notebook Store** (`src/store/notebookStore.ts`):
- Stores all annotations and notes for all books
- Structure:
  ```typescript
  {
    booknotes: {
      [bookHash: string]: {
        highlights: Annotation[],
        notes: Note[]
      }
    }
  }
  ```

**Annotation Data Structure**:
```typescript
interface Annotation {
  id: string;
  cfi: string;           // EPUB CFI or PDF location
  color: string;         // Highlight color
  text: string;          // Selected text
  note?: string;         // Optional note text
  created: number;       // Timestamp
  updated?: number;      // Last modified timestamp
}
```

### Integration with Foliate-js

The annotation system interacts with foliate-js through:

1. **Events from foliate-js** (caught in `useFoliateEvents.ts`):
   - `draw-annotation`: User made a text selection
   - `create-annotation`: Request to create annotation
   - `delete-annotation`: Request to delete annotation
   - `show-annotation`: Click on existing annotation

2. **Commands to foliate-js** (sent from React components):
   - `view.addAnnotation(annotation)`: Draw highlight overlay
   - `view.deleteAnnotation(id)`: Remove highlight
   - `view.getCFI()`: Get current selection CFI

### Highlight Color System

Predefined colors in `HighlightOptions.tsx`:
- Yellow (default)
- Green
- Blue
- Pink
- Purple

Colors are applied as semi-transparent overlays on the text.

## AI Agent Modification Guidelines

### Adding a New Annotation Action

To add a new action button (e.g., "Copy Quote"):

1. **Update `Annotator.tsx`**:
   ```typescript
   // Add new popup state
   const [showCopyPopup, setShowCopyPopup] = useState(false);

   // Add button in the highlight options area
   <PopupButton
     onClick={() => {
       navigator.clipboard.writeText(selectedText);
       // Show success toast
     }}
   >
     Copy Quote
   </PopupButton>
   ```

2. **Create new popup component** (if needed):
   - Follow pattern of `WikipediaPopup.tsx`
   - Place in `src/app/reader/components/annotator/`
   - Use `PopupButton` for consistent styling

3. **Update event handling** in `useFoliateEvents.ts` if new events are needed

### Changing Highlight Colors

To add or modify highlight colors:

1. **Edit `HighlightOptions.tsx`**:
   ```typescript
   const colors = [
     '#ffeb3b', // Yellow
     '#4caf50', // Green
     '#2196f3', // Blue
     '#e91e63', // Pink
     '#9c27b0', // Purple
     '#ff9800', // Orange (new)
   ];
   ```

2. **Update color display** logic if needed (e.g., color names)

3. **Consider accessibility**: Ensure sufficient contrast for readability

### Improving Annotation Persistence

To enhance how annotations are saved/loaded:

1. **Modify `notebookStore.ts`**:
   - Add methods for batch operations
   - Implement undo/redo functionality
   - Add export/import capabilities

2. **Update `appService.ts`**:
   - Enhance `saveBookConfig()` to handle annotations
   - Add backup/sync functionality

3. **Example: Add annotation export**:
   ```typescript
   // In notebookStore.ts
   exportAnnotations: (bookHash: string) => {
     const booknotes = get().booknotes[bookHash];
     const json = JSON.stringify(booknotes, null, 2);
     return new Blob([json], { type: 'application/json' });
   }
   ```

### Supporting Annotation Types

To add different annotation types (e.g., underline, strikethrough):

1. **Extend annotation interface** in `src/types/book.ts`:
   ```typescript
   interface Annotation {
     id: string;
     cfi: string;
     type: 'highlight' | 'underline' | 'strikethrough';
     color: string;
     text: string;
     // ...
   }
   ```

2. **Update foliate-js** integration:
   - Modify how annotations are rendered in iframe
   - Update CSS classes for different styles

3. **Update UI** in `HighlightOptions.tsx`:
   - Add type selector buttons
   - Update preview display

### Syncing Annotations Across Views

When a book is open in multiple views (grid layout):

1. **Current implementation**:
   - `readerStore.getViewsById()` gets all views of same book
   - Each view's foliate instance updates annotations
   - Located in: `Annotator.tsx` annotation save handler

2. **To improve sync**:
   - Add debouncing to prevent excessive updates
   - Batch annotation updates
   - Add conflict resolution for simultaneous edits

### Adding Translation Services

To add a new translation service besides DeepL:

1. **Create new popup component**:
   ```typescript
   // src/app/reader/components/annotator/GoogleTranslatePopup.tsx
   export function GoogleTranslatePopup({ text, onClose }) {
     // Implement Google Translate API integration
   }
   ```

2. **Add button in `Annotator.tsx`**:
   ```typescript
   <PopupButton onClick={() => setShowGoogleTranslate(true)}>
     Google Translate
   </PopupButton>
   ```

3. **Handle API keys** in settings:
   - Add to `SystemSettings` in `src/types/settings.ts`
   - Store in `settingsStore.ts`

### Handling Annotation Conflicts

For books with complex layouts (e.g., fixed-layout EPUBs):

1. **CFI validation**:
   - Add validation in annotation save handler
   - Handle cases where CFI becomes invalid after book updates

2. **Position recalculation**:
   - Implement logic to recalculate positions on layout changes
   - Store additional metadata for recovery

---

## Popover Footnotes (Added Dec 2024)

### Overview

The **Popover Footnotes** feature (commits 0cbb950c, b57fd8bd - Dec 2024) displays footnotes in an inline popup without navigating away from the main text. It supports both horizontal and vertical writing modes, making it ideal for CJK languages.

### Key Components

**Primary Files:**
- **`src/app/reader/components/FootnotePopup.tsx`** - Main footnote popup component
- **`src/components/Popup.tsx`** - Reusable popup container with directional triangles
- **`src/utils/sel.ts`** - Position calculation utilities (getPosition, getPopupPosition)
- **`src/libs/document.ts`** - Direction detection (getDirection function)

### Footnote Flow

1. **User clicks footnote link** (e.g., `<a href="#fn1">¹</a>`)
2. **Event captured** in `useFoliateEvents` hook (`link` event)
3. **FootnoteHandler** from foliate-js renders footnote content
4. **FootnotePopup** component displays popup with:
   - Nested FoliateView rendering footnote HTML
   - Calculated position based on anchor location
   - Directional triangle pointing to source
   - Responsive sizing based on content
5. **User dismisses** popup by clicking outside or pressing Esc

### Vertical Writing Mode Support

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

### Popup Directions

**Four directional triangles** (defined in `Popup.tsx`):
- **'up'**: Triangle points upward (horizontal mode, popup below text)
- **'down'**: Triangle points downward (horizontal mode, popup above text)
- **'left'**: Triangle points left (vertical mode, popup on right)
- **'right'**: Triangle points right (vertical mode, popup on left)

### Responsive Sizing

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

### Footnote Rendering

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

### Event Handling

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

### Integration with Annotator

Both `Annotator` and `FootnotePopup` use shared positioning utilities:
- `getPosition(range)` - Get bounding rect for selection/anchor
- `getPopupPosition(rect, isVertical)` - Calculate popup placement
- Both respect vertical writing mode
- Both use `Popup` component for consistent triangles

### Adding Custom Footnote Formatting

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

### Closing Popup

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

### Accessibility

**Keyboard Support:**
- Footnote link clickable via keyboard (Tab + Enter)
- Popup dismissible via Escape key
- Focus remains on trigger link after closing

**Screen Reader Support:**
- Popup has `role="dialog"`
- Footnote content is readable
- ARIA label describes popup purpose

### Common Issues

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

### Related Files

| File | Purpose |
|------|---------|
| `FootnotePopup.tsx:handleFootnotePopupEvent` | Custom footnote event handling |
| `Popup.tsx:styles` | Triangle CSS (lines 103-147) |
| `sel.ts:getPopupPosition` | Position calculation (lines 108-125) |
| `document.ts:getDirection` | Vertical mode detection (lines 227-233) |
| `useFoliateEvents.ts:link` | Footnote link event capture |
| `types/book.ts:BookLayout.vertical` | Vertical mode flag |

---

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add annotation action | `Annotator.tsx` | `PopupButton.tsx`, new popup component |
| Change highlight colors | `HighlightOptions.tsx` | - |
| Modify storage | `notebookStore.ts` | `appService.ts` |
| Add annotation type | `src/types/book.ts`, `Annotator.tsx` | foliate-js viewer |
| Add translation service | `Annotator.tsx`, new popup | `settingsStore.ts` |
| Fix event handling | `useFoliateEvents.ts` | `iframeEventHandlers.ts` |
| Improve sync | `Annotator.tsx` | `readerStore.ts` |

## Common Issues and Debugging

### Problem: Highlights not appearing

- Check if `view.addAnnotation()` is being called in `Annotator.tsx`
- Verify CFI is valid for the current book format
- Inspect foliate-js iframe for overlay elements
- Check CSS z-index conflicts

### Problem: Annotations lost after restart

- Verify `notebookStore` is persisting to `appService.saveBookConfig()`
- Check `bookDataStore` for loaded annotations
- Ensure book hash is consistent (check MD5 calculation)

### Problem: Selection not detected

- Check `useFoliateEvents.ts` is properly attached to view
- Verify iframe message passing is working
- Inspect foliate-js event emission

### Problem: Popup positioning incorrect

- Check viewport calculations in `Annotator.tsx`
- Verify scroll position handling
- Adjust popup CSS positioning logic

## Dependencies

- **foliate-js**: Provides selection and annotation overlay APIs
- **zustand**: State management for annotations
- **react-icons**: Icons for annotation UI
- **External APIs**: DeepL (translation), Wikipedia, Wiktionary

## Performance Considerations

- **Batch updates**: When adding/removing many annotations, batch the updates
- **Debounce saves**: Don't save on every keystroke in note editor
- **Lazy load popups**: Dynamic import heavy components (e.g., DeepL API)
- **Optimize re-renders**: Use React.memo for popup components
