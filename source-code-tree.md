# Readest Source Code Tree

This document outlines the hierarchical folder structure of the Readest project at commit `571baf98` (Add About Readest window).

## Repository Root

```
readest/
├── .git/                          # Git version control
├── .vscode/                       # VSCode editor configuration
├── apps/                          # Application packages (monorepo workspace)
│   └── readest-app/              # Main Next.js + Tauri application
│       ├── public/               # Static assets served by Next.js
│       ├── scripts/              # Build and deployment scripts
│       ├── src/                  # Application source code
│       │   ├── app/             # Next.js App Router pages
│       │   │   ├── fonts/       # Custom fonts
│       │   │   ├── library/     # Library page and components
│       │   │   │   ├── components/
│       │   │   │   │   ├── Bookshelf.tsx
│       │   │   │   │   └── LibraryHeader.tsx
│       │   │   │   └── page.tsx
│       │   │   ├── reader/      # Reader page and components
│       │   │   │   ├── components/
│       │   │   │   │   ├── annotator/     # Text annotation features
│       │   │   │   │   │   ├── AnnotationPopup.tsx
│       │   │   │   │   │   ├── Annotator.tsx
│       │   │   │   │   │   ├── DeepLPopup.tsx
│       │   │   │   │   │   ├── HighlightOptions.tsx
│       │   │   │   │   │   ├── PopupButton.tsx
│       │   │   │   │   │   ├── WikipediaPopup.tsx
│       │   │   │   │   │   └── WiktionaryPopup.tsx
│       │   │   │   │   ├── notebook/       # Note-taking interface
│       │   │   │   │   │   ├── Header.tsx
│       │   │   │   │   │   ├── NoteEditor.tsx
│       │   │   │   │   │   └── Notebook.tsx
│       │   │   │   │   ├── settings/       # Reader settings panels
│       │   │   │   │   │   ├── ColorPanel.tsx
│       │   │   │   │   │   ├── DialogMenu.tsx
│       │   │   │   │   │   ├── FontDropDown.tsx
│       │   │   │   │   │   ├── FontPanel.tsx
│       │   │   │   │   │   ├── LayoutPanel.tsx
│       │   │   │   │   │   ├── MiscPanel.tsx
│       │   │   │   │   │   ├── NumberInput.tsx
│       │   │   │   │   │   └── SettingsDialog.tsx
│       │   │   │   │   ├── sidebar/        # Reader sidebar components
│       │   │   │   │   │   ├── BookCard.tsx
│       │   │   │   │   │   ├── BookMenu.tsx
│       │   │   │   │   │   ├── BooknoteItem.tsx
│       │   │   │   │   │   ├── BooknoteView.tsx
│       │   │   │   │   │   ├── Content.tsx
│       │   │   │   │   │   ├── Header.tsx
│       │   │   │   │   │   ├── SearchBar.tsx
│       │   │   │   │   │   ├── SearchOptions.tsx
│       │   │   │   │   │   ├── SearchResults.tsx
│       │   │   │   │   │   ├── SideBar.tsx
│       │   │   │   │   │   ├── TOCView.tsx
│       │   │   │   │   │   └── TabNavigation.tsx
│       │   │   │   │   ├── BookmarkToggler.tsx
│       │   │   │   │   ├── BooksGrid.tsx
│       │   │   │   │   ├── FoliateViewer.tsx
│       │   │   │   │   ├── FooterBar.tsx
│       │   │   │   │   ├── HeaderBar.tsx
│       │   │   │   │   ├── NotebookToggler.tsx
│       │   │   │   │   ├── PageInfo.tsx
│       │   │   │   │   ├── ReaderContent.tsx
│       │   │   │   │   ├── Ribbon.tsx
│       │   │   │   │   ├── SectionInfo.tsx
│       │   │   │   │   ├── SidebarToggler.tsx
│       │   │   │   │   └── ViewMenu.tsx
│       │   │   │   ├── hooks/           # Reader-specific React hooks
│       │   │   │   │   ├── useBookShortcuts.ts
│       │   │   │   │   ├── useBooksManager.ts
│       │   │   │   │   ├── useDragBar.ts
│       │   │   │   │   ├── useFoliateEvents.ts
│       │   │   │   │   ├── useScrollToItem.ts
│       │   │   │   │   └── useSidebar.ts
│       │   │   │   ├── utils/           # Reader utilities
│       │   │   │   │   └── iframeEventHandlers.ts
│       │   │   │   └── page.tsx
│       │   │   ├── layout.tsx           # Root layout with providers
│       │   │   └── page.tsx             # Home page
│       │   ├── components/              # Shared UI components
│       │   │   ├── AboutWindow.tsx
│       │   │   ├── Alert.tsx
│       │   │   ├── Button.tsx
│       │   │   ├── Dropdown.tsx
│       │   │   ├── MenuItem.tsx
│       │   │   ├── Popup.tsx
│       │   │   ├── Spinner.tsx
│       │   │   ├── Toast.tsx
│       │   │   └── WindowButtons.tsx
│       │   ├── context/                 # React context providers
│       │   │   ├── AuthContext.tsx
│       │   │   ├── EnvContext.tsx
│       │   │   └── PHContext.tsx
│       │   ├── helpers/                 # Helper utilities
│       │   │   └── shortcuts.ts
│       │   ├── hooks/                   # Shared React hooks
│       │   │   ├── useShortcuts.ts
│       │   │   ├── useTheme.ts
│       │   │   └── useTrafficLight.ts
│       │   ├── libs/                    # Core libraries
│       │   │   └── document.ts          # Document loader for EPUB/PDF/MOBI/CBZ/FB2
│       │   ├── services/                # Application services
│       │   │   ├── appService.ts        # Abstract AppService base class
│       │   │   ├── constants.ts         # Service constants
│       │   │   ├── environment.ts       # Environment configuration
│       │   │   └── nativeAppService.ts  # Tauri native implementation
│       │   ├── store/                   # Zustand state management stores
│       │   │   ├── bookDataStore.ts     # Book data and configs
│       │   │   ├── libraryStore.ts      # Library state
│       │   │   ├── notebookStore.ts     # Notebook state
│       │   │   ├── readerStore.ts       # Reader view states
│       │   │   ├── settingsStore.ts     # System settings
│       │   │   └── sidebarStore.ts      # Sidebar state
│       │   ├── styles/                  # Global styles
│       │   │   ├── fonts.css
│       │   │   ├── globals.css
│       │   │   └── themes.ts
│       │   ├── types/                   # TypeScript type definitions
│       │   │   ├── book.ts
│       │   │   ├── settings.ts
│       │   │   └── system.ts
│       │   └── utils/                   # Utility functions
│       │       ├── book.ts
│       │       ├── event.ts
│       │       ├── file.ts
│       │       ├── grid.ts
│       │       ├── md5.ts
│       │       ├── misc.ts
│       │       ├── sel.ts
│       │       ├── style.ts
│       │       └── toc.ts
│       ├── src-tauri/                   # Tauri native shell (Rust)
│       │   ├── capabilities/            # Tauri security capabilities
│       │   ├── icons/                   # Platform-specific app icons
│       │   │   ├── android/
│       │   │   │   ├── mipmap-hdpi/
│       │   │   │   ├── mipmap-mdpi/
│       │   │   │   ├── mipmap-xhdpi/
│       │   │   │   ├── mipmap-xxhdpi/
│       │   │   │   └── mipmap-xxxhdpi/
│       │   │   └── ios/
│       │   ├── src/                     # Rust source code
│       │   │   ├── lib.rs              # Main Tauri setup and plugins
│       │   │   ├── main.rs             # Application entry point
│       │   │   └── tauri_traffic_light_positioner_plugin.rs
│       │   ├── Cargo.toml              # Rust dependencies
│       │   └── tauri.conf.json         # Tauri configuration
│       ├── package.json                 # NPM package configuration
│       └── README.md
├── packages/                            # Vendored packages (git submodules)
│   ├── foliate-js/                     # Fork of foliate-js reading engine
│   └── tauri/                          # Patched Tauri crates
├── .gitignore
├── .gitmodules                          # Git submodules configuration
├── .prettierignore
├── .prettierrc.json                     # Prettier code formatter config
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

### `/apps/readest-app/src/app/reader/components/`
Contains all reader UI components organized by feature:
- **annotator**: Text selection, highlighting, translation, dictionary lookup
- **notebook**: Note-taking and editing interface
- **settings**: Reader configuration panels (fonts, colors, layout, misc)
- **sidebar**: TOC, search, booknotes, bookmarks navigation

### `/apps/readest-app/src/services/`
Platform abstraction layer:
- `environment.ts`: Determines native vs web environment
- `appService.ts`: Abstract base class for app services
- `nativeAppService.ts`: Tauri native implementation

### `/apps/readest-app/src/store/`
Zustand stores managing application state:
- `readerStore.ts`: Multi-view reader state and progress
- `settingsStore.ts`: System-wide settings
- `libraryStore.ts`: Book library state
- `bookDataStore.ts`: Loaded book data and configs
- `notebookStore.ts`: Note-taking state
- `sidebarStore.ts`: Sidebar panel state

### `/apps/readest-app/src/libs/`
Core libraries:
- `document.ts`: Universal document loader supporting EPUB, PDF, MOBI, CBZ, FB2/FBZ formats

### `/apps/readest-app/src-tauri/`
Rust-based Tauri native shell providing:
- Native filesystem access
- Platform-specific window decorations
- Menu integration
- System integrations (plugins)

### `/packages/`
Git submodules for vendored dependencies:
- `foliate-js`: Customized ebook reading engine
- `tauri`: Patched Tauri framework crates

## Build Artifacts (Not in Repo)

The following directories are generated during build and are git-ignored:
- `node_modules/` - NPM dependencies
- `.next/` - Next.js build output
- `dist/` - Production build artifacts
- `target/` - Rust build artifacts
- `build/` - Intermediate build files

## Configuration Files

- `pnpm-workspace.yaml`: Defines monorepo workspace packages
- `tsconfig.json`: TypeScript compiler configuration
- `.prettierrc.json`: Code formatting rules
- `apps/readest-app/src-tauri/tauri.conf.json`: Tauri app configuration
- `apps/readest-app/src-tauri/Cargo.toml`: Rust dependencies
- `apps/readest-app/package.json`: App-specific NPM scripts and dependencies
