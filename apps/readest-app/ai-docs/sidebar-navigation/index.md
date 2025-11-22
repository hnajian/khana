# Feature: Sidebar Navigation

## Overview

The Sidebar provides navigation and content discovery features including Table of Contents (TOC), full-text search, bookmarks, and booknotes (highlights and annotations). It's a collapsible panel on the reader page that helps users navigate and explore books.

## Key Components

### Primary Files

- **`src/app/reader/components/sidebar/SideBar.tsx`** - Main sidebar container
- **`src/app/reader/components/sidebar/TOCView.tsx`** - Table of contents tree view
- **`src/app/reader/components/sidebar/SearchResults.tsx`** - Full-text search results
- **`src/app/reader/components/sidebar/BooknoteView.tsx`** - List of highlights and notes
- **`src/app/reader/components/sidebar/TabNavigation.tsx`** - Tab switcher for sidebar views

### Related Files

- **`src/app/reader/components/sidebar/Header.tsx`** - Sidebar header with close button
- **`src/app/reader/components/sidebar/SearchBar.tsx`** - Search input component
- **`src/app/reader/components/sidebar/SearchOptions.tsx`** - Search configuration options
- **`src/app/reader/components/sidebar/BookCard.tsx`** - Book metadata display
- **`src/app/reader/components/sidebar/BookMenu.tsx`** - Book actions menu
- **`src/app/reader/components/sidebar/BooknoteItem.tsx`** - Individual highlight/note item
- **`src/app/reader/components/sidebar/Content.tsx`** - Tab content container
- **`src/app/reader/components/SidebarToggler.tsx`** - Button to show/hide sidebar
- **`src/app/reader/hooks/useSidebar.ts`** - Sidebar state and logic hook
- **`src/store/sidebarStore.ts`** - Sidebar state management

## Architecture

### Sidebar Tabs

The sidebar has multiple tabs for different navigation modes:

1. **TOC (Table of Contents)**: Hierarchical book structure
2. **Search**: Full-text search within the book
3. **Bookmarks**: Saved reading positions
4. **Booknotes**: Highlights and annotations

### State Management

**Sidebar Store** (`src/store/sidebarStore.ts`):
```typescript
interface SidebarStore {
  isVisible: boolean;              // Sidebar visibility
  activeTab: 'toc' | 'search' | 'bookmarks' | 'booknotes';
  searchQuery: string;
  searchResults: SearchResult[];
  setActiveTab: (tab: string) => void;
  toggleSidebar: () => void;
  // ...
}
```

### TOC (Table of Contents)

**Data Structure**:
```typescript
interface TOCItem {
  id: number;
  label: string;       // Chapter/section title
  href: string;        // Link to location in book
  subitems?: TOCItem[]; // Nested subsections
}
```

**Implementation** (`TOCView.tsx`):
- Renders recursive tree structure from `bookDoc.toc`
- Clicking a TOC item navigates the reader view to that location
- Currently reading section is highlighted
- Uses `useFoliateEvents` to track reading progress and update active item

### Full-Text Search

**Search Flow**:
1. User enters query in `SearchBar.tsx`
2. Query sent to foliate-js search API
3. Results returned with context snippets
4. `SearchResults.tsx` displays matches with surrounding text
5. Clicking a result navigates to that location and highlights match

**Search Options** (`SearchOptions.tsx`):
- Case sensitive
- Whole word only
- Regular expression mode

**Implementation Details**:
- Search is performed by foliate-js iframe
- Results include CFI locations for navigation
- Match highlighting uses same system as annotations

### Booknotes View

**Features**:
- Lists all highlights and notes for current book
- Grouped by color or creation date
- Click to navigate to annotation location
- Edit or delete notes inline

**Implementation** (`BooknoteView.tsx`, `BooknoteItem.tsx`):
- Reads from `notebookStore.booknotes[bookHash]`
- Displays `highlights` and `notes` arrays
- Uses `BooknoteItem` component for each entry
- Navigation via `view.goTo(cfi)`

### Bookmarks View

**Features**:
- List of saved reading positions
- Add/remove bookmark at current location
- Click to jump to bookmarked position

**Implementation**:
- Bookmarks stored in book config
- Ribbon indicator shows bookmark status
- `BookmarkToggler` component in header bar

## AI Agent Modification Guidelines

### Adding a New Sidebar Tab

To add a new tab (e.g., "Statistics" showing reading stats):

1. **Update `sidebarStore.ts`**:
   ```typescript
   type TabType = 'toc' | 'search' | 'bookmarks' | 'booknotes' | 'statistics';
   ```

2. **Create tab content component**:
   ```typescript
   // src/app/reader/components/sidebar/StatisticsView.tsx
   export function StatisticsView({ bookKey }: { bookKey: string }) {
     // Implement statistics display
     return <div>Reading statistics...</div>;
   }
   ```

3. **Update `TabNavigation.tsx`**:
   ```typescript
   <button onClick={() => setActiveTab('statistics')}>
     Statistics
   </button>
   ```

4. **Update `Content.tsx`**:
   ```typescript
   {activeTab === 'statistics' && <StatisticsView bookKey={bookKey} />}
   ```

### Enhancing TOC Display

To improve TOC visualization:

1. **Add depth indicators** in `TOCView.tsx`:
   ```typescript
   <div style={{ paddingLeft: `${depth * 16}px` }}>
     {item.label}
   </div>
   ```

