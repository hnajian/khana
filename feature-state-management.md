# Feature: State Management

## Overview

Readest uses **Zustand** for centralized state management, organizing application state into specialized stores. This architecture separates concerns and enables efficient reactivity across the UI. At commit `571baf98`, there are six primary stores managing different aspects of the application.

## Key Components

### Store Files

- **`src/store/readerStore.ts`** - Reader view states, progress tracking, multi-view management
- **`src/store/settingsStore.ts`** - System-wide settings and preferences
- **`src/store/libraryStore.ts`** - Book library catalog
- **`src/store/bookDataStore.ts`** - Loaded book data and configurations
- **`src/store/notebookStore.ts`** - Annotations, highlights, and notes
- **`src/store/sidebarStore.ts`** - Sidebar UI state and navigation

## Architecture

### Zustand Store Pattern

Readest uses Zustand's functional API:

```typescript
import { create } from 'zustand';

interface ExampleStore {
  value: number;
  setValue: (val: number) => void;
}

export const useExampleStore = create<ExampleStore>((set, get) => ({
  value: 0,
  setValue: (val) => set({ value: val }),
}));
```

**Usage in Components**:
```typescript
const value = useExampleStore(state => state.value);
const setValue = useExampleStore(state => state.setValue);
```

### Store Responsibilities

#### 1. Reader Store (`readerStore.ts`)

**Purpose**: Manages reader view instances, reading progress, and multi-book layout

**Key State**:
```typescript
interface ReaderStore {
  viewStates: { [key: string]: ViewState };  // Map of view states by key
  bookKeys: string[];                         // Keys of books in current grid
  hoveredBookKey: string | null;             // Currently hovered book
  // ... methods
}

interface ViewState {
  key: string;                    // Unique view key (format: "{bookHash}-{index}")
  view: FoliateView | null;       // Foliate-js view instance
  isPrimary: boolean;             // Is this the primary view for the book?
  loading: boolean;               // Loading state
  error: string | null;           // Error message if load failed
  progress: BookProgress | null;  // Current reading progress
  ribbonVisible: boolean;         // Bookmark ribbon visibility
  viewSettings: ViewSettings | null; // View-specific settings
}
```

**Key Methods**:
- `initViewState()`: Initialize a new view for a book
- `clearViewState()`: Remove a view
- `setProgress()`: Update reading progress
- `setView()`: Set foliate-js view instance
- `getViews()`: Get all active views
- `getViewsById()`: Get all views for a specific book (multi-view support)

**Multi-View Support**:
- When a book is opened multiple times (grid layout), each gets a unique view
- View keys follow pattern: `{bookHash}-{index}`
- Primary view persists settings and progress to book config
- Non-primary views are memory-only

#### 2. Settings Store (`settingsStore.ts`)

**Purpose**: Global application settings

**Key State**:
```typescript
interface SettingsState {
  settings: SystemSettings;                      // Global settings object
  isFontLayoutSettingsDialogOpen: boolean;       // Settings dialog visibility
  isFontLayoutSettingsGlobal: boolean;           // Is dialog in global mode?
  setSettings: (settings: SystemSettings) => void;
  saveSettings: (envConfig: EnvConfigType, settings: SystemSettings) => void;
  // ...
}
```

**Persistence**:
- Calls `appService.saveSettings()` to persist changes
- Loaded on app initialization

#### 3. Library Store (`libraryStore.ts`)

**Purpose**: Manages the book collection

**Key State**:
```typescript
interface LibraryStore {
  library: BookMetadata[];        // Array of all books
  addBook: (book: BookMetadata) => void;
  removeBook: (hash: string) => void;
  updateBook: (hash: string, updates: Partial<BookMetadata>) => void;
  // ...
}
```

**Book Metadata**:
```typescript
interface BookMetadata {
  hash: string;           // MD5 hash for unique identification
  title: string;
  author: string;
  cover?: string;         // Base64 or blob URL
  format: BookFormat;
  // ...
}
```

#### 4. Book Data Store (`bookDataStore.ts`)

**Purpose**: Stores loaded book documents and configs

