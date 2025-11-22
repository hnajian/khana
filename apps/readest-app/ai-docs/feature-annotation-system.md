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

---

## Cloud Notes Synchronization (Added Dec 2024)

### Overview

The **Cloud Notes Synchronization** feature (commit 83cb7166 - Dec 24, 2024) enables automatic synchronization of annotations and notes across devices via Supabase cloud storage. This allows users to access their highlights, notes, and bookmarks from any device.

### Key Components

**Primary Files:**
- **`src/app/reader/hooks/useNotesSync.ts`** - Hook for automatic notes synchronization
- **`src/hooks/useSync.ts`** - Shared sync utilities
- **`src/pages/api/sync.ts`** - Cloud sync API endpoint
- **`src/utils/supabase.ts`** - Supabase client utilities
- **`src/context/SyncContext.tsx`** - Cloud sync state context

### Architecture

**Sync Flow:**
1. **User authenticates** via OAuth (see `src/app/auth/`)
2. **On book open**, `useNotesSync` hook pulls latest notes from cloud
3. **Local changes tracked** - New/modified annotations detected based on `updatedAt` timestamp
4. **Periodic sync** - Notes automatically synced at regular intervals (SYNC_NOTES_INTERVAL_SEC)
5. **Conflict resolution** - Merges local and remote notes, prioritizing newer timestamps
6. **Soft deletes** - Deleted notes marked with `deletedAt` timestamp, synced across devices

### Notes Sync Hook

**`useNotesSync(bookKey: string)`**

Automatically invoked in `Annotator.tsx:40` for every book:

```typescript
useNotesSync(bookKey);
```

**How it works:**

1. **Initial Pull** (on mount):
   ```typescript
   useEffect(() => {
     if (!user) return;
     syncNotes([], bookHash, 'pull');
   }, []);
   ```
   Downloads all existing notes for the book from cloud.

2. **Change Detection**:
   ```typescript
   const getNewNotes = () => {
     const bookNotes = config.booknotes ?? [];
     const newNotes = bookNotes.filter(
       (note) => lastSyncedAtNotes < note.updatedAt ||
                 lastSyncedAtNotes < (note.deletedAt ?? 0)
     );
     return newNotes;
   };
   ```
   Identifies notes modified since last sync.

3. **Periodic Sync**:
   - Syncs every `SYNC_NOTES_INTERVAL_SEC` seconds (default: configurable in constants)
   - Uses debouncing to avoid excessive sync calls
   - Mode: `'both'` (pushes local changes AND pulls remote changes)

4. **Merge Strategy**:
   ```typescript
   const mergedNotes = [
     ...oldNotes.filter((oldNote) =>
       !newNotes.some((newNote) => newNote.id === oldNote.id)
     ),
     ...newNotes,
   ];
   ```
   Remote notes override local notes with same ID.

### Data Structure

**BookNote Interface** (synchronized to cloud):
```typescript
interface BookNote {
  id: string;                // Unique note ID
  type: 'annotation' | 'excerpt' | 'bookmark';
  cfi: string;               // EPUB CFI or PDF location
  style?: 'highlight' | 'underline' | 'squiggly';
  color?: string;            // Highlight color
  text?: string;             // Selected text
  note?: string;             // User's note
  createdAt: number;         // Creation timestamp
  updatedAt: number;         // Last modification timestamp
  deletedAt?: number;        // Soft delete timestamp
  bookHash: string;          // Book identifier (MD5)
}
```

### Integration Points

**Annotator Actions that Trigger Sync:**
- **Highlight**: Creates annotation → triggers sync
- **Annotate**: Adds note to annotation → triggers sync
- **Copy**: Creates excerpt → triggers sync
- **Delete Highlight**: Soft-deletes annotation (sets `deletedAt`) → triggers sync

**Authentication Required:**
- Notes sync only active when `user` is authenticated (via `useAuth()`)
- Unauthenticated users store notes locally only
- Upon login, local notes can be merged with cloud notes

### Sync Timing

**Constants** (from `src/services/constants.ts`):
```typescript
export const SYNC_NOTES_INTERVAL_SEC = 30;  // Sync every 30 seconds
```