2. **Add expand/collapse** for nested items:
   - Add state to track expanded items
   - Show/hide `subitems` based on expansion state
   - Add toggle icons (chevron up/down)

3. **Add progress indicators**:
   - Show read percentage for each section
   - Use progress data from `readerStore`
   - Display as progress bar or badge

### Improving Search Functionality

To enhance search capabilities:

1. **Add search history**:
   ```typescript
   // In sidebarStore.ts
   searchHistory: string[];
   addToHistory: (query: string) => {
     set(state => ({
       searchHistory: [query, ...state.searchHistory].slice(0, 10)
     }));
   }
   ```

2. **Add search filters**:
   - Filter by section/chapter
   - Filter by annotation color
   - Filter by date range

3. **Improve result display**:
   - Add context expansion (show more surrounding text)
   - Highlight search term in context
   - Group results by chapter

4. **Add search within notes**:
   - Search through annotations and notes
   - Combine book text search with notes search

### Customizing Booknote Display

To change how highlights and notes are displayed:

1. **Add grouping options** in `BooknoteView.tsx`:
   ```typescript
   const [groupBy, setGroupBy] = useState<'color' | 'date' | 'chapter'>('color');

   const groupedNotes = useMemo(() => {
     if (groupBy === 'color') return groupByColor(booknotes);
     if (groupBy === 'date') return groupByDate(booknotes);
     return groupByChapter(booknotes);
   }, [booknotes, groupBy]);
   ```

2. **Add sorting**:
   - Sort by creation date (newest/oldest first)
   - Sort by location in book
   - Sort by highlight color

3. **Add filtering**:
   - Filter by color
   - Filter by has note / no note
   - Filter by date range

### Adding Bookmark Features

To enhance bookmarking:

1. **Add bookmark labels**:
   ```typescript
   interface Bookmark {
     cfi: string;
     label: string;        // User-defined label
     created: number;
     color?: string;       // Optional color coding
   }
   ```

2. **Add bookmark organization**:
   - Folders/categories for bookmarks
   - Tags for bookmarks
   - Search within bookmarks

3. **Add bookmark export**:
   - Export as JSON
   - Export as text with quotes
   - Share bookmarks

### Improving Sidebar Responsiveness

To make sidebar work better on different screen sizes:

1. **Update `SideBar.tsx` styles**:
   - Use CSS media queries for width
   - Add swipe gesture to close on mobile
   - Auto-hide on small screens

2. **Add resize handle**:
   - Implement draggable divider
   - Store user's preferred width in settings
   - Use `useDragBar` hook pattern

3. **Optimize for mobile**:
   - Full-screen sidebar on mobile
   - Bottom sheet style presentation
   - Touch-optimized tap targets

### Tracking Active Section in TOC

To improve "current section" highlighting:

1. **In `TOCView.tsx`**:
   ```typescript
   const currentTocId = useReaderStore(state =>
     state.getProgress(bookKey)?.sectionId
   );

   const isActive = (item: TOCItem) => item.id === currentTocId;
   ```

2. **Auto-scroll to active item**:
   - Use `useScrollToItem` hook
   - Scroll when progress changes
   - Smooth scroll animation

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add sidebar tab | `sidebarStore.ts`, `TabNavigation.tsx`, `Content.tsx` | New view component |
| Enhance TOC | `TOCView.tsx` | `readerStore.ts` |
| Improve search | `SearchBar.tsx`, `SearchResults.tsx` | `SearchOptions.tsx`, `sidebarStore.ts` |
| Customize booknotes | `BooknoteView.tsx`, `BooknoteItem.tsx` | `notebookStore.ts` |
| Add bookmark features | Bookmark view component | `bookDataStore.ts` |
| Resize sidebar | `SideBar.tsx` | `useDragBar.ts`, `settingsStore.ts` |
| Fix navigation | `useSidebar.ts` | `readerStore.ts`, foliate-js |

## Common Issues and Debugging

### Problem: TOC not updating to show current section

- Check if `useFoliateEvents` is emitting progress updates
- Verify `setProgress` in `readerStore` updates `sectionId`
- Check `TOCView` is reading correct progress from store
- Ensure TOC items have unique IDs from `updateTocID()`

### Problem: Search not finding text

- Verify search is being sent to foliate-js view
- Check if text is searchable (e.g., scanned PDFs with no OCR won't work)
- Test if regex mode is causing issues
- Check case sensitivity settings

### Problem: Sidebar not closing on mobile

- Verify `toggleSidebar()` is being called
- Check CSS transitions are not broken
- Test touch event handlers
- Verify z-index isn't blocking click

### Problem: Booknotes not appearing

- Check `notebookStore.booknotes[bookHash]` contains data
- Verify book hash is correct and consistent
- Check if annotations are being saved properly
- Inspect filter/grouping logic isn't hiding notes

## Dependencies

- **foliate-js**: Provides TOC data and search API
- **zustand**: State management for sidebar
- **react-icons**: Icons for tabs and buttons
- **useScrollToItem hook**: Auto-scroll to active TOC item

## Performance Considerations

- **Virtualize long lists**: For books with many TOC items or search results, use virtual scrolling
- **Debounce search**: Don't search on every keystroke, wait for pause
- **Lazy load tabs**: Only render active tab content
- **Memoize grouping**: Use `useMemo` for booknote grouping/sorting
- **Optimize re-renders**: Use React.memo for list items
