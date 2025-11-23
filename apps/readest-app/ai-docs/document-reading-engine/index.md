# Feature: Document Reading Engine

## Overview

The Document Reading Engine is the core functionality of Readest that enables loading, parsing, and rendering of multiple ebook and document formats. At commit `d757555f`, it supports EPUB, PDF, MOBI, CBZ (comic books), and FB2/FBZ formats through integration with the foliate-js library.

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

---

## Version 0.9.64 - 0.9.67 Updates (f5b686ab → 33b2ba16)

### PDF Custom Background Theming (v0.9.67, #1661)

**Major Feature**: Support for applying custom background colors and themes to PDF files.

**Overview**: Users can now customize PDF background colors, apply dark mode themes, and ensure consistent theming across all document formats (EPUB, PDF, MOBI, etc.).

**Problem Addressed**:
- PDFs traditionally have fixed background colors (usually white)
- Dark mode reading difficult with white PDF backgrounds
- Inconsistent theming between EPUB (customizable) and PDF (fixed)
- Eye strain from bright backgrounds in low-light conditions

**Implementation**:

**1. PDF.js Custom Layer** (`packages/foliate-js/pdf.js`):
```javascript
class PDFView {
  constructor(book, opts) {
    this.book = book;
    this.opts = opts;
    this.customBackground = opts.customBackground || null;
    this.invertColors = opts.invertColors || false;
  }

  renderPage(pageNum) {
    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');

    // Render PDF page
    const renderContext = {
      canvasContext: context,
      viewport: this.viewport
    };

    this.pdfPage.render(renderContext).promise.then(() => {
      // Apply custom background
      if (this.customBackground) {
        this.applyBackground(canvas, this.customBackground);
      }

      // Apply color inversion for dark mode
      if (this.invertColors) {
        this.invertCanvasColors(canvas);
      }
    });
  }

  applyBackground(canvas, backgroundColor) {
    const context = canvas.getContext('2d');
    const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;
    const bgColor = this.hexToRgb(backgroundColor);

    // Replace white background with custom color
    for (let i = 0; i < data.length; i += 4) {
      // Check if pixel is close to white (background)
      if (this.isBackgroundPixel(data[i], data[i+1], data[i+2])) {
        data[i] = bgColor.r;     // Red
        data[i+1] = bgColor.g;   // Green
        data[i+2] = bgColor.b;   // Blue
        // data[i+3] is alpha, keep unchanged
      }
    }

    context.putImageData(imageData, 0, 0);
  }

  isBackgroundPixel(r, g, b) {
    // Consider pixels close to white as background
    const threshold = 240;
    return r > threshold && g > threshold && b > threshold;
  }

  invertCanvasColors(canvas) {
    const context = canvas.getContext('2d');
    const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;

    for (let i = 0; i < data.length; i += 4) {
      data[i] = 255 - data[i];       // Invert red
      data[i+1] = 255 - data[i+1];   // Invert green
      data[i+2] = 255 - data[i+2];   // Invert blue
      // data[i+3] (alpha) unchanged
    }

    context.putImageData(imageData, 0, 0);
  }

  hexToRgb(hex) {
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
    return result ? {
      r: parseInt(result[1], 16),
      g: parseInt(result[2], 16),
      b: parseInt(result[3], 16)
    } : { r: 255, g: 255, b: 255 };
  }
}
```

**2. Settings Integration** (`src/types/settings.ts`):
```typescript
interface ViewSettings {
  // ...existing settings

  // PDF-specific theming
  pdfCustomBackground?: string;       // Hex color code
  pdfInvertColors?: boolean;          // Dark mode inversion
  pdfBrightness?: number;             // 0-100, default 100
  pdfContrast?: number;               // 0-200, default 100
}
```

