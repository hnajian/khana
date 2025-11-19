# Feature: Library Management

## Overview

The Library Management feature handles the book collection, including import, organization, metadata management, and cover image generation. At commit `571baf98`, the library provides a grid-based bookshelf view with book metadata cards.

## Key Components

### Primary Files

- **`src/app/library/page.tsx`** - Library page component
- **`src/app/library/components/Bookshelf.tsx`** - Grid display of books
- **`src/app/library/components/LibraryHeader.tsx`** - Header with import and actions
- **`src/store/libraryStore.ts`** - Library state management
- **`src/services/appService.ts`** - Book import/export/storage methods

### Related Files

- **`src/types/book.ts`** - `BookMetadata` type definition
- **`src/utils/book.ts`** - Book utility functions
- **`src/utils/md5.ts`** - Book hash generation
- **`src/services/nativeAppService.ts`** - Native file system operations

## Architecture

### Book Metadata Structure

```typescript
interface BookMetadata {
  hash: string;           // MD5 hash (unique identifier)
  title: string;          // Book title
  author: string;         // Author name
  language: string;       // Language code
  publisher?: string;     // Publisher
  published?: string;     // Publication date
  format: BookFormat;     // EPUB, PDF, MOBI, etc.
  coverImageUrl?: string; // Cover image URL or data URI
  addedDate: number;      // Timestamp when added
  lastOpened?: number;    // Last opened timestamp
  tags?: string[];        // User-defined tags
  // ...
}
```

### Library Store

**State** (`src/store/libraryStore.ts`):
```typescript
interface LibraryStore {
  library: BookMetadata[];               // All books
  selectedBooks: string[];               // Selected book hashes
  viewMode: 'grid' | 'list';            // View mode (grid implemented)
  sortBy: 'title' | 'author' | 'recent'; // Sort order
  addBook: (book: BookMetadata) => void;
  removeBook: (hash: string) => void;
  updateBook: (hash: string, updates: Partial<BookMetadata>) => void;
  // ...
}
```

### Book Import Flow

1. **User triggers import**:
   - Click import button in `LibraryHeader`
   - File picker dialog opens (Tauri dialog API)

2. **File processing**:
   - `appService.importBooks(files)` is called
   - For each file:
     - Calculate MD5 hash
     - Load using `DocumentLoader`
     - Extract metadata from `BookDoc`
     - Generate cover image
     - Copy file to library storage

3. **Metadata extraction**:
   - Title, author, language from `bookDoc.metadata`
   - Cover from `bookDoc.getCover()`
   - Format from `DocumentLoader` detection

4. **Storage**:
   - Book file saved to app storage directory
   - Metadata added to library catalog
   - Cover image cached

5. **UI update**:
   - `libraryStore.addBook(metadata)` called
   - Bookshelf re-renders with new book

### Cover Image Handling

**Generation** (in `appService.ts`):
```typescript
async generateCover(bookDoc: BookDoc): Promise<string> {
  const coverBlob = await bookDoc.getCover();
  if (!coverBlob) return DEFAULT_COVER;

  // Convert to data URI or save to cache
  const url = URL.createObjectURL(coverBlob);
  return url;
}
```

**Caching**:
- Covers stored in app cache directory
- Indexed by book hash
- Loaded lazily in bookshelf

**Fallback**:
- If no cover available, show default placeholder
- Placeholder can include first letter of title

### Book Organization

**Current Implementation**:
- Flat list of all books
- Sorting by title, author, or recent
- No folders/categories (as of this commit)

**Future Expansion Points**:
- Collections/shelves
- Tags/labels
- Search/filter
- Custom sorting

## AI Agent Modification Guidelines

### Adding Book Search/Filter

To implement library search:

1. **Add search state** to `libraryStore.ts`:
   ```typescript
   interface LibraryStore {
     // ...
     searchQuery: string;
     setSearchQuery: (query: string) => void;
     getFilteredBooks: () => BookMetadata[];
   }

   // In store implementation
   setSearchQuery: (query) => set({ searchQuery: query }),
   getFilteredBooks: () => {
     const { library, searchQuery } = get();
     if (!searchQuery) return library;

     const lowerQuery = searchQuery.toLowerCase();
     return library.filter(book =>
       book.title.toLowerCase().includes(lowerQuery) ||
       book.author.toLowerCase().includes(lowerQuery)
     );
   }
   ```

2. **Add search UI** in `LibraryHeader.tsx`:
   ```typescript
   <input
     type="text"
     placeholder="Search books..."
     value={searchQuery}
     onChange={(e) => setSearchQuery(e.target.value)}
   />
   ```

3. **Update bookshelf** to use filtered books:
   ```typescript
   const books = useLibraryStore(state => state.getFilteredBooks());
   ```

### Implementing Collections/Shelves

To add book collections:

1. **Extend library store**:
   ```typescript
   interface Collection {
     id: string;
     name: string;
     bookHashes: string[];
     color?: string;
   }

   interface LibraryStore {
     // ...
     collections: Collection[];
     addCollection: (name: string) => void;
     addBookToCollection: (bookHash: string, collectionId: string) => void;
     removeBookFromCollection: (bookHash: string, collectionId: string) => void;
   }
   ```

2. **Add UI for collections**:
   - Sidebar with collection list
   - Drag-and-drop books to collections
   - Collection management dialog

3. **Persist collections**:
   - Save in app settings or separate file
   - Update `appService` to handle collection storage