**Key State**:
```typescript
interface BookDataStore {
  booksData: {
    [bookHash: string]: {
      id: string;               // Book hash
      book: BookMetadata;       // Metadata
      file: File;               // Book file
      config: BookConfig;       // Reading config (progress, settings)
      bookDoc: BookDoc;         // Parsed document from foliate-js
    }
  };
}
```

**Behavior**:
- Books are loaded on-demand when opened in reader
- Cached in memory during session
- Config changes trigger updates to this store
- Data persists across view changes but not app restarts

#### 5. Notebook Store (`notebookStore.ts`)

**Purpose**: Manages annotations, highlights, and notes

**Key State**:
```typescript
interface NotebookStore {
  booknotes: {
    [bookHash: string]: {
      highlights: Annotation[];
      notes: Note[];
    }
  };
  // ...
}
```

**Persistence**:
- Saved as part of book config
- Updated when annotations are added/deleted/modified

#### 6. Sidebar Store (`sidebarStore.ts`)

**Purpose**: UI state for sidebar panel

**Key State**:
```typescript
interface SidebarStore {
  isVisible: boolean;
  activeTab: 'toc' | 'search' | 'bookmarks' | 'booknotes';
  searchQuery: string;
  searchResults: SearchResult[];
  // ...
}
```

### Store Interactions

**Common Patterns**:

1. **Reader → BookData**:
   - Reader initializes view → loads book data → caches in BookDataStore

2. **Reader → Settings**:
   - Reader applies settings from SettingsStore (global) + BookDataStore (book-specific)

3. **Notebook → BookData**:
   - Annotations saved → Updates NotebookStore → Triggers BookDataStore config update

4. **Library → Reader**:
   - User selects book → Library provides metadata → Reader loads full book data

## AI Agent Modification Guidelines

### Creating a New Store

To add a new store (e.g., for reading statistics):

1. **Create store file** (`src/store/statisticsStore.ts`):
   ```typescript
   import { create } from 'zustand';

   interface ReadingStats {
     pagesRead: number;
     timeSpent: number;
     booksFinished: number;
   }

   interface StatisticsStore {
     stats: ReadingStats;
     updateStats: (update: Partial<ReadingStats>) => void;
     resetStats: () => void;
   }

   export const useStatisticsStore = create<StatisticsStore>((set) => ({
     stats: {
       pagesRead: 0,
       timeSpent: 0,
       booksFinished: 0,
     },
     updateStats: (update) =>
       set((state) => ({
         stats: { ...state.stats, ...update },
       })),
     resetStats: () =>
       set({
         stats: { pagesRead: 0, timeSpent: 0, booksFinished: 0 },
       }),
   }));
   ```

2. **Use in components**:
   ```typescript
   import { useStatisticsStore } from '@/store/statisticsStore';

   function StatisticsPanel() {
     const stats = useStatisticsStore(state => state.stats);
     return <div>Pages Read: {stats.pagesRead}</div>;
   }
   ```

3. **Add persistence** (if needed):
   - Integrate with `appService.saveSettings()`
   - Load on app start
   - Save on changes (debounced)

### Optimizing Store Selectors

To prevent unnecessary re-renders:

1. **Use specific selectors**:
   ```typescript
   // Bad: Re-renders on any store change
   const store = useReaderStore();

   // Good: Only re-renders when progress changes
   const progress = useReaderStore(state => state.getProgress(bookKey));
   ```

2. **Use shallow equality** for objects:
   ```typescript
   import { shallow } from 'zustand/shallow';

   const { bookKeys, hoveredBookKey } = useReaderStore(
     state => ({ bookKeys: state.bookKeys, hoveredBookKey: state.hoveredBookKey }),
     shallow
   );
   ```

3. **Memoize computed values**:
   ```typescript
   const activeViews = useReaderStore(
     state => Object.values(state.viewStates).filter(v => !v.loading),
     shallow
   );
   ```

### Adding Store Middleware

To add logging, persistence, or other middleware:

1. **Persist middleware** (example):
   ```typescript
   import { create } from 'zustand';
   import { persist } from 'zustand/middleware';

   export const usePersistedStore = create(
     persist(
       (set) => ({
         value: 0,
         setValue: (val) => set({ value: val }),
       }),
       {
         name: 'persisted-store',
         storage: createJSONStorage(() => localStorage),
       }
     )
   );
   ```

