# Sub-Feature: Bookshelf Views and Sorting

## Overview

The Bookshelf feature provides multiple view modes (grid and list) and sorting options for organizing the book library. Added in v0.9.37 (commit #955) and enhanced in v0.9.35 (commit #887), it enables users to browse and manage their collection efficiently.

## Key Components

### Primary Files

- **`src/app/library/components/Bookshelf.tsx`** - Main bookshelf component with view rendering
- **`src/app/library/components/BookCard.tsx`** - Individual book card for grid view
- **`src/app/library/components/BookListItem.tsx`** - Book row for list view
- **`src/store/libraryStore.ts`** - View mode and sorting state management

### Related Files

- **`src/app/library/components/LibraryHeader.tsx`** - View mode and sort controls
- **`src/utils/book.ts`** - Book sorting utility functions
- **`src/types/book.ts`** - BookMetadata type definition

## Architecture

### View Modes

**Grid View** (Default):
- **Layout**: CSS Grid with responsive columns
- **Card Design**: Cover image + title + author
- **Columns**:
  - Mobile: 2-3 columns
  - Tablet: 4-6 columns
  - Desktop: 6-8 columns
- **Features**:
  - Cover image prominent
  - Hover effects
  - Quick actions menu

**List View** (commit ea8ab91e):
- **Layout**: Vertical list with rows
- **Row Design**: Compact with cover thumbnail + metadata
- **Features**:
  - More metadata visible (file size, format, last opened)
  - Better for text search
  - Efficient for large libraries (100+ books)
  - Sortable columns

### Sorting Options

**Available Sort Methods** (commit 59f875f5):

1. **Recent** (Default)
   - Sorts by `lastOpened` timestamp (most recent first)
   - Falls back to `addedDate` if never opened
   - Best for resuming reading

2. **Title** (A-Z)
   - Alphabetical by book title
   - Case-insensitive sorting
   - Handles special characters and numbers
   - Best for finding specific books

3. **Author** (A-Z)
   - Alphabetical by author name
   - Groups books by the same author
   - Best for browsing by author

### State Management

**Library Store** (`src/store/libraryStore.ts`):

```typescript
interface LibraryStore {
  // ... other state
  viewMode: 'grid' | 'list';        // Current view mode
  sortBy: 'recent' | 'title' | 'author'; // Sort method
  setViewMode: (mode: 'grid' | 'list') => void;
  setSortBy: (method: string) => void;
  getSortedBooks: () => BookMetadata[];
}
```

**Sorted Books Computation**:
```typescript
getSortedBooks: () => {
  const { library, sortBy } = get();
  const books = [...library];

  switch (sortBy) {
    case 'title':
      return books.sort((a, b) =>
        a.title.localeCompare(b.title, undefined, {
          sensitivity: 'base'
        })
      );

    case 'author':
      return books.sort((a, b) =>
        (a.author || '').localeCompare(b.author || '', undefined, {
          sensitivity: 'base'
        })
      );

    case 'recent':
    default:
      return books.sort((a, b) => {
        const aTime = a.lastOpened || a.addedDate;
        const bTime = b.lastOpened || b.addedDate;
        return bTime - aTime; // Most recent first
      });
  }
}
```

### UI Components

**View Mode Switcher** (in `LibraryHeader.tsx`):
```typescript
<div className="view-mode-buttons">
  <button
    className={viewMode === 'grid' ? 'active' : ''}
    onClick={() => setViewMode('grid')}
    aria-label="Grid view"
  >
    <BsGrid3X3Gap />
  </button>
  <button
    className={viewMode === 'list' ? 'active' : ''}
    onClick={() => setViewMode('list')}
    aria-label="List view"
  >
    <BsList />
  </button>
</div>
```

**Sort Dropdown**:
```typescript
<select
  value={sortBy}
  onChange={(e) => setSortBy(e.target.value)}
  aria-label="Sort books"
>
  <option value="recent">Recently Opened</option>
  <option value="title">Title (A-Z)</option>
  <option value="author">Author (A-Z)</option>
</select>
```

## AI Agent Modification Guidelines

### Adding New Sort Methods

To add custom sorting options (e.g., by file size, format):

1. **Update library store**:
   ```typescript
   type SortMethod = 'recent' | 'title' | 'author' | 'size' | 'format';

   getSortedBooks: () => {
     const { library, sortBy } = get();
     const books = [...library];

     switch (sortBy) {
       // ... existing cases

       case 'size':
         return books.sort((a, b) => {
           const aSize = a.fileSize || 0;
           const bSize = b.fileSize || 0;
           return bSize - aSize; // Largest first
         });

       case 'format':
         return books.sort((a, b) =>
           a.format.localeCompare(b.format)
         );

       default:
         return books;
     }
   }
   ```

2. **Add to UI**:
   ```typescript
   <option value="size">File Size</option>
   <option value="format">Format</option>
   ```

### Implementing Grid Column Customization

To allow users to adjust grid columns:

1. **Add column setting to store**:
   ```typescript
   interface LibraryStore {
     // ... existing
     gridColumns: number; // 2-10
     setGridColumns: (columns: number) => void;
   }
   ```

2. **Apply to grid CSS**:
   ```typescript
   <div
     className="bookshelf-grid"
     style={{
       gridTemplateColumns: `repeat(${gridColumns}, 1fr)`
     }}
   >
     {books.map(book => <BookCard key={book.hash} book={book} />)}
   </div>
   ```

3. **Add UI slider**:
   ```typescript
   <label>
     Grid Columns: {gridColumns}
     <input
       type="range"
       min="2"
       max="10"
       value={gridColumns}
       onChange={(e) => setGridColumns(Number(e.target.value))}
     />
   </label>
   ```

### Adding List View Column Customization

To allow users to show/hide list columns:

1. **Define visible columns state**:
   ```typescript
   interface LibraryStore {
     // ... existing
     listColumns: {
       title: boolean;
       author: boolean;
       format: boolean;
       fileSize: boolean;
       lastOpened: boolean;
       progress: boolean;
     };
     setListColumns: (columns: Record<string, boolean>) => void;
   }
   ```

2. **Render only visible columns**:
   ```typescript
   <tr>
     {listColumns.title && <td>{book.title}</td>}
     {listColumns.author && <td>{book.author}</td>}
     {listColumns.format && <td>{book.format}</td>}
     {listColumns.fileSize && <td>{formatFileSize(book.fileSize)}</td>}
     {listColumns.lastOpened && <td>{formatDate(book.lastOpened)}</td>}
     {listColumns.progress && <td>{book.progress}%</td>}
   </tr>
   ```

3. **Add column visibility controls**:
   ```typescript
   <div className="column-visibility">
     <label>
       <input
         type="checkbox"
         checked={listColumns.fileSize}
         onChange={(e) => setListColumns({
           ...listColumns,
           fileSize: e.target.checked
         })}
       />
       File Size
     </label>
     {/* Repeat for other columns */}
   </div>
   ```

### Implementing Virtual Scrolling

For large libraries (1000+ books):

1. **Install react-window**:
   ```bash
   pnpm add react-window
   ```

2. **Implement virtual grid**:
   ```typescript
   import { FixedSizeGrid } from 'react-window';

   const VirtualBookshelf = ({ books }: { books: BookMetadata[] }) => {
     const columnCount = 6;
     const rowCount = Math.ceil(books.length / columnCount);

     const Cell = ({ columnIndex, rowIndex, style }) => {
       const index = rowIndex * columnCount + columnIndex;
       const book = books[index];

       if (!book) return null;

       return (
         <div style={style}>
           <BookCard book={book} />
         </div>
       );
     };

     return (
       <FixedSizeGrid
         columnCount={columnCount}
         columnWidth={200}
         height={window.innerHeight - 200}
         rowCount={rowCount}
         rowHeight={300}
         width={window.innerWidth}
       >
         {Cell}
       </FixedSizeGrid>
     );
   };
   ```

### Adding Grouped View

To group books by author, format, or custom tags:

1. **Compute grouped books**:
   ```typescript
   getGroupedBooks: (groupBy: 'author' | 'format' | 'tag') => {
     const { library } = get();
     const groups: Record<string, BookMetadata[]> = {};

     library.forEach(book => {
       const key = groupBy === 'author'
         ? book.author
         : groupBy === 'format'
         ? book.format
         : book.tags?.[0] || 'Untagged';

       if (!groups[key]) groups[key] = [];
       groups[key].push(book);
     });

     return groups;
   }
   ```

2. **Render grouped view**:
   ```typescript
   const groupedBooks = libraryStore.getGroupedBooks('author');

   return (
     <div className="grouped-bookshelf">
       {Object.entries(groupedBooks).map(([group, books]) => (
         <div key={group} className="book-group">
           <h3>{group}</h3>
           <div className="books-grid">
             {books.map(book => <BookCard key={book.hash} book={book} />)}
           </div>
         </div>
       ))}
     </div>
   );
   ```

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add sort method | `libraryStore.ts` | `LibraryHeader.tsx` |
| Customize grid layout | `Bookshelf.tsx` | `BookCard.tsx` |
| Add list columns | `BookListItem.tsx` | `libraryStore.ts` |
| Implement virtual scroll | `Bookshelf.tsx` | Install react-window |
| Add grouped view | `libraryStore.ts` | `Bookshelf.tsx` |
| Customize view UI | `LibraryHeader.tsx` | `Bookshelf.tsx` |

## Common Issues and Debugging

### Problem: Sort not working

- Check if `getSortedBooks()` is being called
- Verify sort method is valid in store
- Inspect book metadata has required fields (title, author, etc.)
- Check console for sorting errors

### Problem: View mode not switching

- Verify `setViewMode()` updates store state
- Check if CSS for both views is loaded
- Ensure components re-render on state change
- Inspect React DevTools for state updates

### Problem: Grid columns not responsive

- Check CSS grid breakpoints
- Verify responsive column calculation
- Test on different screen sizes
- Inspect CSS with DevTools

### Problem: List view slow with many books

- Implement virtual scrolling for large libraries
- Check if re-renders are excessive
- Verify memo/useMemo is used for expensive computations
- Profile with React DevTools Profiler

## Dependencies

- **React Icons**: BsGrid3X3Gap, BsList for view mode icons
- **CSS Grid**: Grid layout for bookshelf
- **Zustand**: State management for view mode and sort
- **react-window** (optional): Virtual scrolling for large libraries

## Performance Considerations

- **Virtual scrolling**: Essential for libraries with 500+ books
- **Memoization**: Use `useMemo` for sorted books array
- **Lazy loading**: Load covers on demand with IntersectionObserver
- **Debounce sort**: Don't re-sort on every keystroke in search
- **CSS containment**: Use `contain: layout` for book cards

## Accessibility

- **Keyboard navigation**: Arrow keys to navigate books
- **Screen reader labels**: Proper ARIA labels for controls
- **Focus indicators**: Visible focus states for keyboard users
- **View mode announcements**: Announce view mode changes

---

**Implemented in commits:**
- ea8ab91e: feat: add list view for the bookshelf, closes #542
- 59f875f5: feat: add books sorting by title and author in the library page, closes #540