**3. UI Controls** (`src/app/reader/components/settings/ThemePanel.tsx`):
```typescript
const PDFThemeSettings = () => {
  const { settings, updateSettings } = useSettings();
  const isPDF = useReaderStore(state => state.currentBook?.format === 'PDF');

  if (!isPDF) return null;

  return (
    <div className="pdf-theme-settings">
      <h3>PDF Theming</h3>

      <ColorPicker
        label="Background Color"
        value={settings.pdfCustomBackground}
        onChange={(color) => updateSettings({ pdfCustomBackground: color })}
      />

      <Toggle
        label="Invert Colors (Dark Mode)"
        checked={settings.pdfInvertColors}
        onChange={(checked) => updateSettings({ pdfInvertColors: checked })}
      />

      <Slider
        label="Brightness"
        min={0}
        max={100}
        value={settings.pdfBrightness || 100}
        onChange={(val) => updateSettings({ pdfBrightness: val })}
      />

      <Slider
        label="Contrast"
        min={0}
        max={200}
        value={settings.pdfContrast || 100}
        onChange={(val) => updateSettings({ pdfContrast: val })}
      />
    </div>
  );
};
```

**4. Apply Settings to PDF View** (`src/app/reader/components/FoliateViewer.tsx`):
```typescript
useEffect(() => {
  if (!view || book.format !== 'PDF') return;

  const pdfView = view as PDFView;
  pdfView.customBackground = viewSettings.pdfCustomBackground;
  pdfView.invertColors = viewSettings.pdfInvertColors;
  pdfView.brightness = viewSettings.pdfBrightness / 100;
  pdfView.contrast = viewSettings.pdfContrast / 100;

  // Re-render current page with new settings
  pdfView.render(pdfView.currentPage);
}, [viewSettings.pdfCustomBackground, viewSettings.pdfInvertColors,
    viewSettings.pdfBrightness, viewSettings.pdfContrast]);
```

**Features**:
- **Custom Background**: Choose any color for PDF background
- **Dark Mode**: Invert colors for dark mode reading
- **Brightness Control**: Adjust overall brightness
- **Contrast Control**: Enhance text readability
- **Theme Presets**: Predefined themes (Sepia, Dark, Light, etc.)
- **Per-Book Settings**: Each PDF can have different theme settings

**Theme Presets**:
```typescript
const PDF_THEME_PRESETS = {
  light: {
    pdfCustomBackground: '#ffffff',
    pdfInvertColors: false,
    pdfBrightness: 100,
    pdfContrast: 100
  },
  dark: {
    pdfCustomBackground: '#1a1a1a',
    pdfInvertColors: true,
    pdfBrightness: 80,
    pdfContrast: 110
  },
  sepia: {
    pdfCustomBackground: '#f4ecd8',
    pdfInvertColors: false,
    pdfBrightness: 95,
    pdfContrast: 105
  },
  night: {
    pdfCustomBackground: '#000000',
    pdfInvertColors: true,
    pdfBrightness: 70,
    pdfContrast: 120
  }
};
```

**Performance Considerations**:
- Canvas manipulation done on page render only
- Settings changes trigger re-render of current page
- Cached rendered pages invalidated on theme change
- Background workers can be used for intensive operations

**Limitations**:
- Image-heavy PDFs may take longer to process
- Very high contrast settings may affect image quality
- Color inversion may not work perfectly with all PDFs

**User Experience**:
- Settings panel shows PDF-specific controls when PDF is open
- Live preview of theme changes
- Smooth transitions between themes
- Settings saved per book

**Files**:
- `packages/foliate-js/pdf.js` - PDF rendering with theming
- `src/app/reader/components/settings/ThemePanel.tsx` - Theme UI
- `src/types/settings.ts` - Settings types
- `src/app/reader/components/FoliateViewer.tsx` - Settings application

---

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

---

## Version 0.9.79 - 0.9.82 Updates (cc3cc58d → e1691661)

### Fixed Layout EPUB Enhancements (v0.9.80, #1995)

**Major Feature**: Swipe gestures for paginating fixed-layout EPUB books.

