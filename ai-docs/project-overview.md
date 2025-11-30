# khana – Project Overview

## 1. Product Summary

**khana** is a customized fork of [Readest](https://github.com/readest/readest), an open-source ebook reader designed for immersive and deep reading experiences. Readest is built as a modern rewrite of "Foliate", leveraging Next.js 16 and Tauri v2 to deliver a cross-platform experience across web, desktop (macOS, Windows, Linux), and mobile (Android, iOS).

**Development Focus:** Web application with Supabase backend for authentication, storage, and sync.

**Development Strategy:** Following the Open/Closed Principle from SOLID—new features and customizations are implemented in separate files to minimize modifications to upstream Readest code, avoiding merge conflicts when pulling updates from the upstream repository.

At a high level, the product:

- Manages a cloud-synced library of books with cross-device synchronization (web focus)
- Imports book files, computes a hash, and stores them in Supabase storage
- Renders book content using `foliate-js` and related document utilities
- Tracks per-book reading progress, bookmarks, notes, and view settings across devices
- Uses Supabase PostgreSQL for data persistence and sync (schema: `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md`)
- Supports desktop/mobile via Tauri (not current focus), which uses local filesystem-based storage

## 2. High-Level Architecture

**Current Focus: Web Application Architecture**

- **Backend (Supabase):**
  - PostgreSQL database with tables for books, book_configs, book_notes, and files
  - Row-level security (RLS) policies for user data isolation
  - Authentication via Supabase Auth
  - File storage for book files and covers
  - Database schema documented in `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md`

- **Frontend Application:**
  - Next.js 16 app under `apps/readest-app/src/app`
  - Uses React 18 with the App Router, client components for interactive views, and a redirect from `/` to `/library`
  - UI components are organized in `apps/readest-app/src/components` and feature-specific folders under `src/app/library` and `src/app/reader`
  - Supabase client integration for auth, data sync, and storage

- **State Management:**
  - Zustand stores in `apps/readest-app/src/store` manage library, reader, notebook, settings, sidebar state, and more
  - `readerStore` coordinates view instances, reading progress, and per-book view settings
  - `bookDataStore` caches loaded `BookDoc` structures and configurations per book
  - Stores sync with Supabase for cross-device persistence

- **Domain and Services Layer:**
  - `apps/readest-app/src/services` contains abstractions like `BaseAppService` for backend interactions, book import, configuration loading, and persistence
  - Types for books, system, and settings live under `apps/readest-app/src/types`
  - Utility logic (TOC handling, hashing, file path helpers, styling utilities) resides in `apps/readest-app/src/utils`

**Optional: Desktop/Mobile Architecture (Not Current Focus)**

- **Desktop Shell:**
  - Implemented with Tauri 2, configured in `apps/readest-app/src-tauri/tauri.conf.json`
  - Rust entrypoint at `apps/readest-app/src-tauri/src/main.rs` delegates to `readestlib::run()`
  - Provides filesystem access and OS-level integration via `@tauri-apps/api` and plugins
  - Uses local JSON files for offline-first storage

## 3. Key Flows

**Web App Flows (Current Focus):**

- **Startup Flow:**
  - Next.js web app loads in browser
  - Supabase client initializes with auth state
  - App redirects from `/` to `/library`
  - Global context providers and styles are initialized via `layout.tsx`
  - Library data syncs from Supabase (books, configs, notes)

- **Library Management Flow:**
  - User imports a book via file upload
  - App processes the file:
    - Uses `DocumentLoader` to parse metadata and content into a `BookDoc`
    - Computes a partial MD5 hash to uniquely identify the book
    - Uploads book file to Supabase Storage
    - Stores book metadata in Supabase `books` table
    - Syncs library state across all user devices

- **Reading Flow:**
  - User selects a book from library
  - Reader view initializes a `FoliateView` for the selected book
  - `readerStore.initViewState` loads `BookDoc` and book configuration from Supabase
  - Reader updates progress, location, and TOC context via `setProgress`
  - Progress and view settings sync to Supabase `book_configs` table in real-time
  - Annotations (highlights, notes) sync to `book_notes` table

**Desktop/Mobile Flows (Optional, Not Current Focus):**

- **Startup Flow:**
  - Tauri starts the Rust backend and serves the Next.js frontend from `devUrl` or `frontendDist`
  - Local filesystem-based configuration loading

- **Library Management Flow:**
  - Loads files via Tauri filesystem APIs and `RemoteFile`
  - Stores in local `Books` directory with JSON configs

- **Reading Flow:**
  - Persists to local book config files via Tauri filesystem APIs

## 4. Non-Functional Characteristics

- **Performance:**
  - Web app optimized for modern browsers with fast rendering via Next.js
  - Supabase provides low-latency data access and real-time sync
  - Reading logic leans on `foliate-js` for efficient document rendering
  - Desktop builds (when needed) use Tauri for native performance

- **Portability:**
  - Web app runs on any modern browser (Chrome, Firefox, Safari, Edge)
  - Cross-device sync via Supabase ensures consistent experience
  - Optional desktop/mobile builds available via Tauri for macOS, Windows, Linux, Android, iOS
  - Frontend is standard Next.js/React, easing platform development

- **Extensibility:**
  - **Open/Closed Principle:** New features implemented in separate files to avoid upstream conflicts
  - Domain logic factored into services, utils, and Zustand stores
  - Supabase backend enables cloud-based features (sync, collaboration, AI integration)
  - Modular architecture supports adding features like advanced search, AI study tools, or social features

## 5. Suggested Next Documentation Steps

If you want deeper documentation for planning and AI-assisted work:

- Generate an `architecture.md` focusing on:
  - Supabase integration patterns and data flow
  - Module boundaries and service abstractions
  - Custom vs. upstream code organization (Open/Closed Principle implementation)
- Generate a `component-inventory.md` describing key UI components and pages (library, reader, settings)
- Create a `development-guide.md` documenting:
  - Supabase setup and configuration
  - Environment setup and debugging
  - Upstream merge strategies and conflict resolution
  - Build/release flows for web and optional desktop/mobile builds

