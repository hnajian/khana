# Readest Codebase Documentation Index

**Version**: Documentation for commits `571baf98` through `33b2ba16` (0.8.5 → 0.9.67 releases)
**Last Updated**: November 2025

## Introduction

This documentation provides a comprehensive guide for AI coding agents working with the Readest codebase. Readest is a cross-platform ebook reader built with **Next.js 15** and **Tauri v2**, supporting EPUB, PDF, MOBI, CBZ, and FB2/FBZ formats.

## Quick Start for AI Agents

### Understanding the Project

1. **Read the overview** (below) to understand the architecture
2. **Consult [source-code-tree.md](./source-code-tree.md)** for the folder structure
3. **Find your feature** in the feature documentation list below
4. **Locate entry points** using the "Common Tasks Quick Reference" section

### Project Architecture Overview

Readest is structured as a **pnpm monorepo** with these key characteristics:

- **Frontend**: Next.js 15 with App Router, React 18, Tailwind CSS, DaisyUI
- **Native Shell**: Tauri v2 (Rust) for desktop apps (macOS, Windows, Linux)
- **Reading Engine**: foliate-js (vendored fork in `packages/foliate-js`)
- **State Management**: Zustand stores for reactive state
- **Build System**: pnpm workspaces, Next.js build, Tauri build

**Repository Structure**:
```
readest/
├── apps/readest-app/          # Main application
│   ├── src/                   # Next.js/React source
│   └── src-tauri/            # Rust native shell
├── packages/                  # Vendored dependencies (git submodules)
│   ├── foliate-js/           # Reading engine fork
│   └── tauri/                # Patched Tauri crates
└── [config files]
```

## Documentation Files

### Core Documentation

- **[source-code-tree.md](./source-code-tree.md)** - Complete hierarchical folder structure with descriptions
- **[index.md](./index.md)** (this file) - Navigation hub for all documentation

### Feature Documentation

Each feature document contains:
- Overview of the feature
- Key components and file locations
- Architecture and data flow
- AI agent modification guidelines
- Entry points for common tasks
- Debugging tips

| Feature | File | Description |
|---------|------|-------------|
| **Document Reading Engine** | [feature-document-reading-engine.md](./feature-document-reading-engine.md) | EPUB/PDF/MOBI/CBZ/FB2 support, format detection, book loading |
| **Annotation System** | [feature-annotation-system.md](./feature-annotation-system.md) | Highlighting, notes, translation, dictionary integration, **popover footnotes** |
| **Sidebar Navigation** | [feature-sidebar-navigation.md](./feature-sidebar-navigation.md) | TOC, search, bookmarks, booknotes views |
| **Settings System** | [feature-settings-system.md](./feature-settings-system.md) | Three-tier settings hierarchy (global/book/view) |
| **State Management** | [feature-state-management.md](./feature-state-management.md) | Zustand stores architecture and patterns |
| **Library Management** | [feature-library-management.md](./feature-library-management.md) | Book import/export, metadata, cover handling |
| **Cross-Platform Support** | [feature-cross-platform-support.md](./feature-cross-platform-support.md) | Tauri native shell, **web platform support**, **PWA enhancements**, platform-specific code |
| **Authentication & Sync** | [feature-auth-sync.md](./feature-auth-sync.md) | User accounts, OAuth, cloud sync for progress and notes |
| **Internationalization** | [feature-internationalization.md](./feature-internationalization.md) | 14 languages, i18next framework, translation management |
| **Text-to-Speech (TTS)** | [feature-text-to-speech.md](./feature-text-to-speech.md) | Dual backend (Web Speech + Edge TTS), 100+ voices, audio preloading |
| **Translation System** | [translation-system/index.md](./translation-system/index.md) | **NEW**: Multi-provider translation (DeepL, Azure, Google, Yandex), two-tier caching, quota management |

## Common Tasks Quick Reference

### Adding Support for a New Book Format

**Documents to consult**: [feature-document-reading-engine.md](./feature-document-reading-engine.md)

**Primary files to modify**:
1. `src/types/book.ts` - Add format to `BookFormat` type
2. `src/libs/document.ts` - Add format detection and loader
3. `packages/foliate-js/` - Implement format parser (if needed)

**See**: "Adding Support for a New Format" section in document-reading-engine.md

### Adding a New Reader Setting

**Documents to consult**: [feature-settings-system.md](./feature-settings-system.md)

**Primary files to modify**:
1. `src/types/settings.ts` - Add to `SystemSettings` interface
2. `src/types/book.ts` - Add to `ViewSettings` interface
3. `src/app/reader/components/settings/*Panel.tsx` - Add UI control
4. `src/app/reader/components/FoliateViewer.tsx` - Apply setting to view

**See**: "Adding a New Setting" section in settings-system.md

### Implementing a New Annotation Action

**Documents to consult**: [feature-annotation-system.md](./feature-annotation-system.md)

**Primary files to modify**:
1. `src/app/reader/components/annotator/Annotator.tsx` - Add button/handler
2. Create new popup component (if needed) following `WikipediaPopup.tsx` pattern
3. `src/store/notebookStore.ts` - Update if storing new data

**See**: "Adding a New Annotation Action" section in annotation-system.md

### Adding a Sidebar Tab

**Documents to consult**: [feature-sidebar-navigation.md](./feature-sidebar-navigation.md)

**Primary files to modify**:
1. `src/store/sidebarStore.ts` - Add tab to type definition
2. `src/app/reader/components/sidebar/TabNavigation.tsx` - Add tab button
3. `src/app/reader/components/sidebar/Content.tsx` - Add tab content
4. Create new view component following `TOCView.tsx` pattern

**See**: "Adding a New Sidebar Tab" section in sidebar-navigation.md

### Adding a Tauri Plugin or Native Feature

**Documents to consult**: [feature-cross-platform-support.md](./feature-cross-platform-support.md)

**Primary files to modify**:
1. `apps/readest-app/src-tauri/Cargo.toml` - Add dependency
2. `apps/readest-app/src-tauri/src/lib.rs` - Register plugin
3. `src/services/nativeAppService.ts` - Use plugin APIs in TypeScript