**Debouncing Logic:**
- If notes change within sync interval, sync is delayed
- Prevents excessive API calls when user is actively annotating
- Final sync occurs after user stops making changes

### Cloud Storage

**Supabase Backend:**
- Notes stored in Supabase PostgreSQL database
- Indexed by `bookHash` and `userId`
- Real-time subscriptions possible (future enhancement)

**API Endpoint** (`src/pages/api/sync.ts`):
- POST `/api/sync` - Sync notes for a specific book
- Request body: `{ bookHash, notes, lastSyncedAt }`
- Response: `{ syncedNotes, lastSyncedAtNotes }`

### Modification Guidelines for AI Agents

#### Adjusting Sync Interval

To change sync frequency:

1. **Edit `src/services/constants.ts`**:
   ```typescript
   export const SYNC_NOTES_INTERVAL_SEC = 60;  // Sync every 60 seconds
   ```

2. **For immediate sync on every change**:
   Modify `useNotesSync.ts` to sync in `useEffect` without debouncing.

#### Adding Real-time Sync

To implement real-time sync with Supabase subscriptions:

1. **Subscribe to changes** in `useNotesSync.ts`:
   ```typescript
   useEffect(() => {
     const subscription = supabase
       .from('notes')
       .on('INSERT', handleRemoteInsert)
       .on('UPDATE', handleRemoteUpdate)
       .on('DELETE', handleRemoteDelete)
       .subscribe();

     return () => subscription.unsubscribe();
   }, [bookHash]);
   ```

2. **Handle remote changes**:
   ```typescript
   const handleRemoteInsert = (payload) => {
     const newNote = payload.new;
     if (newNote.bookHash === bookHash) {
       // Merge into local notes
       setConfig(bookKey, {
         ...config,
         booknotes: [...config.booknotes, newNote]
       });
     }
   };
   ```

#### Sync Conflict Resolution

Current strategy: **Last Write Wins** (remote notes override local with same ID)

**Alternative strategies:**

1. **Manual Conflict Resolution**:
   ```typescript
   if (localNote.updatedAt > remoteNote.updatedAt) {
     // Keep local
   } else if (remoteNote.updatedAt > localNote.updatedAt) {
     // Keep remote
   } else {
     // Show conflict dialog to user
     showConflictDialog(localNote, remoteNote);
   }
   ```

2. **Merge Content**:
   ```typescript
   const mergedNote = {
     ...localNote,
     note: localNote.note + '\n---\n' + remoteNote.note,
     updatedAt: Date.now(),
   };
   ```

#### Offline Support

Current implementation handles offline gracefully:
- Notes saved locally when offline
- Sync resumes when connection restored
- Use `useSync()` hook's connection status

**Enhancement: Sync Queue**:
```typescript
// In useNotesSync.ts
const syncQueue = useRef<BookNote[]>([]);

const queueNoteForSync = (note: BookNote) => {
  syncQueue.current.push(note);
  if (navigator.onLine) {
    flushSyncQueue();
  }
};

window.addEventListener('online', flushSyncQueue);
```

### Common Issues

**Issue: Notes not syncing**
- Check user is authenticated (`useAuth()`)
- Verify `SYNC_NOTES_INTERVAL_SEC` constant
- Inspect browser console for API errors
- Check Supabase connection in `src/utils/supabase.ts`

**Issue: Duplicate notes after sync**
- Verify `id` field is unique per note
- Check merge logic in `useNotesSync.ts:64-68`
- Ensure `uniqueId()` generates globally unique IDs

**Issue: Deleted notes reappearing**
- Verify soft delete sets `deletedAt` timestamp
- Check sync includes deleted notes (`getNewNotes()` filters)
- Ensure cloud API respects `deletedAt` field

**Issue: Sync too frequent / too slow**
- Adjust `SYNC_NOTES_INTERVAL_SEC` in constants
- Check debouncing logic in `useNotesSync.ts:36-57`
- Monitor network tab for API call frequency

### Related Files

