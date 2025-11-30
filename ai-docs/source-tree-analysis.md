# Source Tree Analysis – khana

This document summarizes the key directories and their responsibilities for the khana project.

**khana** is a customized fork of [Readest](https://github.com/readest/readest), an open-source cross-platform ebook reader. The current development focus is on the **web application** with Supabase backend, following the Open/Closed Principle to minimize modifications to upstream Readest code.

## 1. Repository Root

- `package.json` – Monorepo root config with a `tauri` script that targets the desktop application.  
- `pnpm-workspace.yaml` – Defines the workspace structure.  
- `apps/` – Contains the primary application(s); here, the main one is `readest-app`.  
- `packages/` – Reserved for shared libraries and native modules (currently mostly placeholders).  
- `.bmad/` – BMad method configuration, agents, workflows, and documentation templates.

## 2. Web/Desktop App – `apps/readest-app`

**Current Focus: Web Application**

- `package.json` – Next.js web app definition with optional Tauri support, scripts for dev, build, and platform-specific builds
- `next.config.mjs` – Next.js configuration for the frontend
- `tailwind.config.ts`, `postcss.config.mjs` – Styling pipeline configuration
- `.env.local` – Supabase configuration (URL, anon key, admin key, storage quota, API base URL)

**Optional: Desktop/Mobile (Not Current Focus)**

- `src-tauri/` – Tauri 2 backend for desktop/mobile builds:
  - `src-tauri/src/main.rs` – Rust entrypoint delegating to `readestlib::run()`
  - `src-tauri/tauri.conf.json` – Core Tauri configuration incl. CSP, asset protocol, and bundle targets
  - `src-tauri/icons/` – App icons for different platforms

**Backend: Supabase**

- Database schema: `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md`
- Tables: `books`, `book_configs`, `book_notes`, `files`
- Authentication: Supabase Auth
- Storage: Supabase Storage for book files and covers
- Row-level security policies for user data isolation

## 3. Frontend Source – `apps/readest-app/src`

- `app/` – Next.js App Router structure:
  - `app/layout.tsx` – Application layout wrapper, fonts, providers.  
  - `app/page.tsx` – Redirects to `/library`.  
  - `app/library/` – Library listing, selection, and book import flows.  
  - `app/reader/` – Reader UI, including `components`, `hooks`, and `utils` specific to the reading experience.

- `components/` – Shared UI components:
  - Buttons, dropdowns, window controls, alert/toast components, spinners, and modal-like elements.

- `store/` – Zustand stores:
  - `readerStore.ts` – Manages reading views, progress, and per-view settings.  
  - `libraryStore.ts` – Tracks the library of books and related metadata.  
  - `settingsStore.ts` – System-wide reading and view settings.  
  - `bookDataStore.ts`, `notebookStore.ts`, `sidebarStore.ts` – Additional domain-specific state.

- `services/` – Domain services and environment abstractions:
  - `appService.ts` – Abstract `BaseAppService` handling settings, library persistence, and book import
  - `environment.ts` – Runtime environment detection and configuration
  - `nativeAppService.ts` – Tauri API bindings (for desktop builds)
  - `supabaseService.ts` – Supabase client integration for web app (auth, storage, database sync)
  - `constants.ts` – Shared configuration defaults

- `utils/` – Utility modules:
  - `book` utilities (paths, filenames, configuration helpers).  
  - `file`, `md5`, and TOC helpers.  
- `types/` – Type definitions for books, system, and settings.  
- `styles/` – Theme configuration and styling utilities.  
- `context/` – React context providers for environment and auth.  
- `libs/` and `helpers/` – Library integration and shared helper functions.

## 4. Packages

- `packages/foliate-js` – Placeholder for the foliate-js integration package (likely providing book rendering capabilities).  
- `packages/tauri` – Placeholder for shared Tauri-related code or configuration.

## 5. Takeaways for Future Work

**Web App Development (Current Focus):**

- **UI changes:** `apps/readest-app/src/app` and `apps/readest-app/src/components`
- **State management:** `apps/readest-app/src/store` with Supabase sync integration
- **Backend integration:** `apps/readest-app/src/services` for Supabase operations (auth, storage, database)
- **Database schema:** Reference `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md` for data models
- **Open/Closed Principle:** Create new files and components rather than modifying upstream Readest code to avoid merge conflicts

**Desktop/Mobile Development (Optional, Not Current Focus):**

- Native capabilities or OS integrations go in `apps/readest-app/src-tauri` and wire through to frontend via services
- Desktop builds use local filesystem storage instead of Supabase

**Upstream Relationship:**

- Fork of: https://github.com/readest/readest
- Strategy: Keep customizations separate for clean upstream merges
- Modification policy: Only modify Readest files when absolutely necessary

