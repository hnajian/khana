# Feature: Document Reading Engine

## Overview

The Document Reading Engine is the core functionality of Readest that enables loading, parsing, and rendering of multiple ebook and document formats. At commit `571baf98`, it supports EPUB, PDF, MOBI, CBZ (comic books), and FB2/FBZ formats through integration with the foliate-js library.

## Key Components

### Primary Files

- **`src/libs/document.ts`** - Universal document loader and format detection
- **`packages/foliate-js/`** - Vendored fork of foliate-js reading engine (git submodule)
- **`src/app/reader/components/FoliateViewer.tsx`** - React wrapper for foliate-js viewer
- **`public/vendor/pdfjs/`** - pdf.js library assets (copied during build)

### Related Files

- **`src/types/book.ts`** - Type definitions for `BookFormat`, `BookContent`, `BookConfig`, etc.
- **`src/services/appService.ts`** - `loadBookContent()` method
- **`src/store/bookDataStore.ts`** - Stores loaded book data

## Architecture

### Document Loader (`src/libs/document.ts`)

The `DocumentLoader` class provides format-agnostic book loading:

```typescript
export class DocumentLoader {
  constructor(file: File)
  async open(): Promise<{ book: BookDoc; format: BookFormat }>
}
```

**Format Detection Strategy:**
1. **ZIP-based formats**: Checks first 4 bytes for ZIP magic number (`0x50 0x4B 0x03 0x04`)
   - **EPUB**: Default for ZIP files (foliate-js EPUB parser)
   - **CBZ**: Detected by MIME type or `.cbz` extension
   - **FBZ**: Detected by `.fb2.zip` or `.fbz` extension
2. **PDF**: Checks first 5 bytes for PDF magic (`0x25 0x50 0x44 0x46 0x2D` = `%PDF-`)
3. **MOBI**: Uses foliate-js `isMOBI()` function
4. **FB2**: Detected by MIME type or `.fb2` extension

### Supported Formats

| Format | Extension | Library Used | Notes |
|--------|-----------|--------------|-------|
| EPUB | `.epub` | foliate-js/epub.js | Default for ZIP files |
| PDF | `.pdf` | foliate-js/pdf.js | Wraps pdf.js |
| MOBI | `.mobi` | foliate-js/mobi.js | Requires fflate for decompression |
| CBZ | `.cbz` | foliate-js/comic-book.js | Comic book archive |
| FB2 | `.fb2` | foliate-js/fb2.js | FictionBook XML |
| FBZ | `.fbz`, `.fb2.zip` | foliate-js/fb2.js | Compressed FB2 |

### Book Document Interface

All formats are normalized to a common `BookDoc` interface:

```typescript
export interface BookDoc {
  metadata: {
    title: string;
    author: string;
    language: string | string[];
    editor?: string;
    publisher?: string;
  };
  toc: Array<TOCItem>;
  getCover(): Promise<Blob | null>;
}
```

### Integration with Reader Store

The reader loads books through this flow:

1. **User selects book** → Library triggers load
2. **`readerStore.initViewState()`** → Calls `appService.loadBookContent()`
3. **AppService** → Retrieves file from storage
4. **`DocumentLoader.open()`** → Parses format and returns `BookDoc`
5. **TOC processing** → `updateTocID()` assigns IDs to TOC items
6. **Store update** → `bookDataStore` caches the loaded book
7. **View initialization** → `FoliateViewer` renders the book

## AI Agent Modification Guidelines

### Adding Support for a New Format

To add support for a new ebook format (e.g., AZW3):

1. **Update `src/libs/document.ts`**:
   - Add format to `BookFormat` type in `src/types/book.ts`
   - Add extension to `EXTS` constant
   - Implement format detection method (e.g., `isAZW3()`)
   - Add case in `open()` method to handle the new format
   - Import the appropriate foliate-js parser (or create new one)