### Adding Book Tags

To implement tagging system:

1. **Update `BookMetadata`**:
   ```typescript
   interface BookMetadata {
     // ...
     tags: string[];
   }
   ```

2. **Add tag management UI**:
   ```typescript
   function TagEditor({ book }: { book: BookMetadata }) {
     const updateBook = useLibraryStore(state => state.updateBook);

     const addTag = (tag: string) => {
       updateBook(book.hash, {
         tags: [...(book.tags || []), tag],
       });
     };

     return (
       <div>
         {book.tags?.map(tag => (
           <span key={tag}>{tag}</span>
         ))}
         <button onClick={() => addTag(prompt('Tag name'))}>+</button>
       </div>
     );
   }
   ```

3. **Add tag filtering**:
   - Filter bookshelf by selected tags
   - Show tag cloud or list

### Improving Book Import

To enhance import functionality:

1. **Batch import with progress**:
   ```typescript
   async importBooksWithProgress(
     files: File[],
     onProgress: (current: number, total: number) => void
   ): Promise<void> {
     for (let i = 0; i < files.length; i++) {
       await this.importBook(files[i]!);
       onProgress(i + 1, files.length);
     }
   }
   ```

2. **Add import validation**:
   - Check for duplicates (by hash)
   - Validate file format
   - Show import errors/warnings

3. **Import from URL**:
   ```typescript
   async importFromUrl(url: string): Promise<BookMetadata> {
     const response = await fetch(url);
     const blob = await response.blob();
     const file = new File([blob], url.split('/').pop()!);
     return this.importBook(file);
   }
   ```

### Adding Export Functionality

To export books:

1. **Export single book**:
   ```typescript
   async exportBook(hash: string): Promise<void> {
     const book = this.library.find(b => b.hash === hash);
     if (!book) throw new Error('Book not found');

     const file = await this.appService.loadBookFile(book);
     // Trigger download or save dialog
     this.appService.saveFile(file, book.title);
   }
   ```

2. **Export with metadata**:
   - Package book + cover + annotations as ZIP
   - Export in standard format (e.g., Calibre-compatible)

3. **Bulk export**:
   - Export selected books
   - Export entire library

### Improving Cover Management

To enhance cover handling:

1. **Custom cover upload**:
   ```typescript
   async updateCover(hash: string, coverFile: File): Promise<void> {
     const url = URL.createObjectURL(coverFile);
     this.updateBook(hash, { coverImageUrl: url });
     await this.appService.saveCoverImage(hash, coverFile);
   }
   ```

2. **Cover extraction improvement**:
   - Better fallback strategies
   - Fetch covers from online databases (Open Library, Google Books)
   - Generate covers from metadata (title + author on colored background)

3. **Cover caching**:
   - Implement LRU cache for cover images
   - Lazy load covers in viewport
   - Optimize cover image sizes

### Adding Library Statistics

To show library statistics:

1. **Compute stats**:
   ```typescript
   const libraryStats = useLibraryStore(state => ({
     totalBooks: state.library.length,
     byFormat: Object.groupBy(state.library, b => b.format),
     recentlyAdded: state.library
       .sort((a, b) => b.addedDate - a.addedDate)
       .slice(0, 5),
   }));
   ```

2. **Display in UI**:
   - Stats panel in library header
   - Charts/graphs for visualization
   - Reading progress summary

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add search/filter | `libraryStore.ts`, `LibraryHeader.tsx` | `Bookshelf.tsx` |
| Implement collections | `libraryStore.ts` | New collection UI components |
| Add tags | `BookMetadata` type, `libraryStore.ts` | Tag editor component |
| Improve import | `appService.ts`, `LibraryHeader.tsx` | Import dialog component |
| Add export | `appService.ts` | Export dialog, file utils |
| Enhance covers | `appService.ts` | Cover cache utilities |
| Add statistics | `libraryStore.ts` | Stats display component |
| Custom sorting | `libraryStore.ts`, `Bookshelf.tsx` | Sort controls |

## Common Issues and Debugging

### Problem: Books not appearing in library

- Check if `libraryStore.library` is populated
- Verify `appService.loadLibrary()` is called on app start
- Check for errors during import
- Verify book hash calculation is consistent

### Problem: Covers not displaying

- Check if `coverImageUrl` is valid
- Verify cover cache directory is accessible
- Check browser console for image load errors
- Test with `DEFAULT_COVER` placeholder

### Problem: Duplicate books

- Verify MD5 hash is calculated correctly
- Check for hash collisions (unlikely but possible)
- Add duplicate detection in import flow
- Allow manual merge of duplicates

### Problem: Import fails silently

- Add error logging in `appService.importBook()`
- Check `DocumentLoader` for parsing errors
- Verify file permissions (native app)
- Add user-facing error messages

## Dependencies

- **Tauri Dialog API**: File picker for import
- **Tauri FS API**: File system access for storage
- **DocumentLoader**: Book parsing and metadata extraction
- **MD5 utility**: Hash generation for book IDs

## Performance Considerations

- **Lazy load covers**: Only load visible covers in bookshelf
- **Virtual scrolling**: For large libraries (100+ books)
- **Debounce search**: Don't filter on every keystroke
- **Cache metadata**: Don't re-parse books unnecessarily
- **Optimize cover sizes**: Resize covers to thumbnail dimensions
- **Batch operations**: Import/delete multiple books efficiently
