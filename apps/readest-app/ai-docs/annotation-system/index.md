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

## Updates (v0.9.32-0.9.43)

### Text Selector Hook Refactor (v0.9.41, #1131, #1020)

**Overview**: Refactored text selection logic into a reusable hook for better code organization and maintainability.

**File**: `src/app/reader/hooks/useTextSelector.ts`

**Benefits**:
- Centralized text selection logic
- Easier to test and maintain
- Reusable across annotator and other components
- Better separation of concerns

### Selection Anchor Preservation (v0.9.38, #968, #873)

**Issue**: Selection anchor was lost when spanning across multiple pages in paginated content.

**Fix**: Implemented anchor preservation logic to maintain selection across page boundaries.

**Implementation** (`src/app/reader/components/annotator/Annotator.tsx`):
```typescript
const preserveSelectionAnchor = (selection: Selection) => {
  // Store anchor node and offset
  const anchorNode = selection.anchorNode;
  const anchorOffset = selection.anchorOffset;

  // Restore after page turn
  selection.setBaseAndExtent(anchorNode, anchorOffset, focusNode, focusOffset);
};
```

### Highlight Positioning Improvements (v0.9.32, #845)

**Feature**: Underline and squiggly highlight decorations now positioned at the middle between text lines for better visual appeal.

**CSS Update**:
```css
.highlight-underline {
  text-decoration-line: underline;
  text-decoration-style: solid;
  text-underline-offset: 0.15em;  /* Middle between lines */
}

.highlight-squiggly {
  text-decoration-line: underline;
  text-decoration-style: wavy;
  text-underline-offset: 0.15em;
}
```

### Popup Footnotes Enhancements (v0.9.37-0.9.40)

