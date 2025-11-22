# Readest Source Code Tree

This document outlines the hierarchical folder structure of the Readest project at commit `d757555f` (PWA: save a page reload between library and reader page navigation - January 23, 2025).

## Repository Root

```
readest/
├── .git/                          # Git version control
├── .vscode/                       # VSCode editor configuration
├── .github/                       # GitHub workflows and actions
├── apps/                          # Application packages (monorepo workspace)
│   └── readest-app/              # Main Next.js + Tauri application
│       ├── public/               # Static assets served by Next.js
│       │   ├── locales/          # i18n translation files (21 languages)
│       │   │   ├── de/translation.json
│       │   │   ├── el/translation.json
│       │   │   ├── en/translation.json
│       │   │   ├── es/translation.json
│       │   │   ├── fr/translation.json
│       │   │   ├── hi/translation.json
│       │   │   ├── id/translation.json
│       │   │   ├── it/translation.json
│       │   │   ├── ja/translation.json
│       │   │   ├── ko/translation.json
│       │   │   ├── pl/translation.json
│       │   │   ├── pt/translation.json
│       │   │   ├── ru/translation.json
│       │   │   ├── tr/translation.json
│       │   │   ├── uk/translation.json
│       │   │   ├── vi/translation.json
│       │   │   ├── zh-CN/translation.json
│       │   │   └── zh-TW/translation.json
│       │   ├── apple-touch-icon.png
│       │   ├── favicon.ico
│       │   ├── icon.png
│       │   └── manifest.json
│       ├── scripts/              # Build and deployment scripts
│       │   └── release-mac-appstore.sh
│       ├── src/                  # Application source code
│       │   ├── app/             # Next.js App Router pages
│       │   │   ├── layout.tsx   # Root layout with providers
│       │   │   ├── page.tsx     # Home page (redirects to library or reader)
│       │   │   ├── fonts/       # Custom fonts
│       │   │   │   ├── GeistVF.woff
│       │   │   │   └── GeistMonoVF.woff
│       │   │   ├── auth/        # Authentication pages
│       │   │   │   ├── page.tsx # Main auth page (OAuth login)
│       │   │   │   ├── callback/
│       │   │   │   │   └── page.tsx  # OAuth callback handler
│       │   │   │   └── error/
│       │   │   │       └── page.tsx  # Auth error page
│       │   │   ├── library/     # Library page and components
│       │   │   │   ├── page.tsx
│       │   │   │   ├── components/
│       │   │   │   │   ├── Bookshelf.tsx      # Book grid with nested groups
│       │   │   │   │   ├── LibraryHeader.tsx  # Header with search/filters
│       │   │   │   │   ├── ReadingProgress.tsx
│       │   │   │   │   └── SettingsMenu.tsx
│       │   │   │   └── hooks/
│       │   │   │       └── useDemoBooks.ts
│       │   │   └── reader/      # Reader page and components
│       │   │       ├── page.tsx
│       │   │       ├── components/
│       │   │       │   ├── Reader.tsx          # Root reader component
│       │   │       │   ├── ReaderContent.tsx   # Content wrapper with layout
│       │   │       │   ├── FoliateViewer.tsx   # Foliate-based viewer
│       │   │       │   ├── HeaderBar.tsx       # Top navigation bar
│       │   │       │   ├── FooterBar.tsx       # Bottom control bar
│       │   │       │   ├── Ribbon.tsx          # Bookmarks/highlights ribbon
│       │   │       │   ├── BookmarkToggler.tsx
│       │   │       │   ├── NotebookToggler.tsx
│       │   │       │   ├── SettingsToggler.tsx
│       │   │       │   ├── SidebarToggler.tsx
│       │   │       │   ├── ViewMenu.tsx
│       │   │       │   ├── PageInfo.tsx
│       │   │       │   ├── SectionInfo.tsx
│       │   │       │   ├── FootnotePopup.tsx   # Popover footnotes (Dec 2024)
│       │   │       │   ├── BooksGrid.tsx       # Grid of open books
│       │   │       │   ├── annotator/          # Text annotation features
│       │   │       │   │   ├── Annotator.tsx   # Main annotation handler
│       │   │       │   │   ├── AnnotationPopup.tsx
│       │   │       │   │   ├── HighlightOptions.tsx
│       │   │       │   │   ├── PopupButton.tsx
│       │   │       │   │   ├── DeepLPopup.tsx
│       │   │       │   │   ├── WikipediaPopup.tsx
│       │   │       │   │   └── WiktionaryPopup.tsx
│       │   │       │   ├── notebook/           # Note-taking interface
│       │   │       │   │   ├── Notebook.tsx
│       │   │       │   │   ├── Header.tsx
│       │   │       │   │   └── NoteEditor.tsx
│       │   │       │   ├── settings/           # Reader settings panels
│       │   │       │   │   ├── SettingsDialog.tsx
│       │   │       │   │   ├── DialogMenu.tsx
│       │   │       │   │   ├── FontPanel.tsx
│       │   │       │   │   ├── FontDropDown.tsx
│       │   │       │   │   ├── ColorPanel.tsx
│       │   │       │   │   ├── LayoutPanel.tsx
│       │   │       │   │   ├── MiscPanel.tsx
│       │   │       │   │   └── NumberInput.tsx
│       │   │       │   ├── sidebar/            # Reader sidebar components
│       │   │       │   │   ├── SideBar.tsx
│       │   │       │   │   ├── Header.tsx
│       │   │       │   │   ├── Content.tsx
│       │   │       │   │   ├── TabNavigation.tsx
│       │   │       │   │   ├── TOCView.tsx
│       │   │       │   │   ├── SearchBar.tsx
│       │   │       │   │   ├── SearchOptions.tsx
│       │   │       │   │   ├── SearchResults.tsx
│       │   │       │   │   ├── BookCard.tsx
│       │   │       │   │   ├── BookMenu.tsx
│       │   │       │   │   ├── BooknoteView.tsx
│       │   │       │   │   └── BooknoteItem.tsx
│       │   │       │   └── tts/                # Text-to-speech components
│       │   │       │       ├── TTSControl.tsx  # TTS playback controls
│       │   │       │       ├── TTSIcon.tsx     # TTS status icon
│       │   │       │       └── TTSPanel.tsx    # TTS settings panel
│       │   │       ├── hooks/                  # Reader-specific React hooks
│       │   │       │   ├── useAutoHideScrollbar.ts
│       │   │       │   ├── useBookShortcuts.ts
│       │   │       │   ├── useBooksManager.ts
│       │   │       │   ├── useClickEvent.ts
│       │   │       │   ├── useDragBar.ts
│       │   │       │   ├── useFoliateEvents.ts
│       │   │       │   ├── useNotesSync.ts
│       │   │       │   ├── useProgressSync.ts
│       │   │       │   ├── useScrollToItem.ts
│       │   │       │   └── useSidebar.ts
│       │   │       └── utils/                  # Reader utilities
│       │   │           └── iframeEventHandlers.ts
│       │   ├── components/              # Shared UI components
│       │   │   ├── Providers.tsx        # Root context providers wrapper
│       │   │   ├── AboutWindow.tsx
│       │   │   ├── Alert.tsx
│       │   │   ├── BookDetailModal.tsx  # Book details modal dialog
│       │   │   ├── Button.tsx
│       │   │   ├── Dialog.tsx           # Modal dialog component
│       │   │   ├── Dropdown.tsx
│       │   │   ├── MenuItem.tsx
│       │   │   ├── Popup.tsx
│       │   │   ├── Spinner.tsx
│       │   │   ├── Toast.tsx
│       │   │   └── WindowButtons.tsx
│       │   ├── context/                 # React context providers
│       │   │   ├── AuthContext.tsx      # Authentication state context
│       │   │   ├── EnvContext.tsx       # Environment configuration
│       │   │   ├── PHContext.tsx        # PostHog analytics context
│       │   │   └── SyncContext.tsx      # Cloud sync state context
│       │   ├── data/                    # Application data
│       │   │   └── demo/                # Demo library data
│       │   │       ├── library.en.json
│       │   │       └── library.zh.json
│       │   ├── helpers/                 # Helper utilities
│       │   │   ├── auth.ts              # Authentication helpers
│       │   │   ├── cli.ts               # CLI/desktop app helpers
│       │   │   ├── shortcuts.ts         # Keyboard shortcut definitions
│       │   │   └── updater.ts           # App update checking
│       │   ├── hooks/                   # Shared React hooks
│       │   │   ├── useResponsiveSize.ts
│       │   │   ├── useShortcuts.ts
│       │   │   ├── useSync.ts           # Cloud synchronization hook
│       │   │   ├── useTheme.ts
│       │   │   ├── useTrafficLight.ts
│       │   │   └── useTranslation.ts    # i18n translations hook
│       │   ├── i18n/                    # Internationalization
│       │   │   └── i18n.ts              # i18next configuration
│       │   ├── libs/                    # Core libraries
│       │   │   ├── document.ts          # Document loader (EPUB/PDF/MOBI/CBZ/FB2)
│       │   │   ├── edgeTTS.ts           # Edge TTS implementation
│       │   │   └── sync.ts              # Cloud sync library
│       │   ├── pages/                   # Next.js Pages Router (legacy/API)
│       │   │   ├── _app.tsx             # Next.js app wrapper
│       │   │   ├── api/                 # API routes
│       │   │   │   ├── sync.ts          # Cloud sync API endpoint
│       │   │   │   └── deepl/
│       │   │   │       └── translate.ts # DeepL translation API
│       │   │   └── reader/
│       │   │       └── [ids].tsx        # Dynamic reader page route
│       │   ├── services/                # Application services
│       │   │   ├── appService.ts        # Abstract app service base class
│       │   │   ├── webAppService.ts     # Web-specific implementation
│       │   │   ├── nativeAppService.ts  # Tauri native implementation
│       │   │   ├── environment.ts       # Environment configuration
│       │   │   ├── constants.ts         # Service constants
│       │   │   └── tts/                 # Text-to-speech services
│       │   │       ├── TTSController.ts # TTS orchestration controller
│       │   │       ├── TTSClient.ts     # Base TTS client interface
│       │   │       ├── WebSpeechClient.ts  # Web Speech API
│       │   │       ├── EdgeTTSClient.ts    # Edge TTS service
│       │   │       ├── TTSData.ts       # TTS data types
│       │   │       └── index.ts         # TTS service exports
│       │   ├── store/                   # Zustand state management stores
│       │   │   ├── readerStore.ts       # Reader view state and progress
│       │   │   ├── libraryStore.ts      # Book library state
│       │   │   ├── settingsStore.ts     # User settings
│       │   │   ├── sidebarStore.ts      # Sidebar UI state
│       │   │   ├── notebookStore.ts     # Annotations and notes
│       │   │   ├── bookDataStore.ts     # Book data caching
│       │   │   └── parallelViewStore.ts # Parallel reading view state
│       │   ├── styles/                  # Global styles
│       │   │   ├── globals.css
│       │   │   ├── fonts.css
│       │   │   └── themes.ts
│       │   ├── types/                   # TypeScript type definitions
│       │   │   ├── book.ts              # Book data types
│       │   │   ├── records.ts           # Reading progress and notes types
│       │   │   ├── settings.ts          # Settings data types
│       │   │   ├── system.ts            # System/environment types
│       │   │   └── view.ts              # UI view types
│       │   └── utils/                   # Utility functions
│       │       ├── book.ts              # Book processing utilities
│       │       ├── cors.ts              # CORS handling utilities
│       │       ├── css.ts               # CSS utilities
│       │       ├── event.ts             # Event handling utilities
│       │       ├── file.ts              # File operation utilities
│       │       ├── grid.ts              # Grid layout utilities
│       │       ├── lru.ts               # LRU cache implementation
│       │       ├── md5.ts               # MD5 hashing utility
│       │       ├── misc.ts              # Miscellaneous utilities
│       │       ├── nav.ts               # Navigation utilities
│       │       ├── os.ts                # OS detection utilities
│       │       ├── queue.ts             # Queue data structure
│       │       ├── sel.ts               # Text selection utilities
│       │       ├── serializer.ts        # Data serialization utilities
│       │       ├── ssml.ts              # SSML generation for TTS
│       │       ├── style.ts             # Style manipulation utilities
│       │       ├── supabase.ts          # Supabase client utilities
│       │       ├── toc.ts               # Table of contents utilities
│       │       ├── transform.ts         # Content transformation utilities
│       │       ├── ui.ts                # UI helper utilities
│       │       └── window.ts            # Window/desktop utilities
│       ├── src-tauri/                   # Tauri native shell (Rust)
│       │   ├── capabilities/            # Tauri security capabilities
│       │   │   ├── default.json         # Default security permissions
│       │   │   └── desktop.json         # Desktop-specific permissions
│       │   ├── icons/                   # Platform-specific app icons
│       │   │   ├── android/
│       │   │   │   ├── mipmap-hdpi/
│       │   │   │   ├── mipmap-mdpi/
│       │   │   │   ├── mipmap-xhdpi/
│       │   │   │   ├── mipmap-xxhdpi/
│       │   │   │   └── mipmap-xxxhdpi/
│       │   │   ├── ios/
│       │   │   ├── favicon.ico
│       │   │   ├── icon.png
│       │   │   └── icon.icns (macOS)
│       │   ├── src/                     # Rust source code
│       │   │   ├── main.rs              # Application entry point
│       │   │   ├── lib.rs               # Rust library definitions
│       │   │   ├── menu.rs              # Native menu definitions
│       │   │   └── traffic_light_plugin.rs  # macOS window controls
│       │   ├── build.rs                 # Tauri build script
│       │   ├── Cargo.toml               # Rust dependencies
│       │   └── tauri.conf.json          # Tauri configuration
│       ├── .env                         # Environment variables
│       ├── .env.local.example           # Example environment config
│       ├── .env.tauri                   # Tauri-specific env vars
│       ├── .env.web                     # Web-specific env vars
│       ├── eslint.config.mjs            # ESLint configuration
│       ├── i18next-scanner.config.js    # i18n scanner config
│       ├── next.config.mjs              # Next.js configuration
│       ├── package.json                 # NPM package configuration
│       ├── postcss.config.mjs           # PostCSS configuration
│       ├── tailwind.config.ts           # Tailwind CSS configuration
│       ├── tsconfig.json                # TypeScript configuration
│       ├── raw-loader.d.ts              # TypeScript declaration
│       ├── release-notes.json           # Release notes data
│       └── README.md
├── packages/                            # Vendored packages (git submodules)
│   ├── foliate-js/                     # Fork of foliate-js reading engine
│   │   [Git submodule: https://github.com/chrox/foliate-js.git]
│   └── tauri/                          # Patched Tauri crates
│       [Git submodule: https://github.com/chrox/tauri.git]
├── data/                                # Project data
│   └── screenshots/                     # Screenshot images
│       ├── annotations.png
│       ├── dark_mode.png
│       ├── deepl.png
│       ├── footnote_popover.png
│       ├── landing_preview.png
│       ├── theming_dark_mode.png
│       ├── tts_control.png
│       └── wikipedia_vertical.png
├── ops/                                 # Operations and deployment files
├── .editorconfig                        # Editor configuration
├── .gitignore
├── .gitmodules                          # Git submodules configuration
├── .prettierignore
├── .prettierrc.json                     # Prettier code formatter config
├── Cargo.lock                           # Rust lockfile
├── Cargo.toml                           # Workspace Rust dependencies
├── CONTRIBUTING.md
├── LICENSE                              # AGPL V3 License
├── README.md                            # Project README
├── package.json                         # Root package.json for monorepo
├── pnpm-lock.yaml                       # pnpm lockfile
├── pnpm-workspace.yaml                  # pnpm workspace configuration
└── tsconfig.json                        # TypeScript configuration
```

