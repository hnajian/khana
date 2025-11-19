# Readest Codebase Documentation Index

**Version**: Documentation for commit `571baf98` (Add About Readest window)
**Last Updated**: November 2024

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
| **Annotation System** | [feature-annotation-system.md](./feature-annotation-system.md) | Highlighting, notes, translation, dictionary integration |
| **Sidebar Navigation** | [feature-sidebar-navigation.md](./feature-sidebar-navigation.md) | TOC, search, bookmarks, booknotes views |
| **Settings System** | [feature-settings-system.md](./feature-settings-system.md) | Three-tier settings hierarchy (global/book/view) |
| **State Management** | [feature-state-management.md](./feature-state-management.md) | Zustand stores architecture and patterns |
| **Library Management** | [feature-library-management.md](./feature-library-management.md) | Book import/export, metadata, cover handling |
| **Cross-Platform Support** | [feature-cross-platform-support.md](./feature-cross-platform-support.md) | Tauri native shell, platform-specific code |

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

---

**For any questions or issues with this documentation, please consult the commit history around `571baf98` for context on the codebase state at this point.**
