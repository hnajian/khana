# Feature: Annotation System

## Overview

The Annotation System enables users to highlight text, add notes, and interact with selected text through various tools (translation, dictionary lookup). This feature is central to Readest's goal of providing an immersive deep-reading experience.

## Sub-Features

This feature includes several specialized sub-features:

- **[Keyboard Shortcuts](./keyboard-shortcuts.md)** - Quick access to annotation actions via keyboard (v0.9.11+)
- **[Popover Footnotes](./popover-footnotes.md)** - Inline footnote display with vertical writing mode support (Dec 2024)
- **[Cloud Notes Synchronization](./cloud-notes-sync.md)** - Automatic sync of annotations across devices (Dec 2024)
- **[TTS Integration](./tts-integration.md)** - Text-to-speech from annotation toolbar (Jan 2025)

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

### Highlight Color System (Updated v0.9.17)

Predefined colors in `HighlightOptions.tsx` updated in commit #453 with less saturated, modern palette:

**Current Colors** (as of v0.9.17):
- Yellow: `#FFF9C4` (previously `#ffeb3b` - less saturated for better readability)
- Green: `#C8E6C9` (previously `#4caf50`)
- Blue: `#BBDEFB` (previously `#2196f3`)
- Pink: `#F8BBD0` (previously `#e91e63`)
- Purple: `#E1BEE7` (previously `#9c27b0`)

**Rationale**: Improved readability with lower saturation values, better contrast for both light and dark themes, reduced eye strain during extended reading sessions.

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
     '#FFF9C4', // Yellow
     '#C8E6C9', // Green
     '#BBDEFB', // Blue
     '#F8BBD0', // Pink
     '#E1BEE7', // Purple
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