## Key Directory Descriptions

### `/apps/readest-app/src/app/`
Next.js App Router directory containing all pages and route-specific components. Uses file-based routing.

#### **`/apps/readest-app/src/app/auth/`**
OAuth authentication flow with Supabase:
- `page.tsx`: Main login page
- `callback/page.tsx`: OAuth callback handler
- `error/page.tsx`: Authentication error page

### `/apps/readest-app/src/app/library/`
Library interface for managing book collection:
- **Bookshelf.tsx**: Displays book grid with nested group support (feature added for organizing books)
- **LibraryHeader.tsx**: Search, filters, and view options
- **ReadingProgress.tsx**: Reading statistics and progress tracking
- **SettingsMenu.tsx**: Application settings access

### `/apps/readest-app/src/app/reader/components/`
Contains all reader UI components organized by feature:
- **annotator**: Text selection, highlighting, translation, dictionary lookup
- **notebook**: Note-taking and editing interface
- **settings**: Reader configuration panels (fonts, colors, layout, misc)
- **sidebar**: TOC, search, booknotes, bookmarks navigation
- **tts**: Text-to-speech controls and panel (NEW in this version)

Additional reader components:
- **FootnotePopup.tsx**: Popover footnotes feature (added Dec 2024) - displays footnotes inline without navigation
- **Reader.tsx**: Root reader component that orchestrates all reader features
- **SettingsToggler.tsx**: Toggle for settings dialog