2. **Logging middleware**:
   ```typescript
   const logMiddleware = (config) => (set, get, api) =>
     config(
       (...args) => {
         console.log('Before:', get());
         set(...args);
         console.log('After:', get());
       },
       get,
       api
     );

   export const useLoggedStore = create(logMiddleware((set) => ({ /* ... */ })));
   ```

### Debugging Store State

To inspect store state during development:

1. **Add to window** (dev only):
   ```typescript
   // In store file
   if (typeof window !== 'undefined' && process.env.NODE_ENV === 'development') {
     window.__READEST_STORES__ = {
       reader: useReaderStore.getState,
       settings: useSettingsStore.getState,
       // ... other stores
     };
   }
   ```

2. **Use Zustand DevTools**:
   ```typescript
   import { devtools } from 'zustand/middleware';

   export const useReaderStore = create(
     devtools(
       (set) => ({ /* ... */ }),
       { name: 'ReaderStore' }
     )
   );
   ```

### Handling Async Operations

To manage async operations in stores:

1. **Pattern for loading states**:
   ```typescript
   interface AsyncStore {
     data: any | null;
     loading: boolean;
     error: string | null;
     fetchData: () => Promise<void>;
   }

   export const useAsyncStore = create<AsyncStore>((set) => ({
     data: null,
     loading: false,
     error: null,
     fetchData: async () => {
       set({ loading: true, error: null });
       try {
         const response = await fetch('/api/data');
         const data = await response.json();
         set({ data, loading: false });
       } catch (error) {
         set({ error: error.message, loading: false });
       }
     },
   }));
   ```

2. **Use in components**:
   ```typescript
   const { data, loading, error, fetchData } = useAsyncStore();

   useEffect(() => {
     fetchData();
   }, []);

   if (loading) return <Spinner />;
   if (error) return <div>Error: {error}</div>;
   return <div>{JSON.stringify(data)}</div>;
   ```

### Store Communication

To coordinate between stores:

1. **Direct access** (not recommended for frequent use):
   ```typescript
   // In readerStore
   import { useSettingsStore } from './settingsStore';

   const settings = useSettingsStore.getState().settings;
   ```

2. **Event-based** (preferred):
   - Use custom event dispatcher (`src/utils/event.ts`)
   - Subscribe to events in stores
   - Emit events from actions

3. **Shared context**:
   - Pass `envConfig` to store methods
   - Access `appService` for cross-cutting concerns

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add new store | New store file | Components using the store |
| Modify reader state | `readerStore.ts` | `FoliateViewer.tsx`, reader components |
| Change settings | `settingsStore.ts` | `appService.ts`, settings panels |
| Update library | `libraryStore.ts` | `appService.ts`, library components |
| Modify annotations | `notebookStore.ts` | Annotator components |
| Add persistence | Store file | `appService.ts`, middleware |
| Optimize selectors | Component files | - |

## Common Issues and Debugging

### Problem: Component not re-rendering on state change

- Verify selector is correctly extracting the changed value
- Check if comparison is using reference equality (use `shallow` for objects)
- Ensure `set()` is being called in store action
- Check if component is using the selector correctly

### Problem: Stale state in callbacks

- Use `get()` inside actions instead of capturing in closure:
  ```typescript
  // Bad
  const value = get().value;
  setTimeout(() => console.log(value), 1000); // Stale

  // Good
  setTimeout(() => console.log(get().value), 1000); // Fresh
  ```

### Problem: State not persisting

- Check if `saveSettings()` or equivalent is called
- Verify `appService` is writing data correctly
- Check for errors in save operations
- Ensure store is loaded on app initialization

### Problem: Circular dependencies between stores

- Refactor to use event-based communication
- Move shared logic to utility functions
- Use `getState()` for one-way dependencies only

## Dependencies

- **zustand**: Core state management library
- **appService**: For persistence of settings and configs

## Performance Considerations

- **Use specific selectors**: Only subscribe to needed state slices
- **Shallow equality**: Use for object/array selectors
- **Debounce saves**: Don't save on every state change
- **Lazy initialization**: Only load stores when needed
- **Memoize computations**: Use `useMemo` in components, not stores