**See**: "Adding a New Tauri Plugin" section in cross-platform-support.md

### Creating a New Zustand Store

**Documents to consult**: [feature-state-management.md](./feature-state-management.md)

**Primary files**:
1. Create `src/store/yourStore.ts` following existing store patterns
2. Import and use in components with `useYourStore(state => state.value)`

**See**: "Creating a New Store" section in state-management.md

### Implementing Search/Filter in Library

**Documents to consult**: [feature-library-management.md](./feature-library-management.md)

**Primary files to modify**:
1. `src/store/libraryStore.ts` - Add search state and filter logic
2. `src/app/library/components/LibraryHeader.tsx` - Add search UI
3. `src/app/library/components/Bookshelf.tsx` - Use filtered results

**See**: "Adding Book Search/Filter" section in library-management.md

## Folder Structure Quick Reference

### Frontend Code (`apps/readest-app/src/`)

```
src/
├── app/                   # Next.js App Router
│   ├── layout.tsx        # Root layout with providers
│   ├── page.tsx          # Home page
│   ├── library/          # Library page and components
│   └── reader/           # Reader page and components
│       ├── components/
│       │   ├── annotator/    # Annotation features
│       │   ├── notebook/     # Note-taking
│       │   ├── settings/     # Reader settings
│       │   └── sidebar/      # Navigation sidebar
│       ├── hooks/            # Reader-specific hooks
│       └── utils/            # Reader utilities
├── components/           # Shared UI components
├── context/             # React context providers
├── hooks/               # Shared hooks
├── libs/                # Core libraries (document loader)
├── services/            # AppService abstraction layer
├── store/               # Zustand state stores
├── styles/              # Global styles
├── types/               # TypeScript type definitions
└── utils/               # Utility functions
```

### Native Shell (`apps/readest-app/src-tauri/`)

```
src-tauri/
├── src/
│   ├── lib.rs           # Tauri setup and plugin registration
│   ├── main.rs          # Application entry point
│   └── *.rs             # Additional Rust modules
├── Cargo.toml           # Rust dependencies
└── tauri.conf.json      # Tauri configuration
```

### Key File Purposes

| File Path | Purpose |
|-----------|---------|
| `src/app/layout.tsx` | Root layout, providers, global setup |
| `src/services/environment.ts` | Platform detection, AppService factory |
| `src/services/appService.ts` | Abstract base for app services |
| `src/services/nativeAppService.ts` | Tauri implementation of AppService |
| `src/libs/document.ts` | Universal document loader |
| `src/store/*.ts` | Zustand state stores |
| `src/types/*.ts` | TypeScript type definitions |
| `src-tauri/src/lib.rs` | Tauri plugin setup and window config |

## Development Workflow

### Setting Up Development Environment

```bash
# Clone with submodules
git clone https://github.com/chrox/readest.git
cd readest
git submodule update --init --recursive

# Install dependencies
npm install -g pnpm
pnpm install

# Setup pdf.js assets
pnpm --filter @readest/readest-app setup-pdfjs

# Run development server
pnpm tauri dev          # Native app with hot reload
```

### Build Commands

```bash
# Development
pnpm tauri dev

# Production builds
pnpm tauri build                      # Current platform
pnpm build-macos-universal           # macOS universal
pnpm build-win-x64                   # Windows x64
pnpm build-linux-x64                 # Linux x64
```

### Key NPM Scripts

- `pnpm dev` - Next.js dev server only (web)
- `pnpm tauri dev` - Tauri + Next.js (native app)
- `pnpm build` - Next.js production build
- `pnpm tauri build` - Native app production build
- `pnpm setup-pdfjs` - Copy pdf.js assets to public directory

## Technology Stack

### Frontend
- **Framework**: Next.js 15 (App Router)
- **UI Library**: React 18
- **Styling**: Tailwind CSS, DaisyUI
- **State**: Zustand
- **Icons**: react-icons
- **Book Rendering**: foliate-js (vendored)

### Native Shell
- **Framework**: Tauri v2
- **Language**: Rust
- **Plugins**: fs, dialog, http, os, log, shell

### Build Tools
- **Package Manager**: pnpm (workspaces)
- **TypeScript**: v5
- **Bundler**: Next.js (frontend), Tauri (native)

## Important Patterns and Conventions

### State Management Pattern

All stores follow this pattern:
```typescript
interface StoreInterface {
  // State properties
  value: Type;

  // Actions
  setValue: (val: Type) => void;
}

export const useStore = create<StoreInterface>((set, get) => ({
  value: initialValue,
  setValue: (val) => set({ value: val }),
}));
```

### Component Organization

- **Pages**: `src/app/{page}/page.tsx`
- **Page Components**: `src/app/{page}/components/`
- **Shared Components**: `src/components/`
- **Feature Hooks**: Colocated with feature components
- **Shared Hooks**: `src/hooks/`

### File Naming

- **React Components**: PascalCase (e.g., `BookCard.tsx`)
- **Hooks**: camelCase with `use` prefix (e.g., `useSidebar.ts`)
- **Utilities**: camelCase (e.g., `book.ts`, `toc.ts`)
- **Stores**: camelCase with `Store` suffix (e.g., `readerStore.ts`)
- **Types**: camelCase (e.g., `book.ts`, `settings.ts`)

### Settings Hierarchy

Always remember the three-tier hierarchy:
1. **Global Settings** (SystemSettings) - Defaults for all books
2. **Book Settings** (BookConfig.viewSettings) - Per-book overrides
3. **View Settings** (ViewState.viewSettings) - Per-view overrides

## Debugging Tips

### Common Issues

1. **Book won't load**:
   - Check `DocumentLoader` in `src/libs/document.ts`
   - Verify format detection
   - Check foliate-js console errors

2. **Settings not applying**:
   - Verify settings hierarchy (global < book < view)
   - Check `FoliateViewer` useEffect dependencies
   - Ensure `appService.saveSettings()` is called

3. **Annotations disappearing**:
   - Check `notebookStore` persistence
   - Verify book hash consistency
   - Check `bookDataStore` save operations