### `/apps/readest-app/src/app/reader/hooks/`
Reader-specific React hooks:
- `useAutoHideScrollbar.ts`: Auto-hide scrollbar behavior (NEW)
- `useClickEvent.ts`: Click event handling (NEW)
- `useNotesSync.ts`: Synchronize notes to cloud (NEW)
- `useProgressSync.ts`: Synchronize reading progress (NEW)

### `/apps/readest-app/src/components/`
Shared UI components used across the application:
- **Providers.tsx**: Root context providers wrapper (NEW)
- **BookDetailModal.tsx**: Modal for displaying book details (NEW)
- **Dialog.tsx**: Reusable modal dialog component (NEW)

### `/apps/readest-app/src/context/`
React context providers for global state:
- **AuthContext.tsx**: Authentication state and user session
- **EnvContext.tsx**: Environment configuration (web vs native)
- **PHContext.tsx**: PostHog analytics integration
- **SyncContext.tsx**: Cloud synchronization state (NEW)

### `/apps/readest-app/src/data/`
Application data files:
- **demo/**: Demo library data for initial user experience
  - `library.en.json`: English demo books
  - `library.zh.json`: Chinese demo books

### `/apps/readest-app/src/helpers/`
Helper utilities for common operations:
- `auth.ts`: Authentication helpers for OAuth and session management (NEW)
- `cli.ts`: CLI and desktop app helpers (NEW)
- `shortcuts.ts`: Keyboard shortcut definitions
- `updater.ts`: App update checking utilities (NEW)

### `/apps/readest-app/src/hooks/`
Shared React hooks:
- `useResponsiveSize.ts`: Responsive sizing hook (NEW)
- `useSync.ts`: Cloud synchronization hook (NEW)
- `useTranslation.ts`: i18n translations hook (NEW)

### `/apps/readest-app/src/i18n/`
Internationalization configuration:
- `i18n.ts`: i18next setup with 21 language support

### `/apps/readest-app/src/libs/`
Core libraries:
- `document.ts`: Universal document loader supporting EPUB, PDF, MOBI, CBZ, FB2/FBZ formats
- `edgeTTS.ts`: Edge TTS implementation (NEW)
- `sync.ts`: Cloud sync library (NEW)

### `/apps/readest-app/src/pages/`
Next.js Pages Router (for API routes and legacy pages):
- **api/sync.ts**: Cloud synchronization API endpoint (NEW)
- **api/deepl/translate.ts**: DeepL translation API proxy
- **reader/[ids].tsx**: Dynamic reader page route for multi-view reading

### `/apps/readest-app/src/services/`
Platform abstraction layer:
- `environment.ts`: Determines native vs web environment
- `appService.ts`: Abstract base class for app services
- `nativeAppService.ts`: Tauri native implementation
- `webAppService.ts`: Web-specific implementation (NEW)

#### **`/apps/readest-app/src/services/tts/`** (NEW)
Text-to-speech service architecture:
- **TTSController.ts**: Orchestrates TTS operations across different engines
- **TTSClient.ts**: Base interface for TTS clients
- **WebSpeechClient.ts**: Browser Web Speech API implementation
- **EdgeTTSClient.ts**: Microsoft Edge TTS service implementation
- **TTSData.ts**: TTS data types and voice configurations
- **index.ts**: Service exports

### `/apps/readest-app/src/store/`
Zustand stores managing application state:
- `readerStore.ts`: Multi-view reader state and progress
- `settingsStore.ts`: System-wide settings
- `libraryStore.ts`: Book library state
- `bookDataStore.ts`: Loaded book data and configs
- `notebookStore.ts`: Note-taking state
- `sidebarStore.ts`: Sidebar panel state
- `parallelViewStore.ts`: Parallel reading view state (NEW - for side-by-side reading)

### `/apps/readest-app/src/types/`
TypeScript type definitions:
- `book.ts`: Book data structures
- `records.ts`: Reading progress and notes types (NEW)
- `settings.ts`: Settings data types
- `system.ts`: System/environment types
- `view.ts`: UI view types (NEW)

### `/apps/readest-app/src/utils/`
Comprehensive utility functions:
- `book.ts`: Book processing and metadata
- `cors.ts`: CORS handling utilities (NEW)
- `css.ts`: CSS utilities (NEW)
- `event.ts`: Event handling utilities
- `file.ts`: File operation utilities
- `grid.ts`: Grid layout utilities
- `lru.ts`: LRU cache implementation (NEW - performance optimization)
- `md5.ts`: MD5 hashing utility
- `misc.ts`: Miscellaneous utilities
- `nav.ts`: Navigation utilities (NEW)
- `os.ts`: OS detection utilities (NEW)
- `queue.ts`: Queue data structure (NEW)
- `sel.ts`: Text selection utilities
- `serializer.ts`: Data serialization utilities (NEW)
- `ssml.ts`: SSML generation for TTS (NEW)
- `style.ts`: Style manipulation utilities
- `supabase.ts`: Supabase client utilities (NEW - cloud sync)
- `toc.ts`: Table of contents utilities
- `transform.ts`: Content transformation utilities (NEW)
- `ui.ts`: UI helper utilities (NEW)
- `window.ts`: Window/desktop utilities (NEW)

### `/apps/readest-app/src-tauri/`
Rust-based Tauri native shell providing:
- Native filesystem access
- Platform-specific window decorations
- Menu integration
- System integrations (plugins)

**Source files:**
- `main.rs`: Application entry point with window setup
- `lib.rs`: Rust library definitions and Tauri commands
- `menu.rs`: Native menu creation (NEW)
- `traffic_light_plugin.rs`: macOS window controls

**Configuration:**
- `tauri.conf.json`: App configuration, permissions, and features
- `capabilities/`: Granular security permission definitions
  - `default.json`: Default permissions
  - `desktop.json`: Desktop-specific permissions

### `/packages/`
Git submodules for vendored dependencies:
- `foliate-js`: Customized ebook reading engine (forked for Readest-specific features)
- `tauri`: Patched Tauri framework crates

### `/public/locales/`
i18n translation files for 21 languages:
- German (de), Greek (el), English (en), Spanish (es), French (fr)
- Hindi (hi), Indonesian (id), Italian (it), Japanese (ja), Korean (ko)
- Polish (pl), Portuguese (pt), Russian (ru), Turkish (tr), Ukrainian (uk)
- Vietnamese (vi), Chinese Simplified (zh-CN), Chinese Traditional (zh-TW)

## Build Artifacts (Not in Repo)

The following directories are generated during build and are git-ignored:
- `node_modules/` - NPM dependencies
- `.next/` - Next.js build output
- `dist/` - Production build artifacts
- `target/` - Rust build artifacts
- `build/` - Intermediate build files
- `khana/` - Build/cache directory

## Configuration Files

### Root Configuration
- `pnpm-workspace.yaml`: Defines monorepo workspace packages
- `tsconfig.json`: TypeScript compiler configuration
- `.prettierrc.json`: Code formatting rules
- `Cargo.toml`: Workspace Rust dependencies
- `.editorconfig`: Editor configuration

### Application Configuration
- `apps/readest-app/next.config.mjs`: Next.js configuration
- `apps/readest-app/tailwind.config.ts`: Tailwind CSS configuration
- `apps/readest-app/eslint.config.mjs`: ESLint configuration
- `apps/readest-app/postcss.config.mjs`: PostCSS configuration
- `apps/readest-app/i18next-scanner.config.js`: i18n scanner config
- `apps/readest-app/src-tauri/tauri.conf.json`: Tauri app configuration
- `apps/readest-app/src-tauri/Cargo.toml`: Rust dependencies
- `apps/readest-app/package.json`: App-specific NPM scripts and dependencies

### Environment Files
- `.env`: Shared environment variables
- `.env.tauri`: Tauri-specific environment variables
- `.env.web`: Web-specific environment variables
- `.env.local.example`: Example for local configuration

## Key Features by Directory

### Authentication & Cloud Sync
- `src/app/auth/`: OAuth authentication pages
- `src/context/AuthContext.tsx` & `SyncContext.tsx`: State management
- `src/helpers/auth.ts`: Authentication helpers
- `src/libs/sync.ts`: Sync library
- `src/pages/api/sync.ts`: Sync API endpoint
- `src/utils/supabase.ts`: Supabase client

### Text-to-Speech
- `src/app/reader/components/tts/`: UI controls
- `src/services/tts/`: Service layer with multiple TTS engines
- `src/libs/edgeTTS.ts`: Edge TTS library
- `src/utils/ssml.ts`: SSML generation

### Internationalization
- `src/i18n/i18n.ts`: Configuration
- `public/locales/`: 21 language translations
- `src/hooks/useTranslation.ts`: Translation hook
- `i18next-scanner.config.js`: Translation extraction

### Parallel Reading
- `src/store/parallelViewStore.ts`: State management
- `src/app/reader/components/BooksGrid.tsx`: Multi-book layout
- `src/pages/reader/[ids].tsx`: Dynamic multi-view routing

### Footnote Popover
- `src/app/reader/components/FootnotePopup.tsx`: Popup component
- `src/components/Popup.tsx`: Reusable popup with directional triangles
- `src/utils/sel.ts`: Position calculation utilities
- Feature documented in `feature-annotation-system.md`

## Technology Stack

- **Frontend**: Next.js 14+ with React 19
- **Desktop**: Tauri 2.1+ (Rust backend)
- **State Management**: Zustand
- **Styling**: Tailwind CSS
- **Database**: Supabase (cloud sync)
- **Internationalization**: i18next (21 languages)
- **E-book Viewer**: foliate-js (custom fork)
- **Build**: pnpm monorepo
- **TTS**: Web Speech API & Microsoft Edge TTS