**Overview**: Fixed-layout EPUBs (commonly used for graphic novels, manga, children's books, and illustrated content) now support swipe gestures for page navigation, matching the interaction model of dedicated comic book readers.

**Fixed Layout Detection**:
```typescript
// Detect if EPUB uses fixed layout
const isFixedLayout = (book: EPUBBook): boolean => {
  const metadata = book.package?.metadata;

  // Check rendition:layout property
  if (metadata?.layout === 'pre-paginated') {
    return true;
  }

  // Check META-INF/com.apple.ibooks.display-options.xml
  if (book.container?.ipadOrientation === 'landscape-only' ||
      book.container?.ipadOrientation === 'portrait-only') {
    return true;
  }

  return false;
};
```

**Swipe Implementation** (`packages/foliate-js/epub.js`):
```typescript
class FixedLayoutView {
  enableSwipeNavigation() {
    let touchStartX = 0;
    let touchEndX = 0;

    this.container.addEventListener('touchstart', (e) => {
      touchStartX = e.changedTouches[0].screenX;
    });

    this.container.addEventListener('touchend', (e) => {
      touchEndX = e.changedTouches[0].screenX;
      this.handleSwipe();
    });
  }

  handleSwipe() {
    const swipeThreshold = 50; // pixels
    const diff = touchStartX - touchEndX;

    if (Math.abs(diff) < swipeThreshold) return;

    if (diff > 0) {
      // Swipe left - next page
      this.nextPage();
    } else {
      // Swipe right - previous page
      this.previousPage();
    }
  }
}
```

**Features**:
- Horizontal swipe navigation (left/right for next/previous page)
- Configurable swipe sensitivity
- Prevents accidental navigation during zooming
- Works on touch devices (mobile, tablets)
- Respects reading direction (RTL support)

**User Experience**:
- Swipe left → Next page
- Swipe right → Previous page
- Tap center → Show/hide UI
- Pinch to zoom (swipe disabled while zoomed)
- Double-tap to fit page to screen

**Files**:
- `packages/foliate-js/epub.js` - Fixed layout handling
- `src/app/reader/hooks/useGestures.ts` - Gesture detection
- `src/app/reader/components/FixedLayoutViewer.tsx` - Fixed layout renderer

### PDF Zoom and Scaling Improvements (v0.9.79-0.9.82)

**Restore Scale Factor for PDFs** (v0.9.80, #1989):

**Problem**: PDF zoom level wasn't persisted across sessions.

**Solution**: Save and restore scale factor (zoom level) as part of book config.

**Implementation**:
```typescript
interface PDFConfig extends BookConfig {
  scaleFactor?: number;  // 0.5 = 50%, 1.0 = 100%, 2.0 = 200%, etc.
}

// Save zoom level when changed
const savePDFZoom = (scale: number) => {
  updateBookConfig(bookHash, {
    ...config,
    scaleFactor: scale
  });
};

// Restore zoom level on book open
const restorePDFZoom = (config: PDFConfig) => {
  if (config.scaleFactor) {
    pdfViewer.currentScale = config.scaleFactor;
  }
};
```

**Apply Zoom Shortcuts to PDFs** (v0.9.80, #2016):

**Feature**: Keyboard shortcuts for zooming PDFs.

**Shortcuts**:
- `Ctrl/Cmd +` or `Ctrl/Cmd =` → Zoom in
- `Ctrl/Cmd -` → Zoom out
- `Ctrl/Cmd 0` → Reset to 100%
- `Ctrl/Cmd 9` → Fit width
- `Ctrl/Cmd 8` → Fit height

**Implementation**:
```typescript
const handleZoomShortcut = (e: KeyboardEvent) => {
  if (!e.metaKey && !e.ctrlKey) return;

  switch(e.key) {
    case '+':
    case '=':
      e.preventDefault();
      pdfViewer.increaseScale();
      break;
    case '-':
      e.preventDefault();
      pdfViewer.decreaseScale();
      break;
    case '0':
      e.preventDefault();
      pdfViewer.currentScale = 1.0;
      break;
    case '9':
      e.preventDefault();
      pdfViewer.currentScaleValue = 'page-width';
      break;
    case '8':
      e.preventDefault();
      pdfViewer.currentScaleValue = 'page-height';
      break;
  }
};
```

**Disable Swipe When PDF Zoomed** (v0.9.80, #2068):

**Problem**: Swipe up gesture to toggle action bar interfered with panning zoomed PDFs.

**Solution**: Disable swipe-up gesture when PDF is zoomed in.

**Implementation**:
```typescript
const shouldAllowSwipeUp = (): boolean => {
  // Only allow swipe up to toggle action bar when PDF at 100% zoom
  if (isPDF && pdfViewer.currentScale > 1.0) {
    return false;
  }
  return true;
};

// Gesture handler
const handleSwipeUp = (e: TouchEvent) => {
  if (!shouldAllowSwipeUp()) {
    e.preventDefault();
    return;
  }

  toggleActionBar();
};
```

**Fixed Zoom Handling on iOS/macOS** (v0.9.79, #1978):

**Problem**: Zoom level would reset unexpectedly on iOS and macOS.

**Solution**: Properly handle viewport meta tag and WebKit zoom events.

**Implementation**:
```typescript
// Prevent iOS from auto-zooming
const preventAutoZoom = () => {
  const viewport = document.querySelector('meta[name=viewport]');
  if (viewport) {
    viewport.setAttribute('content',
      'width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no'
    );
  }
};

// Handle WebKit gesture events
document.addEventListener('gesturestart', (e) => {
  e.preventDefault();  // Prevent native zoom
});

document.addEventListener('gesturechange', (e) => {
  e.preventDefault();
  const scale = e.scale;
  applyCustomZoom(scale);
});
```

**Platform-Specific Behavior**:
- **iOS/macOS**: Uses WebKit gesture events for zoom
- **Android**: Uses touch events for pinch-to-zoom
- **Desktop**: Mouse wheel + Ctrl for zoom

### EPUB Font Size Scaling (v0.9.80, #2101)

**Feature**: Adjust font size directly instead of zooming HTML.

**Problem**: Using CSS zoom to change text size affected the entire layout, causing issues with images, margins, and fixed-position elements.

**Solution**: Change `font-size` property instead of `zoom`.

**Before**:
```typescript
// Old approach - zooms entire HTML
const changeFontSize = (multiplier: number) => {
  document.documentElement.style.zoom = `${multiplier * 100}%`;
};
```

**After**:
```typescript
// New approach - adjusts font size only
const changeFontSize = (multiplier: number) => {
  const baseFontSize = 16; // px
  const newSize = baseFontSize * multiplier;

  document.documentElement.style.fontSize = `${newSize}px`;
};
```

**Benefits**:
- Images maintain their original size
- Layout margins and padding stay proportional
- Fixed-position elements (headers, footers) work correctly
- Better compatibility with publisher CSS
- No blurry text from zoom scaling

**Implementation** (`packages/foliate-js/view.js`):
```typescript
class EPUBView {
  setFontSize(size: number) {
    // size: 0.5 to 2.0 (50% to 200%)
    const sizeInRem = size; // 1.0 = 16px by default

    this.iframe.contentDocument.documentElement.style.fontSize =
      `${sizeInRem}rem`;

    // Trigger reflow
    this.emit('relocated');
  }
}
```

**User Experience**:
- Font size slider in reader settings
- Live preview as slider moves
- Settings saved per book
- Separate from PDF zoom controls

**Files**:
- `packages/foliate-js/view.js` - Font size application
- `src/app/reader/components/settings/FontPanel.tsx` - Font size UI
- `src/types/settings.ts` - Font size config

---

## Performance Considerations