4. **Tauri command fails**:
   - Verify command is registered in `invoke_handler`
   - Check function signature matches TypeScript call
   - Review Tauri console output

### Development Tools

- **React DevTools**: Inspect component state
- **Zustand DevTools**: Monitor store changes (add middleware)
- **Tauri DevTools**: Native debugging via Rust/WebView console
- **Browser Console**: Next.js and React errors

## External Resources

- **Readest README**: `README.md` in repository root
- **Tauri Documentation**: https://v2.tauri.app/
- **Next.js Documentation**: https://nextjs.org/docs
- **foliate-js**: Vendored in `packages/foliate-js/`

## Extending This Documentation

When adding new features, create a feature documentation file following this structure:

1. **Overview** - What the feature does
2. **Key Components** - Files and their purposes
3. **Architecture** - How it works
4. **AI Agent Modification Guidelines** - Step-by-step instructions for common tasks
5. **Entry Points** - Table of tasks and files
6. **Common Issues** - Debugging help
7. **Dependencies** - External libraries used

Add the new file to the "Feature Documentation" table above.

## Navigation Guide for AI Agents

### I need to modify the reader view
→ See [feature-document-reading-engine.md](./feature-document-reading-engine.md) and [source-code-tree.md](./source-code-tree.md) under `src/app/reader/`

### I need to add or change settings
→ See [feature-settings-system.md](./feature-settings-system.md)

### I need to work with annotations or highlights
→ See [feature-annotation-system.md](./feature-annotation-system.md)

### I need to modify the library or book management
→ See [feature-library-management.md](./feature-library-management.md)

### I need to understand the state management
→ See [feature-state-management.md](./feature-state-management.md)

### I need to add a Tauri plugin or native feature
→ See [feature-cross-platform-support.md](./feature-cross-platform-support.md)

### I need to modify the sidebar
→ See [feature-sidebar-navigation.md](./feature-sidebar-navigation.md)

### I need to understand the folder structure
→ See [source-code-tree.md](./source-code-tree.md)

### I need to add a new feature
→ Review relevant feature docs, consult [source-code-tree.md](./source-code-tree.md) for file organization, follow existing patterns in [feature-state-management.md](./feature-state-management.md)

### I need to work with user authentication or cloud sync
→ See [feature-auth-sync.md](./feature-auth-sync.md)

## What's New (571baf98 → a23447a8)

### Major Features Added

1. **Authentication and Cloud Sync** (Dec 16-24, 2024)
   - User authentication via OAuth (Google, Apple, GitHub)
   - Cloud sync for reading progress across devices
   - Cloud sync for annotations and notes
   - Supabase backend integration
   - **See**: [feature-auth-sync.md](./feature-auth-sync.md)

2. **Web Platform Support** (Dec 5, 2024)
   - Browser-based PWA version of Readest
   - IndexedDB for client-side storage
   - Service worker for offline support
   - Platform detection and service abstraction
   - **See**: [feature-cross-platform-support.md](./feature-cross-platform-support.md)

3. **Popover Footnotes** (Dec 10, 2024)
   - Inline footnote popups without navigation
   - Vertical writing mode support (CJK languages)
   - Responsive positioning and sizing
   - Nested FoliateView rendering
   - **See**: [feature-annotation-system.md](./feature-annotation-system.md) (Popover Footnotes section)

4. **File Associations & Auto-Updater** (Dec 3, 2024)
   - "Open with Readest" system integration
   - Auto-update functionality for desktop apps
   - Command-line argument support
   - **See**: [feature-cross-platform-support.md](./feature-cross-platform-support.md)

### Enhancements

- Custom CSS support for non-ASCII characters
- Demo library with Feedbooks integration
- Improved OAuth handling on native platforms
- Various annotation and PDF improvements
- Enhanced dark mode support
- Better scrollbar handling across platforms

### Breaking Changes

None - all changes are backward compatible

---

## Version 0.9.0 Updates (a23447a8 → 76c5f585)

### Major Features Added (Dec 25, 2024 - Jan 5, 2025)

1. **Internationalization (i18n)** (Dec 26, 2024)
   - 14 language support with full translations
   - i18next framework with automatic language detection
   - Translation extraction and management tools
   - Persistent language preferences
   - **See**: [feature-internationalization.md](./feature-internationalization.md)

2. **Custom CSS Editor Improvements** (Jan 3, 2025)
   - Draft-based editing with explicit Apply button
   - Advanced CSS validation with detailed error messages
   - Better UX with clear save workflow
   - **See**: [feature-settings-system.md](./feature-settings-system.md) (Custom CSS Editor section)

3. **Vertical/Horizontal Layout Switch** (Jan 3, 2025)
   - Manual override for CJK book text direction
   - Three modes: Auto, Horizontal, Vertical
   - Per-book persistence
   - **See**: [feature-settings-system.md](./feature-settings-system.md) (Vertical/Horizontal Layout section)

4. **Deep Linking for OAuth** (Jan 3-4, 2025)
   - Native deep link support for OAuth callbacks
   - Improved authentication flow on desktop apps
   - Better Windows OAuth handling

5. **Window Position Persistence** (Jan 5, 2025)
   - Saves and restores window size and position
   - Cross-platform support

### Enhancements (0.9.0 Period)

- Release notes display in auto updater
- Book details modal with information display
- Context menu on book covers (Tauri apps)
- Bookmark and bootnote restoration fixes
- CLI interface improvements (`readest` binary name)
- Improved dark mode CSS support
- Multiple keyboard shortcuts added
- Enhanced translations and i18n coverage
- Windows portable binaries

---

## Version 0.9.2 - 0.9.7 Updates (76c5f585 → d757555f)

### Major Features Added (Jan 5-23, 2025)

1. **Text-to-Speech (TTS)** (Jan 7-15, 2025)
   - Dual backend: Web Speech API + Microsoft Edge TTS
   - 100+ neural voices across 50+ languages
   - Speech rate control (0.2x - 3.0x)
   - Audio preloading for seamless playback
   - Sentence navigation (forward/backward)
   - iOS audio unblocking and PWA support
   - Desktop media controls integration
   - **See**: [feature-text-to-speech.md](./feature-text-to-speech.md)