2. **Example code**:
   ```typescript
   // In src/types/book.ts
   export type BookFormat = 'EPUB' | 'PDF' | 'MOBI' | 'CBZ' | 'FB2' | 'FBZ' | 'AZW3';

   // In src/libs/document.ts
   private isAZW3(): boolean {
     return this.file.name.endsWith('.azw3');
   }

   public async open(): Promise<{ book: BookDoc; format: BookFormat }> {
     // ... existing code ...
     else if (this.isAZW3()) {
       const { makeAZW3 } = await import('foliate-js/azw3.js');
       book = await makeAZW3(this.file);
       format = 'AZW3';
     }
   }
   ```

3. **Update foliate-js** (if needed):
   - Navigate to `packages/foliate-js/`
   - Implement format parser following foliate-js conventions
   - Ensure it returns a `BookDoc`-compatible object

4. **Test the integration**:
   - Verify file import works
   - Check metadata extraction
   - Ensure TOC is populated correctly
   - Test cover image retrieval

### Modifying Document Metadata Extraction

To enhance or fix metadata extraction:

1. **For EPUB/MOBI/FB2**: Modify the corresponding parser in `packages/foliate-js/`
2. **For all formats**: Post-process in `appService.loadBookContent()`
   - Location: `src/services/appService.ts:loadBookContent()`
   - Add transformations after `DocumentLoader.open()` call

### Improving Format Detection

To improve detection accuracy:

1. **Edit detection methods** in `src/libs/document.ts`:
   - Make detection more robust (check multiple signatures)
   - Add fallback detection strategies
   - Improve MIME type checking

2. **Example enhancement**:
   ```typescript
   private async isEPUB(loader: any): Promise<boolean> {
     // Check for mimetype file containing "application/epub+zip"
     const mimetype = await loader.loadText('mimetype');
     return mimetype?.trim() === 'application/epub+zip';
   }
   ```

### Handling Large Documents

For performance improvements on large files:

1. **Lazy loading**: Modify `DocumentLoader` to support streaming
2. **Chunked parsing**: Process documents in chunks
3. **Background loading**: Use Web Workers (modify `@zip.js/zip.js` config)
   ```typescript
   configure({ useWebWorkers: true });
   ```

### Debugging Common Issues

**Problem**: Book fails to load
- Check `readerStore.viewStates[key].error` for error message
- Verify file is not corrupted (check `file.size`)
- Check browser console for import errors
- Verify pdf.js assets are present in `public/vendor/pdfjs/`

**Problem**: Incorrect format detected
- Add logging to detection methods in `DocumentLoader`
- Verify MIME type is set correctly on File object
- Check file extension handling

**Problem**: Missing TOC
- Check if `bookDoc.toc` is populated after `DocumentLoader.open()`
- Verify `updateTocID()` is called in `readerStore.initViewState()`
- Inspect format-specific TOC extraction in foliate-js

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add new format | `src/libs/document.ts` | `src/types/book.ts`, `packages/foliate-js/` |
| Fix metadata | `packages/foliate-js/{format}.js` | `src/services/appService.ts` |
| Improve detection | `src/libs/document.ts` | - |
| Performance tuning | `src/libs/document.ts` | `src/app/reader/components/FoliateViewer.tsx` |
| Cover extraction | `packages/foliate-js/{format}.js` | `src/services/appService.ts` |

## Dependencies

- **foliate-js**: Core reading engine (git submodule at `packages/foliate-js/`)
- **@zip.js/zip.js**: ZIP file parsing for EPUB/CBZ/FBZ
- **pdf.js**: PDF rendering (vendored in `public/vendor/pdfjs/`)
- **fflate**: MOBI decompression (imported from foliate-js vendor)

## Build Requirements

The pdf.js assets must be copied before running the app:

```bash
pnpm --filter @readest/readest-app setup-pdfjs
```

This runs scripts defined in `package.json`:
- `prepare-public-vendor`
- `copy-pdfjs-js`
- `copy-pdfjs-fonts`
- `copy-pdfjs-css`