**Namespace Handling** (v0.9.37, #956):
- Fixed popup footnotes for anchors without EPUB namespace
- More robust anchor detection
- Works with non-standard EPUB files

**Font Inheritance** (v0.9.38, #985):
- Popup footnotes now inherit book fonts
- Consistent typography between main text and footnotes
- Better readability

**Visibility Fixes** (v0.9.40, #1099):
- Fixed footnote visibility in certain layouts
- Improved z-index handling
- Better positioning on small screens

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### Markdown Note Support (v0.9.52, #1315)

**Major Feature**: Support for note taking with markdown formatting.

**Overview**: Users can now write notes with markdown syntax, including headers, lists, code blocks, links, and more.

**Supported Markdown Features**:
- **Headers**: `# H1`, `## H2`, `### H3`
- **Emphasis**: `*italic*`, `**bold**`, `***bold italic***`
- **Lists**: Ordered (`1.`) and unordered (`-`, `*`)
- **Links**: `[text](url)`
- **Code**: Inline `` `code` `` and code blocks ` ``` `
- **Blockquotes**: `> quote`
- **Tables**: GitHub-flavored markdown tables
- **Checklists**: `- [ ]` and `- [x]`

**Implementation** (`src/app/reader/components/notebook/NoteEditor.tsx`):
```typescript
import ReactMarkdown from 'react-markdown';

function NoteEditor({ note, onChange }: NoteEditorProps) {
  const [markdown, setMarkdown] = useState(note.content);
  const [isPreview, setIsPreview] = useState(false);

  return (
    <div className="note-editor">
      <div className="editor-toolbar">
        <button onClick={() => setIsPreview(!isPreview)}>
          {isPreview ? 'Edit' : 'Preview'}
        </button>
      </div>

      {isPreview ? (
        <ReactMarkdown className="markdown-preview">
          {markdown}
        </ReactMarkdown>
      ) : (
        <textarea
          value={markdown}
          onChange={(e) => setMarkdown(e.target.value)}
          placeholder="Write your note in markdown..."
        />
      )}
    </div>
  );
}
```

**UI Components**:
- Split editor/preview mode
- Toolbar with markdown shortcuts
- Syntax highlighting for code blocks
- Live preview toggle
- Export notes as markdown files

**Files**:
- `src/app/reader/components/notebook/NoteEditor.tsx` - Markdown editor
- `src/app/reader/components/notebook/MarkdownToolbar.tsx` - Formatting toolbar
- `src/store/notebookStore.ts` - Updated to store markdown content

**Styling**: Custom CSS for markdown rendering with book theme integration.

### Notebook Search Functionality (v0.9.52, #1318)

**Feature**: Search annotations and notes by keyword.

**Search Capabilities**:
- Full-text search across all notes and highlights
- Search by note content
- Search by highlighted text
- Search by book title/author
- Filter by book
- Filter by color
- Sort by date created/modified

**Implementation** (`src/app/reader/components/notebook/NotebookSearch.tsx`):
```typescript
interface SearchOptions {
  query: string;
  bookFilter?: string;      // Filter by specific book
  colorFilter?: string;     // Filter by highlight color
  type?: 'highlight' | 'note' | 'all';
  sortBy?: 'created' | 'modified' | 'relevance';
}

function searchNotes(options: SearchOptions): Note[] {
  const { query, bookFilter, colorFilter, type, sortBy } = options;

  let results = getAllNotes();

  // Filter by book
  if (bookFilter) {
    results = results.filter(n => n.bookHash === bookFilter);
  }

  // Filter by type
  if (type && type !== 'all') {
    results = results.filter(n => n.type === type);
  }

  // Filter by color
  if (colorFilter) {
    results = results.filter(n => n.color === colorFilter);
  }

  // Full-text search
  if (query) {
    results = results.filter(n =>
      n.text.toLowerCase().includes(query.toLowerCase()) ||
      n.note?.toLowerCase().includes(query.toLowerCase())
    );
  }

  // Sort results
  switch (sortBy) {
    case 'created':
      results.sort((a, b) => b.created - a.created);
      break;
    case 'modified':
      results.sort((a, b) => (b.updated || b.created) - (a.updated || a.created));
      break;
    case 'relevance':
      // Score by query match frequency
      results.sort((a, b) => scoreRelevance(b, query) - scoreRelevance(a, query));
      break;
  }

  return results;
}
```

**UI Features**:
- Search bar in notebook panel
- Real-time search as you type
- Search result highlighting
- Click result to jump to location in book
- Clear search button
- Search history dropdown

**Keyboard Shortcuts**:
- `Ctrl/Cmd+F` - Focus search bar (in notebook)
- `Esc` - Clear search
- `Enter` - Jump to first result
- `↑/↓` - Navigate results

**Files**:
- `src/app/reader/components/notebook/NotebookSearch.tsx`
- `src/app/reader/components/notebook/SearchResults.tsx`
- `src/store/notebookStore.ts` - Search state management

### Notebook Layout Tweaks (v0.9.52, #1319)

**Enhancement**: Improved notebook layout for better readability and organization.

**Changes**:
- Responsive card-based layout
- Better spacing between notes
- Collapsible sections for highlights vs notes
- Sticky headers for sections
- Improved mobile layout
- Touch-friendly interaction areas

**File**: `src/app/reader/components/notebook/NotebookLayout.tsx`

### Show Annotation Create Time (v0.9.58, #1412)

**Feature**: Display creation time and last modified time for each annotation.

**UI**:
- Timestamp shown below each note/highlight
- Relative time format ("2 hours ago", "3 days ago")
- Absolute time on hover (tooltip)
- Sort notes by creation time or modification time

**Implementation** (`src/app/reader/components/notebook/AnnotationCard.tsx`):
```typescript
function AnnotationCard({ annotation }: AnnotationCardProps) {
  const { created, updated } = annotation;

  return (
    <div className="annotation-card">
      {/* ... annotation content ... */}

      <div className="annotation-metadata">
        <span className="timestamp" title={new Date(created).toLocaleString()}>
          Created {formatRelativeTime(created)}
        </span>
        {updated && updated !== created && (
          <span className="timestamp-modified" title={new Date(updated).toLocaleString()}>
            Modified {formatRelativeTime(updated)}
          </span>
        )}
      </div>
    </div>
  );
}

function formatRelativeTime(timestamp: number): string {
  const now = Date.now();
  const diff = now - timestamp;
  const seconds = Math.floor(diff / 1000);
  const minutes = Math.floor(seconds / 60);
  const hours = Math.floor(minutes / 60);
  const days = Math.floor(hours / 24);

  if (days > 0) return `${days} day${days > 1 ? 's' : ''} ago`;
  if (hours > 0) return `${hours} hour${hours > 1 ? 's' : ''} ago`;
  if (minutes > 0) return `${minutes} minute${minutes > 1 ? 's' : ''} ago`;
  return 'Just now';
}
```

**Files**:
- `src/app/reader/components/notebook/AnnotationCard.tsx`
- `src/utils/time.ts` - Time formatting utilities

### Restore Full View Settings When Reopening Book (v0.9.57, #1400)

**Feature**: Annotation-related view settings are now properly restored when reopening a book.

**Restored Settings**:
- Highlight visibility toggle state
- Note panel expansion state
- Selected annotation (if editing when closed)
- Scroll position in notebook

**Implementation**:
- Settings saved in BookConfig.viewSettings
- Automatic restoration on book load
- Graceful degradation if settings invalid

---

## Version 0.9.83 - 0.9.90 Updates (e1691661 → dd5371d2)

### Custom Highlight Color Picker (v0.9.88, #2273)

**Major Feature**: Ability to customize highlight colors with hex color picker.

**Overview**: Users can now set custom colors for all five highlight styles (red, violet, blue, green, yellow) using a hex color picker instead of being limited to predefined colors.

**Settings Location**: Settings > Annotations > "Customize Highlight Colors"

**Features**:
- Hex color input for each highlight style
- Color preview with live update
- Reset to default colors button
- Per-user customization (saved globally)
- Responsive layout for color options (#2303)

**Implementation** (`src/app/reader/components/settings/AnnotationSettings.tsx`):
```typescript
interface HighlightColorCustomization {
  color1: string;  // Default: Yellow
  color2: string;  // Default: Green
  color3: string;  // Default: Blue
  color4: string;  // Default: Pink
  color5: string;  // Default: Purple
}

const DEFAULT_COLORS: HighlightColorCustomization = {
  color1: '#FFF9C4',  // Yellow
  color2: '#C8E6C9',  // Green
  color3: '#BBDEFB',  // Blue
  color4: '#F8BBD0',  // Pink
  color5: '#E1BEE7'   // Purple
};

const CustomHighlightColorPicker = () => {
  const [colors, setColors] = useState<HighlightColorCustomization>(
    settingsStore.getState().highlightColors || DEFAULT_COLORS
  );

  const updateColor = (colorKey: keyof HighlightColorCustomization, value: string) => {
    // Validate hex color
    if (!/^#[0-9A-F]{6}$/i.test(value)) {
      return; // Invalid hex color
    }

    const newColors = { ...colors, [colorKey]: value };
    setColors(newColors);

    // Save to settings
    settingsStore.setHighlightColors(newColors);

    // Apply to existing highlights in current book
    applyColorChangesToHighlights(colorKey, value);
  };

  const resetToDefaults = () => {
    setColors(DEFAULT_COLORS);
    settingsStore.setHighlightColors(DEFAULT_COLORS);

    // Reapply default colors to all highlights
    Object.keys(DEFAULT_COLORS).forEach((key, index) => {
      applyColorChangesToHighlights(key as keyof HighlightColorCustomization, DEFAULT_COLORS[key]);
    });
  };

  return (
    <div className="custom-highlight-colors">
      <h3>{t('Customize Highlight Colors')}</h3>

      <div className="color-pickers-grid">
        {Object.entries(colors).map(([key, value], index) => (
          <div key={key} className="color-picker-item">
            <label className="label">
              <span className="label-text">{t(`Color ${index + 1}`)}</span>
            </label>

            <div className="color-input-group">
              <input
                type="color"
                value={value}
                onChange={(e) => updateColor(key as keyof HighlightColorCustomization, e.target.value)}
                className="color-picker"
              />

              <input
                type="text"
                value={value}
                onChange={(e) => updateColor(key as keyof HighlightColorCustomization, e.target.value)}
                placeholder="#FFFFFF"
                pattern="^#[0-9A-Fa-f]{6}$"
                className="input input-bordered hex-input"
              />

              <div
                className="color-preview"
                style={{ backgroundColor: value }}
                title={`Preview: ${value}`}
              >
                <span className="preview-text">Sample</span>
              </div>
            </div>
          </div>
        ))}
      </div>

      <div className="actions">
        <button onClick={resetToDefaults} className="btn btn-outline">
          {t('Reset to Defaults')}
        </button>
      </div>
    </div>
  );
};
```

**Responsive Layout** (#2303):
```css
.color-pickers-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
  margin: 1rem 0;
}

@media (max-width: 768px) {
  .color-pickers-grid {
    grid-template-columns: 1fr;
  }

  .color-input-group {
    flex-direction: column;
    align-items: stretch;
  }

  .color-preview {
    width: 100%;
    height: 50px;
  }
}
```

**Applying Custom Colors to Highlights**:
```typescript
const applyColorChangesToHighlights = (
  colorKey: keyof HighlightColorCustomization,
  newColor: string
) => {
  // Get color index (color1 -> 0, color2 -> 1, etc.)
  const colorIndex = parseInt(colorKey.replace('color', '')) - 1;

  // Update all highlights using this color
  const currentBook = readerStore.getState().currentBook;
  if (!currentBook) return;

  const annotations = notebookStore.getState().booknotes[currentBook.hash]?.highlights || [];

  annotations
    .filter(a => a.colorIndex === colorIndex)
    .forEach(annotation => {
      // Update annotation color
      annotation.color = newColor;

      // Redraw annotation overlay in all views
      const views = readerStore.getViewsById(currentBook.hash);
      views.forEach(view => {
        view.updateAnnotation(annotation.id, { color: newColor });
      });
    });

  // Save updated annotations
  notebookStore.saveAnnotations(currentBook.hash);
};
```

**Color Storage**:
```typescript
interface SystemSettings {
  // ...existing settings
  highlightColors?: HighlightColorCustomization;
}

// In settingsStore.ts
const useSettingsStore = create<SettingsStore>((set, get) => ({
  // ...
  setHighlightColors: (colors: HighlightColorCustomization) => {
    set((state) => ({
      settings: {
        ...state.settings,
        highlightColors: colors
      }
    }));

    // Persist to disk
    appService.saveSettings(get().settings);
  }
}));
```

**Benefits**:
- Personal color preferences for highlighting
- Better accessibility (choose high-contrast colors)
- Color-coding for different types of notes
- Thematic color schemes (e.g., warm vs cool colors)
- Support for colorblind users

**Use Cases**:
- Color-code by importance (red = critical, yellow = review)
- Match highlight colors to book themes
- Use high-contrast colors for better visibility
- Accessibility: Choose colors that work with vision impairments

**Files**:
- `src/app/reader/components/settings/AnnotationSettings.tsx` - Color picker UI
- `src/store/settingsStore.ts` - Color storage
- `src/app/reader/components/annotator/HighlightOptions.tsx` - Apply custom colors
- `src/types/settings.ts` - Type definitions

### Load Annotations of Current Section on Open (#2174)

**Enhancement**: Annotations for the current section are now drawn immediately when opening a book.

**Behavior**:
- Previously: Annotations loaded after navigation or page turn
- Now: Annotations visible as soon as book opens to last reading position

**Implementation**:
```typescript
// In FoliateViewer.tsx
useEffect(() => {
  if (view && bookHash) {
    // Get current section/chapter CFI
    const currentCFI = view.getCurrentLocation();

    // Load annotations for this section
    const sectionAnnotations = notebookStore
      .getAnnotationsBySection(bookHash, currentCFI);

    // Draw annotations
    sectionAnnotations.forEach(annotation => {
      view.addAnnotation(annotation);
    });
  }
}, [view, bookHash]);
```

**Files**:
- `src/app/reader/components/FoliateViewer.tsx` - Annotation loading on mount

---

## Version 0.9.91 Updates (dd5371d2 → 8ee53d3)

### Enhanced PDF Context Menu for Translation and Touch Handling (v0.9.91, #2430)

**Enhancement**: Improved PDF context menu functionality for text selection, translation, and better touch device support.

**Overview**: PDF context menus now provide better support for translation workflows and touch-based interactions, matching the functionality available in EPUB books.

**Key Improvements**:

1. **Translation Context Menu** - Right-click/long-press on selected PDF text now shows translation option
2. **Touch Gesture Support** - Long-press selection on touchscreens properly triggers context menu
3. **Better Selection Handling** - PDF text selection more reliably triggers annotation popup
4. **Unified Annotation UX** - PDF and EPUB now have consistent annotation workflows

**Previous Behavior**:
- PDF selection required awkward mouse operations
- Translation not available from context menu
- Touch devices couldn't access PDF annotation tools
- Context menu would disappear before user could click

**Implementation** (`src/app/reader/components/PDFAnnotator.tsx`):
```typescript
const PDFAnnotator = ({ view, bookHash }: PDFAnnotatorProps) => {
  const [contextMenuPosition, setContextMenuPosition] = useState<{x: number, y: number} | null>(null);
  const [selectedText, setSelectedText] = useState<string>('');
  const [selectionRect, setSelectionRect] = useState<DOMRect | null>(null);

  // Handle PDF text selection (mouse and touch)
  useEffect(() => {
    if (!view) return;

    const handleTextSelection = (e: MouseEvent | TouchEvent) => {
      // Debounce to prevent menu flickering
      clearTimeout(selectionTimeout);
      selectionTimeout = setTimeout(() => {
        const selection = window.getSelection();
        const text = selection?.toString().trim();

        if (text && text.length > 0) {
          setSelectedText(text);

          // Get selection bounding box
          const range = selection.getRangeAt(0);
          const rect = range.getBoundingClientRect();
          setSelectionRect(rect);

          // Show context menu below selection
          setContextMenuPosition({
            x: rect.left + rect.width / 2,
            y: rect.bottom + 5
          });
        } else {
          // Clear menu if no selection
          setContextMenuPosition(null);
        }
      }, 300);  // 300ms debounce
    };

    // Mouse selection
    view.container.addEventListener('mouseup', handleTextSelection);

    // Touch selection (long-press)
    let touchStartTime = 0;
    const handleTouchStart = (e: TouchEvent) => {
      touchStartTime = Date.now();
    };

    const handleTouchEnd = (e: TouchEvent) => {
      const touchDuration = Date.now() - touchStartTime;

      // Long-press (>500ms) triggers selection menu
      if (touchDuration > 500) {
        handleTextSelection(e);
      }
    };

    view.container.addEventListener('touchstart', handleTouchStart);
    view.container.addEventListener('touchend', handleTouchEnd);

    return () => {
      view.container.removeEventListener('mouseup', handleTextSelection);
      view.container.removeEventListener('touchstart', handleTouchStart);
      view.container.removeEventListener('touchend', handleTouchEnd);
    };
  }, [view]);

  // Render context menu
  return (
    <>
      {contextMenuPosition && (
        <div
          className="pdf-context-menu"
          style={{
            position: 'absolute',
            left: contextMenuPosition.x,
            top: contextMenuPosition.y,
            zIndex: 1000
          }}
          onMouseLeave={() => {
            // Delay hiding to allow click
            setTimeout(() => setContextMenuPosition(null), 300);
          }}
        >
          {/* Highlight button */}
          <button onClick={() => createHighlight(selectedText, selectionRect)}>
            <HighlightIcon /> Highlight
          </button>

          {/* Note button */}
          <button onClick={() => createNote(selectedText, selectionRect)}>
            <NoteIcon /> Add Note
          </button>

          {/* Translation button */}
          <button onClick={() => openTranslation(selectedText)}>
            <TranslateIcon /> Translate
          </button>

          {/* Dictionary button */}
          <button onClick={() => openDictionary(selectedText)}>
            <BookIcon /> Dictionary
          </button>

          {/* Copy button */}
          <button onClick={() => copyToClipboard(selectedText)}>
            <CopyIcon /> Copy
          </button>
        </div>
      )}
    </>
  );
};
```

**Translation Integration** (`src/app/reader/components/annotator/TranslationPopup.tsx`):
```typescript
const openTranslation = async (text: string) => {
  // Open translation popup
  setTranslationPopup({
    text,
    position: contextMenuPosition,
    onClose: () => setTranslationPopup(null)
  });

  // Close context menu
  setContextMenuPosition(null);

  // Fetch translation in background
  const translation = await translateText(text, {
    from: bookLanguage,
    to: userLanguage
  });

  setTranslationPopup(prev => ({
    ...prev,
    translation
  }));
};
```

**Touch Improvements**:
- Long-press (>500ms) triggers selection menu
- Visual feedback during long-press
- Prevents accidental menu opening
- Works with pinch-zoom active

**Context Menu Positioning**:
- Appears below selected text (not overlapping)
- Adjusts if near screen edge
- Stays visible long enough to click
- Auto-hides when clicking elsewhere

**Benefits**:
- Consistent annotation UX across EPUB and PDF
- Better support for language learners (quick translation)
- Touch device users can annotate PDFs
- Fewer accidental menu closures

**Files**:
- `src/app/reader/components/PDFAnnotator.tsx` - PDF context menu
- `src/app/reader/components/annotator/TranslationPopup.tsx` - Translation UI
- `src/styles/pdf-annotator.css` - Context menu styling

### Support for More Footnote Formats (v0.9.91, #2425)

**Enhancement**: Improved compatibility with various EPUB footnote formats and conventions.

**Overview**: Readest now recognizes and properly displays footnotes from a wider range of EPUB publishers and formats, including non-standard implementations.

**Supported Footnote Formats**:

1. **Standard EPUB 3** (existing support)
   ```html
   <a epub:type="noteref" href="#note1">1</a>
   <aside epub:type="footnote" id="note1">...</aside>
   ```

2. **EPUB 2 Format** (NEW)
   ```html
   <a class="footnote" href="#fn1">1</a>
   <div class="footnote-text" id="fn1">...</div>
   ```

3. **Legacy HTML Format** (NEW)
   ```html
   <sup><a href="#footnote-1">[1]</a></sup>
   <p class="footnote" id="footnote-1">...</p>
   ```

4. **Kindle Format** (NEW)
   ```html
   <a id="refnote1" href="#note1">1</a>
   <div id="note1" class="note">...</div>
   ```

5. **Publisher-Specific Formats** (NEW)
   - Penguin: `class="penguin-footnote"`
   - Oxford: `class="oxford-note"`
   - Cambridge: `class="note-reference"`
   - O'Reilly: `data-type="footnote"`

**Detection Logic** (`packages/foliate-js/footnotes.ts`):
```typescript
const isFootnoteReference = (element: HTMLAnchorElement): boolean => {
  // 1. Standard EPUB 3
  if (element.getAttribute('epub:type') === 'noteref') {
    return true;
  }

  // 2. EPUB 2 and legacy class-based
  const classList = element.classList;
  const footnoteClasses = [
    'footnote',
    'footnote-ref',
    'note',
    'note-ref',
    'endnote',
    'endnote-ref',
    // Publisher-specific
    'penguin-footnote',
    'oxford-note',
    'cambridge-note',
    'note-reference'
  ];

  if (footnoteClasses.some(cls => classList.contains(cls))) {
    return true;
  }

  // 3. Data attributes
  if (element.dataset.type === 'footnote' || element.dataset.noteref) {
    return true;
  }

  // 4. Heuristic: <sup> tag with link to fragment
  if (element.parentElement?.tagName === 'SUP' &&
      element.getAttribute('href')?.startsWith('#')) {
    return true;
  }

  // 5. Heuristic: Link text is numeric or has brackets
  const text = element.textContent?.trim();
  if (text && (/^\d+$/.test(text) || /^\[\d+\]$/.test(text))) {
    const href = element.getAttribute('href');
    if (href?.startsWith('#')) {
      return true;
    }
  }

  return false;
};

const getFootnoteTarget = (referenceElement: HTMLAnchorElement): HTMLElement | null => {
  const href = referenceElement.getAttribute('href');
  if (!href || !href.startsWith('#')) return null;

  const targetId = href.substring(1);

  // Try direct ID match
  let target = document.getElementById(targetId);
  if (target) return target;

  // Try epub:type="footnote"
  target = document.querySelector(`[epub\\:type="footnote"][id="${targetId}"]`);
  if (target) return target as HTMLElement;

  // Try common footnote classes
  const selectors = [
    `.footnote[id="${targetId}"]`,
    `.footnote-text[id="${targetId}"]`,
    `.note[id="${targetId}"]`,
    `aside[id="${targetId}"]`,
    `[data-type="footnote"][id="${targetId}"]`
  ];

  for (const selector of selectors) {
    target = document.querySelector(selector);
    if (target) return target as HTMLElement;
  }

  return null;
};
```

**Popup Rendering** (`packages/foliate-js/popover-footnotes.js`):
```typescript
const showFootnotePopup = (reference: HTMLAnchorElement) => {
  const target = getFootnoteTarget(reference);
  if (!target) return;

  // Extract footnote content
  let content = target.innerHTML;

  // Clean up footnote content
  content = cleanFootnoteContent(content);

  // Show popup below reference
  const rect = reference.getBoundingClientRect();
  const popup = createFootnotePopup(content, {
    x: rect.left,
    y: rect.bottom + 5
  });

  document.body.appendChild(popup);
};

const cleanFootnoteContent = (html: string): string => {
  const container = document.createElement('div');
  container.innerHTML = html;

  // Remove back-reference links (often included in footnotes)
  const backRefs = container.querySelectorAll('a[href^="#ref"], .footnote-backref');
  backRefs.forEach(ref => ref.remove());

  // Remove footnote number if duplicated
  const firstChild = container.firstElementChild;
  if (firstChild?.tagName === 'SUP') {
    firstChild.remove();
  }

  return container.innerHTML;
};
```

**User Experience**:
- Click/tap footnote reference → Inline popup appears
- Popup shows footnote text without navigating away
- Close popup to continue reading
- Works across all supported formats

**Fallback Behavior**:
- If footnote target not found, navigation to target page
- Warning logged to console for debugging
- Graceful degradation for unsupported formats

**Benefits**:
- Works with books from more publishers
- No manual format detection required
- Better compatibility with older EPUBs
- Consistent footnote UX across books

**Files**:
- `packages/foliate-js/footnotes.ts` - Footnote detection
- `packages/foliate-js/popover-footnotes.js` - Popup rendering
- `src/app/reader/components/FoliateViewer.tsx` - Integration

---

## Dependencies

- **foliate-js**: Provides selection and annotation overlay APIs
- **zustand**: State management for annotations
- **react-icons**: Icons for annotation UI
- **react-markdown**: Markdown rendering for notes
- **External APIs**: DeepL (translation), Wikipedia, Wiktionary

## Performance Considerations

- **Batch updates**: When adding/removing many annotations, batch the updates
- **Debounce saves**: Don't save on every keystroke in note editor
- **Lazy load popups**: Dynamic import heavy components (e.g., DeepL API)
- **Optimize re-renders**: Use React.memo for popup components
- **Search indexing**: Consider indexing notes for faster full-text search on large libraries