2. **Progressive Web App (PWA) Enhancements** (Jan 20-23, 2025)
   - Full PWA support with service worker
   - Installable on mobile devices
   - Dynamic theme color for browser UI
   - Safe area support for notches and home indicators
   - Navigation optimization (no page reloads)
   - Offline capabilities with caching
   - **See**: [feature-cross-platform-support.md](./feature-cross-platform-support.md) (PWA section)

3. **Mobile Platform Optimizations** (Jan 15-23, 2025)
   - Swipe up gesture to toggle header/footer
   - Responsive icon and font sizes
   - Touch-friendly UI elements
   - Responsive settings dialog and sidebar
   - Compact layouts for small screens

4. **Reading Progress in Bookshelf** (Jan 15, 2025)
   - Visual progress indicators on book covers
   - Percentage completion display
   - Quick identification of reading status

5. **Multi-Column Page Layout** (Jan 7, 2025)
   - Support for more than 2 columns
   - Per-book column configuration
   - Adaptive column width

### Enhancements (0.9.2 - 0.9.7 Period)

- Greek language translations added
- Noto Serif JP font support
- Toast notifications refactored to global component
- Improved mobile browser layout
- Delete functionality in book details modal
- PWA theme color in header and safe areas
- Responsive annotation tools for mobile
- Enhanced TTS UX with multiple improvements
- GB18030-2022 L3 charset fallbacks
- Dynamic viewport units for mobile browsers
- Disabled swipe gestures in scrolled mode

---

## Version 0.9.8 - 0.9.18 Updates (d757555f → cab757257)

### Major Features Added (Jan 23 - Feb 26, 2025)

