# Feature: Library Management

## Overview

## Sub-Features

- **[Reading Progress Indicators](./reading-progress-indicators.md)** - Visual progress indicators for books (Commit 46fdea35)


The Library Management feature handles the book collection, including import, organization, metadata management, and cover image generation. At commit `d757555f`, the library provides a grid-based bookshelf view with book metadata cards.

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

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### Select All Button in Select Mode (v0.9.48, #1209)

**Feature**: "Select All" button when in select mode for batch operations.

**UI Location**: Top bar when select mode is active

**Functionality**:
- Selects all books in current view/filter
- Visual feedback for selected state
- Enables batch actions: delete, export, add to group

**Implementation** (`src/app/library/components/LibraryHeader.tsx`):
```typescript
const handleSelectAll = () => {
  const visibleBooks = getFilteredBooks();
  setSelectedBooks(visibleBooks.map(b => b.hash));
};

return (
  <div className="select-mode-header">
    <button onClick={handleSelectAll}>
      Select All ({filteredBooks.length})
    </button>
    <button onClick={handleDeselectAll}>Clear</button>
  </div>
);
```

**Select Filtered Books** (v0.9.48, #1237):
- "Select All" respects current search/filter
- Only selects visible books, not entire library
- Clear indication of selection count

**File**: `src/app/library/components/LibraryHeader.tsx`

### Show Current Books Count (v0.9.52, #1312)

**Feature**: Display total book count in library header/search bar.

**UI**: "342 books" displayed in library header

**Dynamic Updates**:
- Updates when books imported/deleted
- Shows filtered count when search active
- Format: "X of Y books" when filtered

**Implementation** (`src/app/library/components/LibraryHeader.tsx`):
```typescript
const BookCount = () => {
  const { books, filteredBooks, isFiltering } = useLibraryStore();

  return (
    <div className="book-count">
      {isFiltering ? (
        <span>{filteredBooks.length} of {books.length} books</span>
      ) : (
        <span>{books.length} books</span>
      )}
    </div>
  );
};
```

**File**: `src/app/library/components/LibraryHeader.tsx`

### Update Bookshelf After Import/Delete (v0.9.52-0.9.53, #1314, #1331, #1336)

**Enhancement**: Automatic bookshelf refresh after library modifications.

**Issues Fixed**:
- Bookshelf didn't update after importing books (#1314, #1331)
- Deleted books still appeared until page refresh
- Book count not updated after operations

**Solution** (`src/store/libraryStore.ts`):
```typescript
// Trigger library update event
const importBooks = async (files: File[]) => {
  const imported = await processImport(files);

  // Update library state
  set(state => ({
    books: [...state.books, ...imported]
  }));

  // Trigger UI refresh event
  emit('library-updated', { action: 'import', count: imported.length });
};

const deleteBooks = async (hashes: string[]) => {
  await batchDelete(hashes);

  // Update library state
  set(state => ({
    books: state.books.filter(b => !hashes.includes(b.hash))
  }));

  // Trigger UI refresh event
  emit('library-updated', { action: 'delete', count: hashes.length });
};
```

**Bookshelf Listener** (`src/app/library/components/Bookshelf.tsx`):
```typescript
useEffect(() => {
  const handleLibraryUpdate = () => {
    // Refresh current bookshelf view
    refreshBookshelf();
  };

  on('library-updated', handleLibraryUpdate);
  return () => off('library-updated', handleLibraryUpdate);
}, []);
```

**Files**:
- `src/store/libraryStore.ts`
- `src/app/library/components/Bookshelf.tsx`

### Delete Cloud Backup Only (v0.9.63, #1546)

**Feature**: Option to delete only the cloud backup of a book while keeping local copy.

**Use Case**: Free up cloud storage quota without deleting local book.

**UI**: Book context menu > "Delete Cloud Backup"

**Confirmation Dialog**:
```
Delete cloud backup for "Book Title"?

The book will remain in your local library.
Your reading progress and notes will be kept.

[Cancel] [Delete Cloud Backup]
```

**Implementation** (`src/services/syncService.ts`):
```typescript
const deleteCloudBackup = async (bookHash: string) => {
  // Delete from Supabase storage
  await supabase.storage
    .from('books')
    .remove([`${userId}/${bookHash}.epub`]);

  // Update sync status
  await supabase
    .from('book_sync_status')
    .update({ cloud_backup: false })
    .eq('book_hash', bookHash)
    .eq('user_id', userId);

  // Keep local book and data
  // No changes to local library
};
```

**Status Indicator**:
- Icon shows: Cloud synced, Local only, or Cloud + Local
- Tooltip explains sync status
- Quick toggle in book details

**Files**:
- `src/services/syncService.ts`
- `src/app/library/components/BookContextMenu.tsx`
- `src/components/CloudSyncStatus.tsx`

### Exit Select Mode When All Deleted (v0.9.53, #1350)

**Enhancement**: Automatically exit select mode when all selected books are deleted.

**Behavior**:
1. User enters select mode
2. Selects multiple books
3. Deletes all selected books
4. Select mode automatically exits
5. Returns to normal library view

**Implementation** (`src/app/library/components/Bookshelf.tsx`):
```typescript
const handleDelete = async (hashes: string[]) => {
  await deleteBooks(hashes);

  // Check if any books remain
  const remainingBooks = books.filter(b => !hashes.includes(b.hash));

  if (remainingBooks.length === 0 || selectedBooks.length === books.length) {
    // Exit select mode if all books deleted
    setSelectMode(false);
    setSelectedBooks([]);
  }
};
```

**File**: `src/app/library/components/Bookshelf.tsx`

---

## Version 0.9.64 - 0.9.67 Updates (f5b686ab → 33b2ba16)

### Book Metadata Editor (v0.9.64, #1583)

**Major Feature**: Edit book metadata directly from the library interface.

**Overview**: Users can now modify book title, author, publisher, language, and other metadata fields without re-importing books.

**UI Access**:
- Book context menu > "Edit Metadata"
- Book details modal > "Edit" button
- Keyboard shortcut: `E` when book selected

**Editable Fields**:
- Title
- Author
- Publisher
- Publication date
- Language
- ISBN
- Description/Summary
- Tags
- Custom fields

**Implementation** (`src/app/library/components/MetadataEditor.tsx`):
```typescript
interface MetadataEditorProps {
  book: BookMetadata;
  onSave: (updates: Partial<BookMetadata>) => Promise<void>;
  onCancel: () => void;
}

const MetadataEditor: React.FC<MetadataEditorProps> = ({ book, onSave, onCancel }) => {
  const [formData, setFormData] = useState({
    title: book.title,
    author: book.author,
    publisher: book.publisher,
    language: book.language,
    // ...other fields
  });

  const handleSave = async () => {
    await onSave(formData);
    toast.success('Metadata updated successfully');
  };

  return (
    <Dialog>
      <Input label="Title" value={formData.title} onChange={...} />
      <Input label="Author" value={formData.author} onChange={...} />
      {/* ...other inputs */}
      <Button onClick={handleSave}>Save</Button>
      <Button onClick={onCancel}>Cancel</Button>
    </Dialog>
  );
};
```

**Validation**:
- Required fields: Title, Author
- Language code validation
- Date format validation
- ISBN format validation

**Files**:
- `src/app/library/components/MetadataEditor.tsx` - Editor UI component
- `src/store/libraryStore.ts` - Update metadata action
- `src/services/appService.ts` - Persist metadata changes

### Custom Cover Image Upload (v0.9.64, #1588)

**Feature**: Upload custom cover images for books.

**UI**: Metadata editor > Cover image section > "Upload Custom Cover"

**Supported Formats**:
- JPEG (.jpg, .jpeg)
- PNG (.png)
- WebP (.webp)
- Maximum size: 5MB

**Implementation** (`src/app/library/components/CoverUploader.tsx`):
```typescript
const CoverUploader = ({ book, onCoverUpdate }) => {
  const handleFileSelect = async (file: File) => {
    // Validate file
    if (!ALLOWED_FORMATS.includes(file.type)) {
      toast.error('Invalid format. Use JPEG, PNG, or WebP');
      return;
    }

    if (file.size > MAX_SIZE) {
      toast.error('File too large. Maximum size is 5MB');
      return;
    }

    // Process image
    const resized = await resizeImage(file, { maxWidth: 800, maxHeight: 1200 });
    const url = URL.createObjectURL(resized);

    // Save to app storage
    await appService.saveCoverImage(book.hash, resized);

    // Update metadata
    onCoverUpdate(url);
  };

  return (
    <div className="cover-uploader">
      <img src={book.coverImageUrl} alt={book.title} />
      <input type="file" accept="image/*" onChange={handleFileSelect} />
      <button>Upload Custom Cover</button>
    </div>
  );
};
```

**Features**:
- Automatic image resizing for optimal storage
- Preview before saving
- Revert to original cover option
- Custom covers saved in apps (#1588)

**Storage**:
- Custom covers stored separately from extracted covers
- Indexed by book hash
- Synced to cloud if user is signed in

**Files**:
- `src/app/library/components/CoverUploader.tsx`
- `src/services/appService.ts` - Cover image persistence
- `src/utils/image.ts` - Image resizing utilities

### Metadata Sync Across Devices (v0.9.65, #1611)

**Feature**: Sync book metadata changes across all user devices.

**Overview**: When a user edits book metadata on one device, changes are automatically synced to all their other devices.

**Sync Scope**:
- Book metadata (title, author, etc.)
- Custom cover images
- User-added fields
- Tags and categories

**Implementation** (`src/services/syncService.ts`):
```typescript
const syncMetadata = async (bookHash: string, updates: Partial<BookMetadata>) => {
  // Update local store
  libraryStore.updateBook(bookHash, updates);

  // Sync to cloud
  if (isAuthenticated()) {
    await supabase
      .from('book_metadata')
      .upsert({
        user_id: userId,
        book_hash: bookHash,
        metadata: updates,
        updated_at: new Date().toISOString()
      });
  }
};

// Listen for remote metadata changes
const subscribeToMetadataChanges = () => {
  supabase
    .channel('metadata-changes')
    .on('postgres_changes', {
      event: 'UPDATE',
      schema: 'public',
      table: 'book_metadata',
      filter: `user_id=eq.${userId}`
    }, (payload) => {
      const { book_hash, metadata } = payload.new;
      libraryStore.updateBook(book_hash, metadata);
      toast.info(`Metadata updated for "${metadata.title}"`);
    })
    .subscribe();
};
```

**Conflict Resolution**:
- Last-write-wins strategy
- Timestamp-based conflict resolution
- Local changes take precedence if offline

**Files**:
- `src/services/syncService.ts` - Sync logic
- `src/store/libraryStore.ts` - Local state updates
- `src/services/supabase.ts` - Database operations

### Search in Book Format and Group Names (v0.9.67, #1662)

**Feature**: Enhanced library search with format and group filtering.

**Search Capabilities**:
1. **Book Format**: Filter by EPUB, PDF, MOBI, CBZ, FB2, TXT
2. **Group Names**: Search in custom group/collection names
3. **Group Descriptions**: Search in group description text
4. **Combined Search**: Search across multiple fields simultaneously

**UI Implementation** (`src/app/library/components/LibrarySearch.tsx`):
```typescript
const LibrarySearch = () => {
  const [searchQuery, setSearchQuery] = useState('');
  const [formatFilter, setFormatFilter] = useState<BookFormat | 'all'>('all');

  const filteredBooks = useLibraryStore(state => {
    let books = state.library;

    // Filter by format
    if (formatFilter !== 'all') {
      books = books.filter(b => b.format === formatFilter);
    }

    // Filter by search query
    if (searchQuery) {
      const query = searchQuery.toLowerCase();
      books = books.filter(b =>
        b.title.toLowerCase().includes(query) ||
        b.author.toLowerCase().includes(query) ||
        b.format.toLowerCase().includes(query) ||
        state.groups
          .filter(g => g.bookHashes.includes(b.hash))
          .some(g =>
            g.name.toLowerCase().includes(query) ||
            g.description?.toLowerCase().includes(query)
          )
      );
    }

    return books;
  });

  return (
    <div className="library-search">
      <input
        type="text"
        placeholder="Search books, formats, or groups..."
        value={searchQuery}
        onChange={(e) => setSearchQuery(e.target.value)}
      />
      <select value={formatFilter} onChange={(e) => setFormatFilter(e.target.value)}>
        <option value="all">All Formats</option>
        <option value="EPUB">EPUB</option>
        <option value="PDF">PDF</option>
        <option value="MOBI">MOBI</option>
        <option value="CBZ">CBZ</option>
        <option value="FB2">FB2/FBZ</option>
        <option value="TXT">TXT</option>
      </select>
    </div>
  );
};
```

**Search Examples**:
- `"epub"` - Shows all EPUB books
- `"fantasy"` - Shows books in groups with "fantasy" in name/description
- `"tolkien pdf"` - Shows PDF books by Tolkien

**Performance**:
- Debounced search input (300ms delay)
- Indexed search for large libraries
- Cached filter results

**Files**:
- `src/app/library/components/LibrarySearch.tsx`
- `src/store/libraryStore.ts` - Search logic
- `src/utils/search.ts` - Search utilities

### Various Metadata and Bookshelf Fixes (v0.9.67, #1663)

**Enhancements**: Multiple bug fixes and improvements to metadata editor and bookshelf.

**Fixes Included**:
1. **Metadata Editor**:
   - Fixed form validation not working
   - Corrected date picker format
   - Improved error handling
   - Better mobile layout

2. **Bookshelf Display**:
   - Fixed book cards not updating after metadata edit
   - Corrected cover image caching issues
   - Improved grid layout on various screen sizes
   - Fixed selection state persistence

3. **Performance**:
   - Enabled parallel web builds checking
   - Optimized re-renders in bookshelf
   - Improved metadata save performance

**Translation Updates**:
- Updated i18n translations for metadata editor
- Added missing translation keys
- Improved language consistency

**Files Modified**:
- `src/app/library/components/MetadataEditor.tsx`
- `src/app/library/components/Bookshelf.tsx`
- `src/store/libraryStore.ts`
- `src/locales/*.json` - Translation files

---

## Version 0.9.68 - 0.9.78 Updates (33b2ba16 → cc3cc58d)

### Delete Local Copy Feature (v0.9.70, #1773)

**Feature**: Delete only the local copy of a book while keeping cloud backup intact.

**Overview**: Users can now choose to delete just the local copy of a book to free up device storage, while maintaining the book in their cloud library for later re-download.

**Use Cases**:
- Free up local storage space on device
- Keep cloud backup for archival purposes
- Temporary removal of infrequently read books
- Manage storage on devices with limited space

**UI Options**:

Book context menu now offers two deletion options:
1. **Delete Locally** - Removes local copy, keeps cloud backup
2. **Delete Everywhere** - Removes both local copy and cloud backup

**Implementation** (`src/app/library/components/BookContextMenu.tsx`):
```typescript
const handleDeleteLocal = async (bookHash: string) => {
  // Delete local file only
  await appService.deleteBookFile(bookHash);

  // Update library metadata (mark as not-downloaded)
  libraryStore.updateBook(bookHash, {
    isDownloaded: false,
    localPath: null
  });

  toast.success('Local copy deleted. Book remains in cloud.');
};

const handleDeleteEverywhere = async (bookHash: string) => {
  // Delete local file
  await appService.deleteBookFile(bookHash);

  // Delete from cloud storage
  if (isAuthenticated()) {
    await syncService.deleteCloudBook(bookHash);
  }

  // Remove from library
  libraryStore.removeBook(bookHash);

  toast.success('Book deleted from all locations.');
};
```

**Confirmation Dialog**:
```
Delete "Book Title" locally?

The book will be removed from this device, but will remain in your cloud library.
You can re-download it anytime from the cloud.

Your reading progress and notes will be preserved.

[Cancel] [Delete Local Copy]
```

**Re-Download**:
- Books deleted locally show a cloud download icon in the library
- Click to re-download from cloud storage
- Progress and annotations restored automatically

**Files**:
- `src/app/library/components/BookContextMenu.tsx` - Deletion options
- `src/app/library/components/BookCard.tsx` - Download status indicator
- `src/services/appService.ts` - File deletion logic
- `src/services/syncService.ts` - Cloud deletion
- `src/store/libraryStore.ts` - Library state updates

### Author Sorting (Last Name First) (v0.9.75, #1865)

**Feature**: Sort library books by author with last name first.

**Overview**: Books can now be sorted alphabetically by author's last name, following bibliographic conventions common in libraries and catalogs.

**Supported Languages**:
- Arabic
- English
- German
- Spanish
- French
- Hindi
- Italian
- Dutch
- Polish
- Portuguese
- Russian
- Thai
- Turkish
- Ukrainian
- And more...

**Implementation** (`src/utils/authorSort.ts`):
```typescript
// Extract last name from author string
const getLastName = (authorName: string, language: string): string => {
  // Handle different naming conventions by language
  const nameParts = authorName.trim().split(/\s+/);

  if (language === 'ar') {
    // Arabic: last name is typically first
    return nameParts[0];
  } else {
    // Western naming: last name is typically last
    return nameParts[nameParts.length - 1];
  }
};

// Sort books by author last name
const sortByAuthor = (books: BookMetadata[], language: string) => {
  return books.sort((a, b) => {
    const lastNameA = getLastName(a.author, language);
    const lastNameB = getLastName(b.author, language);
    return lastNameA.localeCompare(lastNameB, language);
  });
};
```

**UI Integration** (`src/app/library/components/LibraryHeader.tsx`):
```typescript
<select
  value={sortBy}
  onChange={(e) => setSortBy(e.target.value)}
>
  <option value="title">Title</option>
  <option value="author">Author (Last Name)</option>
  <option value="recent">Recently Added</option>
  <option value="lastOpened">Last Opened</option>
</select>
```

**Example Sorting**:

Before (by first name):
- Albert Einstein
- Carl Sagan
- Marie Curie
- Stephen Hawking

After (by last name):
- Marie Curie
- Albert Einstein
- Stephen Hawking
- Carl Sagan

**Locale-Aware Sorting**:
- Uses `String.localeCompare()` for proper alphabetical order in each language
- Handles diacritics and special characters correctly
- Respects cultural naming conventions

**Files**:
- `src/utils/authorSort.ts` - Author name parsing and sorting logic
- `src/app/library/components/LibraryHeader.tsx` - Sort dropdown UI
- `src/store/libraryStore.ts` - Sorting state management

### Library Data Resilience (v0.9.67, #1689)

**Feature**: Automatic recovery from corrupted library data using backup.

**Overview**: The app now maintains a backup copy of the library metadata and automatically loads it if the main library data becomes corrupted or unavailable.

**Problem**:
- Power failures or crashes during save operations could corrupt the library JSON file
- Users would lose their entire library metadata
- Reading progress and book organization lost

**Solution**:

**Backup Strategy**:
1. **Automatic Backups**: Library data backed up on every successful save
2. **Backup Location**: `library.json.backup` in app data directory
3. **Corruption Detection**: JSON parse errors trigger backup load
4. **User Notification**: Toast message when backup is loaded

**Implementation** (`src/services/appService.ts`):
```typescript
async loadLibrary(): Promise<BookMetadata[]> {
  try {
    // Try to load main library file
    const libraryData = await this.readFile('library.json');
    const library = JSON.parse(libraryData);
    return library;
  } catch (mainError) {
    console.error('Failed to load main library data:', mainError);

    try {
      // Load backup if main file fails
      const backupData = await this.readFile('library.json.backup');
      const library = JSON.parse(backupData);

      // Notify user
      toast.warning('Main library data was unavailable. Loaded from backup.');

      // Restore main file from backup
      await this.writeFile('library.json', backupData);

      return library;
    } catch (backupError) {
      console.error('Failed to load backup library data:', backupError);

      // Both failed - return empty library
      toast.error('Unable to load library data. Starting with empty library.');
      return [];
    }
  }
}

async saveLibrary(library: BookMetadata[]): Promise<void> {
  const libraryData = JSON.stringify(library, null, 2);

  // Save main file
  await this.writeFile('library.json', libraryData);

  // Create backup
  await this.writeFile('library.json.backup', libraryData);
}
```

**Recovery Process**:
1. App attempts to load `library.json`
2. If JSON parsing fails, load `library.json.backup`
3. Restore main file from backup
4. Show notification to user
5. Continue normal operation

**Additional Safety**:
- Cloud sync provides additional backup layer
- Export library feature for manual backups
- Corruption-resistant JSON serialization

**Files**:
- `src/services/appService.ts` - Backup/restore logic
- `src/services/nativeAppService.ts` - Native file operations
- `src/store/libraryStore.ts` - Library state initialization

---

## Dependencies

- **Tauri Dialog API**: File picker for import
- **Tauri FS API**: File system access for storage
- **DocumentLoader**: Book parsing and metadata extraction
- **MD5 utility**: Hash generation for book IDs

---

## Version 0.9.79 - 0.9.82 Updates (cc3cc58d → e1691661)

### Library List View with Groups (v0.9.80, #2009)

**Feature**: Display book groups/collections in list view mode.

**Overview**: Previously, groups were only visible in grid view. Now list view also shows group organization, making it easier to browse large libraries.

**Implementation** (`src/app/library/components/LibraryList.tsx`):
```typescript
const LibraryList = ({ books, groupBy }: LibraryListProps) => {
  const groupedBooks = useMemo(() => {
    if (!groupBy) return { ungrouped: books };

    return books.reduce((groups, book) => {
      const key = getGroupKey(book, groupBy);
      if (!groups[key]) groups[key] = [];
      groups[key].push(book);
      return groups;
    }, {} as Record<string, BookMetadata[]>);
  }, [books, groupBy]);

  return (
    <div className="library-list">
      {Object.entries(groupedBooks).map(([groupName, groupBooks]) => (
        <div key={groupName} className="book-group">
          {groupBy && (
            <h3 className="group-header">{groupName}</h3>
          )}
          {groupBooks.map(book => (
            <BookListItem key={book.hash} book={book} />
          ))}
        </div>
      ))}
    </div>
  );
};
```

**Grouping Options**:
- **Author**: Group by first author
- **Series**: Group by series name
- **Date Added**: Group by import date
- **Reading Status**: Group by read/unread/in-progress
- **Genre/Tags**: Group by custom tags

**UI Design**:
```
┌─────────────────────────────────────┐
│ Author: J.R.R. Tolkien              │
├─────────────────────────────────────┤
│ □ The Hobbit        [75%] [★★★★☆]  │
│ □ The Fellowship... [100%] [★★★★★] │
│ □ The Two Towers    [32%] [★★★★☆]  │
├─────────────────────────────────────┤
│ Author: Brandon Sanderson           │
├─────────────────────────────────────┤
│ □ Mistborn          [0%] [☆☆☆☆☆]   │
│ □ The Way of Kings  [45%] [★★★★★]  │
└─────────────────────────────────────┘
```

**Features**:
- Collapsible group headers
- Group count badges
- Sort within groups
- Search across all groups
- Sticky group headers on scroll

**Files**:
- `src/app/library/components/LibraryList.tsx` - List view with groups
- `src/app/library/components/GroupHeader.tsx` - Collapsible header
- `src/hooks/useLibraryGrouping.ts` - Grouping logic

### Delete Confirmation from Context Menu (v0.9.80, #2029)

**Feature**: Confirmation dialog when deleting books from context menu.

**Problem**: Users could accidentally delete books by clicking the wrong context menu option.

**Solution**: Show confirmation dialog before permanent deletion.

**Implementation**:
```typescript
const handleDeleteFromContextMenu = async (book: BookMetadata) => {
  const confirmed = await showDialog({
    title: 'Delete Book',
    message: `Are you sure you want to delete "${book.title}"?`,
    detail: 'This will remove the book from your library. This action cannot be undone.',
    buttons: ['Cancel', 'Delete'],
    destructive: 1,  // Make "Delete" button red
    defaultButton: 0  // Focus "Cancel" by default
  });

  if (confirmed === 1) {  // User clicked "Delete"
    await deleteBook(book.hash);
    toast.success(`"${book.title}" has been deleted`);
  }
};
```

**Dialog Appearance**:
```
┌──────────────────────────────────────┐
│  Delete Book                     × │
├──────────────────────────────────────┤
│                                      │
│  Are you sure you want to delete     │
│  "The Hobbit"?                       │
│                                      │
│  This will remove the book from your │
│  library. This action cannot be      │
│  undone.                             │
│                                      │
├──────────────────────────────────────┤
│         [Cancel]  [Delete]           │
└──────────────────────────────────────┘
```

**Platform-Specific Implementation**:
- **Desktop**: Native dialog via Tauri
- **Mobile**: Modal overlay
- **Web**: Browser-native confirm() or custom modal

**Additional Safety**:
- Destructive button styled in red
- Default focus on "Cancel"
- ESC key closes dialog without deleting
- Click outside dismisses dialog

**Files**:
- `src/components/ConfirmDialog.tsx` - Confirmation dialog
- `src/app/library/components/BookContextMenu.tsx` - Context menu
- `src/services/dialogService.ts` - Platform-specific dialogs

### Cover Image Management (v0.9.80, #2028)

**Feature**: Option to remove cover image from a book.

**Use Cases**:
- Book has incorrect/placeholder cover
- User wants to use default cover
- Cover image is corrupted
- Reduce storage usage

**Implementation**:
```typescript
const removeCoverImage = async (bookHash: string) => {
  const book = getBook(bookHash);

  // Remove cover file if it exists
  if (book.coverImageUrl && !book.coverImageUrl.startsWith('http')) {
    await deleteFile(book.coverImageUrl);
  }

  // Update book metadata
  updateBook(bookHash, {
    ...book,
    coverImageUrl: null,
    hasCover: false
  });

  // Sync to cloud if enabled
  if (isSyncEnabled()) {
    await syncBookMetadata(bookHash);
  }

  toast.success('Cover image removed');
};
```

**UI**:
```typescript
const CoverActions = ({ book }: CoverActionsProps) => {
  return (
    <Menu>
      <MenuItem onClick={() => changeCover(book)}>
        Change Cover...
      </MenuItem>
      {book.hasCover && (
        <MenuItem onClick={() => removeCover(book)}>
          Remove Cover
        </MenuItem>
      )}
      <MenuItem onClick={() => extractCover(book)}>
        Extract from Book
      </MenuItem>
    </Menu>
  );
};
```

**Fallback Behavior**:
- Shows default cover placeholder
- Displays first letter of title
- Uses theme colors for background

**Files**:
- `src/services/coverService.ts` - Cover management
- `src/app/library/components/CoverActions.tsx` - Cover context menu
- `src/components/DefaultCover.tsx` - Placeholder cover

### Robust Metadata Import (v0.9.80, #2064)

**Feature**: Handle invalid language codes and titles when importing books.

**Problem**: Some ebooks have malformed metadata that caused import failures.

**Solutions**:

**Invalid Language Codes**:
```typescript
const normalizeLanguageCode = (code: string | string[]): string => {
  // Handle array of languages
  if (Array.isArray(code)) {
    return normalizeLanguageCode(code[0] || 'en');
  }

  // Handle undefined/null
  if (!code) return 'en';

  // Normalize code
  const normalized = code.toLowerCase().trim();

  // Map invalid codes to valid ones
  const languageMap: Record<string, string> = {
    'und': 'en',          // Undefined → English
    'unknown': 'en',
    'zxx': 'en',          // No linguistic content
    'mul': 'en',          // Multiple languages
    '': 'en',
    'en-us': 'en',        // Regional → Base
    'en-gb': 'en',
    'zh-cn': 'zh',
    'zh-tw': 'zh'
  };

  return languageMap[normalized] || normalized;
};
```

**Invalid Titles**:
```typescript
const sanitizeTitle = (title: string | undefined, filename: string): string => {
  // Handle missing title
  if (!title || title.trim() === '') {
    // Extract from filename
    const name = filename.replace(/\.[^.]+$/, ''); // Remove extension
    return name || 'Untitled Book';
  }

  // Remove control characters
  let sanitized = title.replace(/[\x00-\x1F\x7F]/g, '');

  // Trim excessive whitespace
  sanitized = sanitized.replace(/\s+/g, ' ').trim();

  // Limit length
  if (sanitized.length > 200) {
    sanitized = sanitized.substring(0, 200) + '...';
  }

  return sanitized || 'Untitled Book';
};
```

**Import with Validation**:
```typescript
const importBook = async (file: File): Promise<BookMetadata> => {
  const { book, format } = await DocumentLoader.open(file);

  const metadata: BookMetadata = {
    hash: generateHash(file),
    filename: file.name,
    title: sanitizeTitle(book.metadata?.title, file.name),
    authors: sanitizeAuthors(book.metadata?.author),
    language: normalizeLanguageCode(book.metadata?.language),
    format,
    dateAdded: Date.now(),
    // ...other fields
  };

  return metadata;
};
```

**Error Recovery**:
- Invalid metadata → Use filename
- Missing language → Default to English
- Corrupted fields → Skip and log warning
- Continue import even with partial metadata

**Files**:
- `src/utils/metadataValidation.ts` - Validation utilities
- `src/libs/document.ts` - Import with validation
- `src/services/importService.ts` - Robust import flow

---

## Performance Considerations

- **Lazy load covers**: Only load visible covers in bookshelf
- **Virtual scrolling**: For large libraries (100+ books)
- **Debounce search**: Don't filter on every keystroke
- **Cache metadata**: Don't re-parse books unnecessarily
- **Optimize cover sizes**: Resize covers to thumbnail dimensions
- **Batch operations**: Import/delete multiple books efficiently