| File | Purpose |
|------|---------|
| `useNotesSync.ts:40` | Hook invocation in Annotator |
| `useNotesSync.ts:17` | Initial pull on mount |
| `useNotesSync.ts:42` | Periodic sync with debouncing |
| `useNotesSync.ts:60` | Merge remote notes into local |
| `useSync.ts:syncNotes` | Shared sync function |
| `api/sync.ts` | Cloud API endpoint |
| `constants.ts:SYNC_NOTES_INTERVAL_SEC` | Sync interval config |

---

## Text-to-Speech Integration (Added Jan 2025)

### Overview

The **Text-to-Speech (TTS) Integration** in the annotation system (commits 07b04b82, 74021412 - Jan 2025) allows users to listen to selected text using either Web Speech API or Microsoft Edge TTS service.

### Key Components

**Primary Files:**
- **`src/app/reader/components/annotator/Annotator.tsx:339-343,358`** - "Speak" button handler
- **`src/services/tts/TTSController.ts`** - TTS orchestration controller
- **`src/services/tts/WebSpeechClient.ts`** - Web Speech API client
- **`src/services/tts/EdgeTTSClient.ts`** - Edge TTS service client
- **`src/utils/event.ts`** - Event dispatcher for TTS commands
- **`src/utils/ssml.ts`** - SSML generation for TTS

### Annotation Toolbar Integration

**"Speak" Button** (8th tool in annotator toolbar):

```typescript
// From Annotator.tsx:358
{ tooltipText: _('Speak'), Icon: FaHeadphones, onClick: handleSpeakText }
```

**Handler** (Annotator.tsx:339-343):
```typescript
const handleSpeakText = async () => {
  if (!selection || !selection.text) return;
  setShowAnnotPopup(false);
  eventDispatcher.dispatch('tts-speak', { bookKey, range: selection.range });
};
```

**Flow:**
1. User **selects text** in book
2. Annotation popup appears with 8 tools
3. User clicks **"Speak"** button (headphones icon)
4. `handleSpeakText()` dispatches `'tts-speak'` event
5. TTS controller receives event and starts speaking

### TTS Architecture

**Two-tier TTS Backend:**

1. **Web Speech API** (browser native):
   - Free, built-in browser TTS
   - Lower quality voices
   - Limited voice options
   - Works offline

2. **Edge TTS** (Microsoft cloud service):
   - High-quality neural voices
   - Many voice options per language
   - Requires internet connection
   - Free (uses Microsoft Edge TTS API)

**Dynamic Backend Selection:**
- User selects voice in TTS panel
- Controller automatically switches backend based on voice
- Web Speech voices → WebSpeechClient
- Edge TTS voices → EdgeTTSClient

### Event-Driven Communication

**TTS Events** (via `eventDispatcher`):

| Event | Payload | Purpose |
|-------|---------|---------|
| `tts-speak` | `{ bookKey, range }` | Start speaking selected text |
| `tts-play` | - | Resume playback |
| `tts-pause` | - | Pause playback |
| `tts-stop` | - | Stop playback |

**Listening for Events:**
```typescript
// In TTSController or TTS components
eventDispatcher.on('tts-speak', handleTTSSpeak);
```

### Text Processing

**SSML Generation** (`src/utils/ssml.ts`):
- Converts plain text to SSML (Speech Synthesis Markup Language)
- Handles emphasis, pauses, pronunciation
- Example:
  ```xml
  <speak>
    <s>Hello, world!</s>
    <break time="500ms"/>
    <s>This is text to speech.</s>
  </speak>
  ```

### Integration with Reading Flow