1. **Library Groups** (v0.9.11, #368)
   - Organize books into custom groups/shelves
   - Nested group support
   - Drag-and-drop organization
   - Visual group indicators
   - **See**: [feature-library-management.md](./feature-library-management.md) (Library Groups section)

2. **User Profile Management** (v0.9.17, #452)
   - Dedicated user profile page at `/profile`
   - Self-service account deletion
   - Profile customization (display name, avatar)
   - Subscription status display
   - **See**: [feature-auth-sync.md](./feature-auth-sync.md) (User Profile section)

3. **iOS Native Features** (v0.9.11-0.9.18)
   - Native Sign in with Apple (#411)
   - Haptics feedback for interactions (#428)
   - iOS-optimized modals and annotation tools (#447)
   - Paging animations enabled by default (#410)
   - Safari-auth plugin for native OAuth (#433, #443)
   - **See**: [feature-cross-platform-support.md](./feature-cross-platform-support.md) (iOS Platform Optimizations)

4. **Keyboard Shortcuts** (v0.9.11-0.9.16)
   - Annotation shortcuts (H, N, D, C, T, W, S) (#378)
   - Half-page navigation (d/u keys) (#437)
   - Select mode toggle and quit app in library (#438)
   - Consistent keyboard UI with `<kbd>` tags (#421)
   - **See**: [feature-annotation-system.md](./feature-annotation-system.md) (Keyboard Shortcuts)

5. **Mobile UI Enhancements** (v0.9.11-0.9.17)
   - Transient auto-hiding toolbars (#394)
   - Pull-down gesture to dismiss modals (#440)
   - Grid view optimizations for mobile (#379)
   - Responsive font sizes for book notes (#415, #416)
   - Improved iPad detection and layouts (#416)
   - **See**: [feature-cross-platform-support.md](./feature-cross-platform-support.md) (Mobile UI Patterns)

6. **Arabic Language Support** (v0.9.15, #432)
   - Full RTL (right-to-left) layout support
   - Complete Arabic translations (115+ keys)
   - Dynamic UI direction switching
   - 15 languages now supported
   - **See**: [feature-internationalization.md](./feature-internationalization.md)

7. **Docker Self-Hosting** (v0.9.14, #430)
   - Official Dockerfile for self-hosted deployments
   - Docker Compose configuration
   - Environment variable management
   - Easier self-hosting for privacy-conscious users

### Enhancements (0.9.8 - 0.9.18 Period)

- **Settings**: Keep screen awake option during reading (#403)
- **Annotations**: Less saturated highlight colors for better readability (#453)
- **TTS**: Normalized language codes for better voice matching (#457)
- **UI**: Popup and dialog style improvements (#448)
- **Compatibility**: Fixed PDF TOC rendering (#402), footnote display (#400), font override issues (#422)
- **Navigation**: Preserve note ID when editing annotations (#436)
- **Alerts**: Improved positioning and z-index handling (#387, #390)

### Breaking Changes
None - all changes are backward compatible

### Documentation in Progress

The following feature sections are referenced above but have not yet been fully documented:

1. **iOS Platform Optimizations** (feature-cross-platform-support.md)
   - Native Sign in with Apple integration (#411)
   - Haptics feedback system (#428)
   - iOS-specific modals and annotation tools (#447)
   - Paging animations configuration (#410)
   - Safari-auth plugin for OAuth flow (#433, #443)

2. **Mobile UI Patterns** (feature-cross-platform-support.md)
   - Transient auto-hiding toolbar implementation (#394)
   - Pull-down modal dismissal gestures (#440)
   - Grid view mobile optimizations (#379)
   - Responsive font sizing for book notes (#415, #416)
   - iPad detection and responsive layouts (#416)

3. **User Profile Management** (feature-auth-sync.md)
   - Profile page architecture at `/profile` (#452)
   - Self-service account deletion flow
   - Profile customization features (display name, avatar)
   - Subscription status display

4. **Library Groups and Organization** (feature-library-management.md)
   - Library groups data structure (#368)
   - Nested group hierarchy
   - Drag-and-drop organization UI
   - Visual group indicators

These sections will be added in future documentation updates.

---

## Version 0.9.19 - 0.9.31 Updates (cab757257 → f4908c45)

### Major Features Added (Feb - Mar 2025)

1. **RTL and Arabic Language Support** (v0.9.19-0.9.20)
   - Full right-to-left (RTL) layout support
   - Arabic language translations and UI mirroring
   - RTL progress bars and navigation
   - Support for 8 RTL languages (Arabic, Hebrew, Persian, Urdu, etc.)
   - **See**: [internationalization/rtl-arabic.md](./internationalization/rtl-arabic.md)

2. **Vertical Layout Enhancements** (v0.9.19-0.9.31)
   - Vertical writing mode for CJK languages
   - Border frames for vertical reading (#612)
   - Punctuation replacement for vertical text (#754)
   - Enhanced vertical layout tools (#830)
   - **See**: Documentation integrated into settings and internationalization docs

3. **CJK Font Configuration** (v0.9.31, #836)
   - Configurable default CJK fonts in CJK environments
   - LXGW WenKai and Noto Serif JP support (#722)
   - Platform-specific CJK font lists
   - **See**: [settings-system/index.md](./settings-system/index.md)

4. **Book Metadata Search** (v0.9.31, #838)
   - Search books by title, author, and metadata in bookshelf
   - Enhanced library organization
   - **See**: [library-management/index.md](./library-management/index.md)

5. **TXT File Import** (v0.9.27, #655, #757, #708)
   - Support for importing TXT files as books
   - Desktop and mobile TXT file handling
   - **See**: [document-reading-engine/index.md](./document-reading-engine/index.md)

6. **Markdown Export for Annotations** (v0.9.25, #689)
   - Export annotations in markdown format
   - Structured export with metadata
   - **See**: [annotation-system/index.md](./annotation-system/index.md)

7. **Custom CSS Enhancements** (v0.9.19, #503, #507, #594)
   - Customize Foliate view styles with CSS (#507)
   - Styled reader UI via custom CSS (#503)
   - Theme editor for custom theme colors (#594)
   - **See**: [settings-system/custom-css-editor.md](./settings-system/custom-css-editor.md)

8. **Platform-Specific Enhancements**
   - **Android**: Content URI handling (#829, #833), Custom Tabs OAuth (#788), file chooser improvements (#798, #799, #807)
   - **iOS**: Background TTS (#822), Sign in with Apple (#411), haptics (#428), native OAuth (#433, #443)
   - **macOS**: Traffic light positioning (#297, #497)
   - **Windows**: Single-instance handler (#724, #726), style tweaks (#707)
   - **Linux**: F-Droid metadata (#682)
   - **See**: Platform-specific docs in [cross-platform-support/](./cross-platform-support/)

9. **Desktop Features**
   - Fullscreen option (#534)
   - Drag and drop import (#536)
   - Transient import mode (#709)
   - Window on top option (#825)

10. **Mobile UI Improvements**
    - Action bar at bottom (#681)
    - Keyboard handling improvements (#720, #762, #763)
    - Mobile-optimized layouts

11. **TTS Improvements**
    - Configurable timeout (#826)
    - Language normalization (#742)
    - Background audio on iOS (#822)
    - **See**: [text-to-speech/index.md](./text-to-speech/index.md)

12. **Reader UI and Interaction Enhancements** (v0.9.20-0.9.27)
    - Continuous scroll option (#522)
    - Settings preview snap dialog for mobile (#646)
    - Global fulltext search shortcut (Ctrl/Cmd+F, #750)
    - Click-to-flip area swap (#727)
    - Show/hide header/footer widgets (#620)
    - Scrolled mode toggler in layout panel (#596)
    - **See**: [settings-system/reader-ui-settings.md](./settings-system/reader-ui-settings.md)

13. **PWA and Web Platform Updates**
    - Cloudflare deployment with OpenNext (ede37757)
    - R2 storage integration (#718)
    - Screen wake lock focus/visibility handling (#502, #505)
    - **See**: [cross-platform-support/pwa-enhancements.md](./cross-platform-support/pwa-enhancements.md), [settings-system/screen-wake-lock.md](./settings-system/screen-wake-lock.md)

### Enhancements (0.9.19 - 0.9.31 Period)

- **Settings**: Reset password page (#731), language preference options (#686)
- **Layout**: Override justify style (#687), responsive footnotes (#632)
- **Performance**: Cache NativeFile with LRU (#816), DeepL free plan translation (#608)
- **UI**: Theme mode fixes (#824), custom CSS textarea (#766), rounded reader widget (#815)
- **Compatibility**: CBZ metadata parsing (#688), PDF TOC (#739), EPUB without dc metadata (#506)
- **Storage**: Fixed storage quota for self-hosted (#806), R2 and S3 support (#718)
- **Sync**: Improved translation API with retry (#770), load balancing (#771)

### Platform Documentation

Platform-specific documentation has been reorganized into separate files:
- [Android Platform](./cross-platform-support/android.md)
- [iOS Platform](./cross-platform-support/ios.md)
- [macOS Platform](./cross-platform-support/macos.md)
- [Windows Platform](./cross-platform-support/windows.md)
- [Linux Platform](./cross-platform-support/linux.md)
- [PWA Enhancements](./cross-platform-support/pwa-enhancements.md)

### Breaking Changes
None - all changes are backward compatible

---

## Version 0.9.32 - 0.9.43 Updates (f4908c45 → def157ca)

### Major Features Added (Mar - Nov 2025)

1. **Translation System** (v0.9.32-0.9.43)
   - Multi-provider support: DeepL, Azure, Google Translate, Yandex
   - Two-tier caching (memory + IndexedDB) for offline translations
   - KV cache for translation backend (#1184)
   - Quota management with automatic fallback
   - API v1/v2 compatibility for DeepL
   - Responsive translator popup (#1160)
   - Extension compatibility (LingKuma, Immersive Translate) (#901)
   - **See**: [translation-system/index.md](./translation-system/index.md)

2. **Bookshelf Views and Sorting** (v0.9.35-0.9.37)
   - List view for bookshelf (#955)
   - Sorting by title and author (#887)
   - Grid view with responsive columns
   - Book file size display (#1114)
   - Book description from metadata (#946)
   - **See**: [library-management/bookshelf.md](./library-management/bookshelf.md)

3. **Theme Customization** (v0.9.37)
   - Primary color input in theme editor (#970)
   - Custom theme synchronization to localStorage (#944)
   - Spell check disabled on color inputs (#1159)
   - **See**: [settings-system/theme-editor.md](./settings-system/theme-editor.md)

4. **Screen Orientation Control** (v0.9.39-0.9.40)
   - Lock screen orientation (auto/portrait/landscape) (#1034)
   - Auto orientation follows system settings (#1122)
   - Page-specific orientation (library unlocked) (#1084)
   - **See**: [settings-system/screen-orientation.md](./settings-system/screen-orientation.md)

5. **Prev/Next Section Navigation** (v0.9.43)
   - Section navigation buttons in footer bar (#1195)
   - Quick navigation between book sections
   - **See**: [settings-system/reader-ui-settings.md](./settings-system/reader-ui-settings.md)

6. **Scrolled Mode Pagination** (v0.9.43)
   - Scrolling overlap in pixels option (#1194)
   - Scroll offset adjustment for header/footer bars (#1193)
   - Unified pagination and scroll hooks (#1021)
   - More robust continuous scroll (#1017)
   - **See**: [document-reading-engine/index.md](./document-reading-engine/index.md)

7. **Volume Keys Navigation** (v0.9.38)
   - Volume keys for page turning (#982)
   - Volume key interception on iOS (#997)
   - Volume retention when backgrounded (#1014)
   - **See**: [annotation-system/keyboard-shortcuts.md](./annotation-system/keyboard-shortcuts.md), [cross-platform-support/ios.md](./cross-platform-support/ios.md)

8. **Immersive UI** (v0.9.36-0.9.41)
   - Immersive reader UI on iOS and Android (#911)
   - Hide navigation bar on Android 11+ (#927)
   - Auto-hide navigation for Android <11 (#960)
   - System navigation bar swipe gesture (#912)
   - Transient navigation bar on Android 9 (#1085)
   - **See**: [cross-platform-support/android.md](./cross-platform-support/android.md), [cross-platform-support/ios.md](./cross-platform-support/ios.md)

9. **File Manager Integration** (v0.9.35)
   - Open files from file manager on Android (#895)
   - Open files from file manager on iOS (#898)
   - Filename with quotes handling (#897)
   - Content provider URI handling (#861)
   - **See**: [cross-platform-support/android.md](./cross-platform-support/android.md), [cross-platform-support/ios.md](./cross-platform-support/ios.md)

10. **In-App Updater** (v0.9.33-0.9.41)
    - Android in-app updater (#885)
    - New updater dialog (#874)
    - Update status in about window (#1109)
    - Disabled for non-AppImage on Linux (#1141)
    - Removed deprecated MSI installer (#1187)
    - **See**: [cross-platform-support/android.md](./cross-platform-support/android.md), [cross-platform-support/linux.md](./cross-platform-support/linux.md)

11. **System Fonts Support** (v0.9.38)
    - Retrieve system fonts on iOS and Android (#976)
    - Font weight variants display (#976, #1158)
    - Filter non-free fonts (#999)
    - Import system fonts list on Android (#998)
    - Font preview fixes for Linux (#1023) and Windows (#1054)
    - **See**: [settings-system/index.md](./settings-system/index.md)

12. **Cloud Backup Status** (v0.9.42)
    - Show cloud backup status for each book (#1173)
    - Mobile and desktop indicators
    - Books without covers can sync (#878)
    - **See**: [auth-sync/index.md](./auth-sync/index.md)

### Enhancements (0.9.32 - 0.9.43 Period)

- **Settings**:
  - Fullscreen keyboard shortcut (F11) (#942)
  - Window maximized/fullscreen conflict resolution (#872)
  - Open last book on start option (#1052)
  - Keep screen awake disabled by default (#1149)
  - Separate header/footer visibility for paginated/scrolled modes (#859)
  - Compact margin when header/footer dismissed (#1047)
  - Horizontal margin default fix (#1155)

- **Library Management**:
  - Download/upload buttons in book details (#891)
  - Library state maintenance across navigation (#1096)
  - Delete book updates store properly (#1191)

- **TTS**:
  - TTS control view hierarchy reorganization (#1119)
  - Don't scroll when selection in current page (#1046)
  - Timeout options for iOS (#1037)
  - Language detection improvements (#1003)
  - Metadata language fallback (#945)
  - More English voices for all locales (#969)

- **Annotations**:
  - Text selector hook refactor (#1131)
  - Selection anchor preservation across pages (#968)
  - Underline/squiggly highlight positioning (#845)
  - Popup footnotes for anchors without EPUB namespace (#956)
  - Inherit book fonts for popup footnotes (#985)
  - Footnote visibility fixes (#1099)

- **Sidebar Navigation**:
  - TOC location information display (#1016)
  - TOC expansion glitches fix (#1081)
  - Clickable region expansion for TOC icons (#1044)

- **Document Reading Engine**:
  - CSS background color override refinements (#1164, #1163)
  - Gutenberg eBooks compatibility (#1154)
  - Font weight variants for built-in fonts (#1158)
  - Text align and indent override (#1104, #1105)
  - TXT parser improvements (#1139, #1077, #1043, #1024, #1005)
  - Viewport size calculation fixes (#1045)
  - Image dimension layout fixes (#1090)
  - Apply layout styles to div tags (#1142)
  - Background replacement on attribute change (#1126)
  - Style overriding fixes (#1091, #1117, #1142)

- **Platform-Specific**:
  - Android: Back key intercept (#983), drag handler padding (#1039, #1110), status bar height query (#947), dismiss status bar on resume (#952), action bar closure (#1086)
  - iOS: Volume keys improvements (#997, #1014)
  - macOS: OAuth via ASWebAuthenticationSession (#866), native Sign in with Apple (#856), avatar caching (#1140)
  - Linux: xdg-mime check for deeplink (#951), ARM HF architecture (#1133)
  - Windows: Fullscreen/maximized handling (#872)

- **UI/UX**:
  - Mobile UI fixes and enhancements (#1113)
  - Less sensitive trackpad/mouse page flipping (#1172)
  - Link styling improvements (#1083)
  - Dialog drag handler layout (#1076, #1110, #1039)
  - Action bar dismiss with footer (#1086)
  - Screen orientation for library page (#1084)

- **Internationalization**:
  - Nederlands translations (#1161)
  - RTL layout for bottom configuration panel (#850)

### Breaking Changes

None - all changes are backward compatible

---

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### Major Features Added (March - June 2025)

1. **Subscription Management** (v0.9.62)
   - Premium subscription tiers (Free, Premium, Pro)
   - Stripe payment integration for web/desktop
   - Native payment integration for iOS/Android
   - Usage quota management (storage, translation)
   - Subscription status display in settings
   - **See**: [auth-sync/index.md](./auth-sync/index.md) (Subscription Management section)

2. **Bilingual Translation** (v0.9.49)
   - Full book translation with side-by-side display
   - TOC translation (#1273)
   - Hide/show original text option (#1511)
   - Translation header auto-hide on mobile (#1274)
   - **See**: [translation-system/index.md](./translation-system/index.md)

3. **Bilingual TTS** (v0.9.49-0.9.50, #1230, #1263)
   - Two voices for bilingual books
   - Automatic language detection per sentence
   - Script-based language inference (#1233)
   - Voice selection UI for each language
   - **See**: [text-to-speech/index.md](./text-to-speech/index.md)

4. **Native Android TTS** (v0.9.56-0.9.57, #1376, #1387, #1394)
   - System TTS engine integration
   - Offline TTS support
   - Better battery efficiency
   - Google TTS, Samsung TTS compatibility
   - **See**: [text-to-speech/index.md](./text-to-speech/index.md)

5. **Markdown Notes** (v0.9.52, #1315)
   - Full markdown syntax support
   - Editor/preview split mode
   - Syntax highlighting for code blocks
   - Export notes as markdown files
   - **See**: [annotation-system/index.md](./annotation-system/index.md)

6. **Notebook Search** (v0.9.52, #1318)
   - Full-text search across all notes and highlights
   - Filter by book, color, type
   - Sort by date or relevance
   - Jump to location from search results
   - **See**: [annotation-system/index.md](./annotation-system/index.md)

7. **iPad Split-Screen Mode** (v0.9.58, #1416)
   - Native iPad split-screen support
   - Resizable sidebars (#1415)
   - Optimized layouts for iPad
   - **See**: [cross-platform-support/ios.md](./cross-platform-support/ios.md)

8. **Individual Margin Adjustment** (v0.9.58, #1410, #1413)
   - Separate controls for top, bottom, left, right margins
   - Per-book margin settings
   - Multiple columns in portrait mode (#1413)
   - **See**: [settings-system/reader-ui-settings.md](./settings-system/reader-ui-settings.md)

9. **MOBI Performance Optimization** (v0.9.63, #1547)
   - Speed up opening for large MOBI books
   - Improved TOC handling (#1542)
   - Link navigation fixes (#1528)
   - Empty fragments handling (#2456)

### Enhancements (0.9.44 - 0.9.63 Period)

- **Translation**:
  - Daily DeepL quota management (#1349, #1363)
  - Punctuation post-processing (#1245)
  - Lazy loading optimization (#1282)
  - Responsive popup on mobile (#1160)

- **TTS**:
  - Media session with speaking sentence (#1289)
  - Read from last sentence (#1291, #1293)
  - Skip empty speech at chapter end (#1243)
  - Independent TTS per book view (#1411)
  - Keyboard shortcut `T` to toggle (#1405)
  - Annotation tools work with TTS (#1406)
  - Translation with background TTS (#1399)

- **Settings**:
  - Invert image color in dark mode (#1223)
  - Opt-out telemetry option (#1236)
  - Enable JavaScript in EPUB (#1295)
  - TOC sort by page number (#1308)
  - Remaining pages in chapter (#1478)
  - Remaining minutes in chapter (#1326)
  - Always show status bar (#1417)
  - Override book fg/bg color (#1335)
  - Parallel reading toggle (#1504)
  - Reset settings option (#1475)

- **Library Management**:
  - Select all button in select mode (#1209)
  - Show current books count (#1312)
  - Update bookshelf after import/delete (#1314, #1331)
  - Delete cloud backup only (#1546)

- **Annotations**:
  - Show annotation create time (#1412)
  - Notebook layout tweaks (#1319)
  - Restore view settings when reopening (#1400)
  - Annotation tools work when TTS enabled (#1406)

- **Authentication & Sync**:
  - Sync status indicator in view menu (#1324)
  - Deleted notes synchronization (#1357)

- **Cross-Platform (iOS/iPad)**:
  - Split-screen mode support (#1416)
  - Resizable sidebars on iPad (#1415)
  - Import reliability improvements (#1439)
  - Smoother orientation changes (#1441)
  - Safe area insets (#1408)
  - Splash screen and icon improvements (#1450)

- **Cross-Platform (Android)**:
  - Native TTS engine (#1387)
  - Overlay scrollbar for TOC (#1506)
  - Compatibility fixes (#1394)

- **Custom CSS & UI**:
  - Custom CSS for reader UI (#1466)
  - Cover crop/fit option (#1469)
  - Fitted cover images (#1476, #1483)
  - Non-ASCII character support

- **Format Support**:
  - MOBI link handling improvements (#1528, #1542)
  - Large MOBI optimization (#1547)
  - Empty fragments handling (#2456)
  - Table scaling to fit constraints (#2455)

- **Internationalization**:
  - Thai (th-TH) translations added (#1548)
  - Additional CJK fonts (#1484)

- **Other**:
  - Book details modal with HTML description (#1317)
  - CJK font loading optimization (#1323)
  - PDF.js bump to v4 (#1325)
  - Supabase.js and Next.js updates (#1361, #1362)
  - Select filtered books when activating select all (#1237)
  - Trigger library update after import (#1336)
  - Exit select mode when all deleted (#1350)
  - Deleted notes synchronization (#1357)
  - Syntax highlighting for code (#1386)
  - Client error handling (#1389)

### Breaking Changes

None - all changes are backward compatible

---

## Version 0.9.64 - 0.9.67 Updates (f5b686ab → 33b2ba16)

### Major Features Added (November 2025)

1. **Update Notes for Releases** (v0.9.64, #1552)
   - Display release notes when new updates are available
   - Auto-updater shows changelog before updating
   - Improved update experience for users
   - **See**: [cross-platform-support/index.md](./cross-platform-support/index.md)

2. **Book Metadata Editor** (v0.9.64, #1583)
   - Edit book title, author, and other metadata
   - Custom cover image upload (#1588)
   - Save custom covers in apps
   - Metadata sync across devices (#1611)
   - **See**: [library-management/index.md](./library-management/index.md)

3. **Multiple Reader Windows** (v0.9.65, #1596)
   - Support for multiple reader windows on desktop
   - Open different books in separate windows
   - Independent window state for each book
   - **See**: [cross-platform-support/index.md](./cross-platform-support/index.md)

4. **Search in Library** (v0.9.67, #1662)
   - Search by book format (EPUB, PDF, MOBI, etc.)
   - Search in group names and descriptions
   - Enhanced library organization
   - **See**: [library-management/index.md](./library-management/index.md)

5. **Yandex Translator Integration** (v0.9.67, #1652)
   - Added Yandex Translator as translation provider
   - Free translation service option
   - Multiple Yandex service options (yandexgpt, yandextranslate, etc.)
   - **See**: [translation-system/index.md](./translation-system/index.md)

6. **iOS In-App Purchase (IAP)** (v0.9.67, #1673, #1676, #1678)
   - Native IAP support for iOS
   - Upgrade to Readest Premium via iOS
   - Server-side receipt validation
   - Sandbox environment for TestFlight
   - **See**: [auth-sync/index.md](./auth-sync/index.md), [cross-platform-support/ios.md](./cross-platform-support/ios.md)

7. **PDF Custom Background Theming** (v0.9.67, #1661)
   - Apply custom background colors to PDF files
   - Dark mode support for PDFs
   - Consistent theming across all formats
   - **See**: [document-reading-engine/index.md](./document-reading-engine/index.md)

8. **Premium Cloud Storage** (v0.9.67, #1696)
   - Increased cloud sync storage for premium users
   - Enhanced backup capabilities
   - Priority sync for premium accounts
   - **See**: [auth-sync/index.md](./auth-sync/index.md)

### Enhancements (0.9.64 - 0.9.67 Period)

- **Window Borders**: Added window borders on Windows 10 (#1556) and Linux (#1570, #1599)
- **Layout & CSS**:
  - Fixed dimension of inline images (#1555, #1560)
  - Lightened highlight color in dark mode (#1489, #1561)
  - Removed unintended indent for images (#1567, #1568)
  - Named container classes for easier CSS customization (#1598, #1600)
  - Maintain layout of anchor elements while increasing tap target size (#1603, #1605)
  - Unset text indent inside list elements (#1609)
  - Fixed insets for double borders and added book spine decorator (#1688)
  - Fixed hard-coded font color (#1695, #1697)
  - Multiply img color in light mode when overriding book color (#1656)

- **Full-text Search**: More responsive full-text search (#1558, #1562)

- **Pull-down Refresh**: Smoother and more responsive (#1564)

- **Bookmark Ribbon**: Correctly placed when sidebar is pinned (#1565)

- **Translation**:
  - Fixed translation not working for table of contents (#1610)
  - Skip translating pre, code and math tags (#1693, #1698)

- **Text-to-Speech**:
  - Handle invalid language codes and show no voices hints (#1579, #1607)
  - Skip TTS for rubys and footnote anchors (#1334, #1608)
  - Convert ISO 639-2 language codes to ISO 639-1 for TTS voice filtering (#1627, #1639)

- **Table of Contents**: Fixed nested TOC items not expanded in very long TOC lists (#1625, #1629)

- **Book Cover**: Replace fallback book cover with new cover image (#1604), update book cover from metadata in sidebar (#1615)

- **Metadata**: Also sync book metadata (#1611), various fixes on metadata editor and bookshelf (#1663)

- **Performance**:
  - Eliminate redundant re-renders of book cover components (#1685)
  - Multi-part download with range access (#1690)

- **Platform-Specific**:
  - **iOS**: Skip context menu when long-press on book cover (#1612, #1613)
  - **macOS**: Hover header to show traffic light window control (#1645, #1653)
  - **Linux**: Fixed multiple instances created in OAuth (#1654, #1659)
  - **Android**: Target Android SDK to version 36

- **Layout**:
  - Fixed hardcoded image layout in fixed layout documents (#1660)
  - Fixed layout for auth and user page (#1637)

- **File Handling**:
  - Chaining file open with OS opening having highest precedence (#1622, #1636)
  - Don't use comma as separator when parsing filenames (#1622, #1650)
  - Set xdg-mime with mime type other than scheme (#1621, #1641)

- **Library**:
  - Fix book not redownloaded for the redownload button in detail modal (#1628, #1638)

- **Error Handling**:
  - Handle ChunkLoadError by refreshing page (#1619)
  - Fix loading chunk error of optional chaining for Android WebView below 80 (#1626)

- **Build & Infrastructure**:
  - Bump Tauri, Next.js, and Zustand to latest versions (#1631)
  - Fix CORS for API with new Next.js version (#1633)
  - Downgrade Next.js to 15.3 for compatibility (#1634)
  - Fixed failed AppImage builds for Linux
  - Suppress warnings from old objc crate (#1686)

- **Fonts**: Fix broken links for online CJK fonts (#1687)

- **Library Data**: Load backup library data if main library data is unavailable (#1672, #1689)

- **Configuration**: Default to open file with new window (#1691)

- **API**:
  - Batch updating daily usage key in KV (#1694)
  - Ensure proper string decoded on edge runtimes (#1680)
  - Use node API endpoint for IAP verifying (#1683)

### Breaking Changes

None - all changes are backward compatible

---

**Version**: Documentation for commits `571baf98` through `33b2ba16` (v0.9.64 → v0.9.67)
**Last Updated**: November 2025

**For any questions or issues with this documentation, please consult the commit history from `571baf98` to `33b2ba16` for context on the codebase state at this point.**