**Coordinated with Reader:**
- TTS can speak selected text (from annotation)
- TTS can also speak entire sections (from reader controls)
- Both use same TTS infrastructure
- Selection-based speech is one-shot (doesn't continue to next section)

### Modification Guidelines for AI Agents

#### Adding Custom TTS Backend

To add a new TTS service (e.g., Google TTS):

1. **Create new client** `src/services/tts/GoogleTTSClient.ts`:
   ```typescript
   import { TTSClient } from './TTSClient';

   export class GoogleTTSClient implements TTSClient {
     async speak(text: string, voice: string): Promise<void> {
       // Implement Google TTS API call
     }

     async getVoices(): Promise<Voice[]> {
       // Fetch available voices
     }
   }
   ```

2. **Register in TTSController**:
   ```typescript
   // In TTSController.ts
   const googleClient = new GoogleTTSClient();

   const selectBackend = (voice: string) => {
     if (voice.startsWith('google-')) return googleClient;
     if (voice.startsWith('edge-')) return edgeClient;
     return webSpeechClient;
   };
   ```

3. **Add voice selection UI** in `TTSPanel.tsx`

#### Customizing Speak Button Behavior

To change what happens when "Speak" is clicked:

**Example: Speak and highlight simultaneously**
```typescript
const handleSpeakText = async () => {
  if (!selection || !selection.text) return;

  // Highlight the text
  handleHighlight(true);

  // Start speaking
  setShowAnnotPopup(false);
  eventDispatcher.dispatch('tts-speak', {
    bookKey,
    range: selection.range,
    highlightWhileSpeaking: true  // Custom option
  });
};
```

#### Adding Speak to Other Popups

To add "Speak" button to Wikipedia/Wiktionary popups:

1. **Edit popup component** (e.g., `WikipediaPopup.tsx`):
   ```typescript
   const handleSpeak = () => {
     eventDispatcher.dispatch('tts-speak', {
       bookKey,
       text: wikiContent  // Speak wiki content, not book text
     });
   };

   return (
     <Popup>
       {/* ... existing content ... */}
       <button onClick={handleSpeak}>
         <FaHeadphones /> Speak
       </button>
     </Popup>
   );
   ```

### Toolbar Button Order

**Current order** (Annotator.tsx:346-359):
1. **Copy** - Copy text to clipboard and notebook
2. **Highlight/Delete** - Toggle highlight on selected text
3. **Annotate** - Open note editor
4. **Search** - Search for text in book
5. **Dictionary** - Wiktionary lookup
6. **Wikipedia** - Wikipedia lookup
7. **Translate** - DeepL translation
8. **Speak** - Text-to-speech (NEW in Jan 2025)

**Modification:**
To change button order, reorder the `buttons` array in `Annotator.tsx:346-359`.

### Performance Considerations

**TTS-specific optimizations:**
- **Cache audio** for frequently read passages
- **Preload voices** on app startup
- **Debounce speak events** if user rapidly clicks
- **Cancel previous speech** when starting new

**Implementation:**
```typescript
let currentSpeech: Promise<void> | null = null;

const handleSpeakText = async () => {
  // Cancel ongoing speech
  if (currentSpeech) {
    eventDispatcher.dispatch('tts-stop');
  }

  currentSpeech = eventDispatcher.dispatch('tts-speak', {
    bookKey,
    range: selection.range
  });
};
```

### Common Issues

**Issue: "Speak" button not working**
- Check TTS controller is initialized
- Verify `eventDispatcher` is imported in Annotator
- Inspect browser console for TTS API errors
- Test if Web Speech API is supported (check browser)

**Issue: No voices available**
- Web Speech API may not support the language
- Edge TTS requires internet connection
- Check `TTSController.getVoices()` returns voices

**Issue: Poor voice quality**
- Switch to Edge TTS backend (select Edge voice)
- Adjust speech rate/pitch in TTS panel
- Some languages have limited Web Speech quality

**Issue: Speech interrupted when scrolling**
- TTS continues independently of scroll position
- Consider pausing TTS on scroll events
- Or implement visual indicator of speaking position

### Related Files

| File | Purpose |
|------|---------|
| `Annotator.tsx:339-343` | handleSpeakText handler |
| `Annotator.tsx:358` | Speak button definition |
| `TTSController.ts` | Main TTS orchestration |
| `WebSpeechClient.ts` | Browser TTS implementation |
| `EdgeTTSClient.ts` | Microsoft Edge TTS |
| `ssml.ts` | SSML generation utilities |
| `event.ts` | Event dispatcher for TTS |
