# khana Documentation Index

**Type:** Cross-platform web/desktop/mobile app (Web focus with Tauri support)
**Primary Language:** TypeScript/Rust
**Architecture:** Next.js 16 web app + Supabase backend (with optional Tauri for desktop/mobile)
**Last Updated:** 2025-11-29

## Project Overview

**khana** is a customized fork of [Readest](https://github.com/readest/readest), an open-source ebook reader designed for immersive and deep reading experiences. Readest is a modern rewrite of "Foliate", leveraging Next.js 16 and Tauri v2 to deliver a smooth, cross-platform experience across macOS, Windows, Linux, Android, iOS, and the Web.

**Current Development Focus:** Web application with Supabase backend. While the codebase supports desktop (Tauri) and mobile platforms, the current focus is exclusively on developing a customized web app experience.

**Development Philosophy:** Following the Open/Closed Principle from SOLID to minimize modifications to upstream Readest files. New features and customizations are implemented in separate files to avoid merge conflicts when pulling updates from the upstream repository.

The project is structured as a small monorepo, with the primary product located at `apps/readest-app`. Additional package folders are reserved for shared libraries and native integrations.

## Quick Reference

- **Tech Stack:** Next.js 16 (React 18, TypeScript), Supabase (backend/auth/storage), Zustand, Tailwind CSS
- **Tauri Support:** Tauri 2 (Rust) available for desktop/mobile builds (not current focus)
- **Entry Point:** `apps/readest-app/src/app/layout.tsx` (web UI)
- **Architecture Pattern:** SPA-style React application with feature-oriented folders for `library` and `reader` flows
- **Database:** Supabase PostgreSQL for web app (schema: `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md`)
- **Desktop/Mobile:** Local JSON files via Tauri filesystem APIs (when using desktop builds)
- **Deployment:**
  - **Web:** Next.js deployment with Supabase backend
  - **Desktop/Mobile (optional):** Tauri bundles for macOS, Windows, Linux, Android, iOS

## Generated Documentation

### Core Documentation

- [Project Overview](./project-overview.md) - Executive summary and high-level architecture  
- [Source Tree Analysis](./source-tree-analysis.md) - Annotated directory structure  

### Optional Documentation

Currently not generated. For deeper planning, you can add:

- `architecture.md` – Detailed technical architecture  
- `component-inventory.md` – Catalog of major components and UI elements  
- `development-guide.md` – Local setup and development workflow details

## Existing Documentation

The main existing docs are:

- `apps/readest-app/README.md` – Generic Next.js starter README  

## Getting Started

### Prerequisites

- Node.js and `pnpm`  
- Rust toolchain compatible with Tauri 2  
- Platform-specific Tauri prerequisites (Xcode/CLT on macOS, Visual Studio Build Tools on Windows, appropriate system libraries on Linux)

### Setup

```bash
pnpm install
```

### Run Locally

**Web App (Primary Focus):**

From the repository root:

```bash
pnpm dev-web
```

This starts the Next.js dev server for the web application. Requires Supabase configuration in `.env.local` (see `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md` for schema setup).

**Desktop App (Optional):**

```bash
pnpm tauri
```

This runs the Tauri dev workflow, which in turn starts the Next.js dev server with desktop shell capabilities.

### Run Tests

```bash
pnpm test
```

(At the moment, this is a placeholder and no test suite is defined.)

## For AI-Assisted Development

**IMPORTANT: Open/Closed Principle**
When implementing new features, prioritize creating new files and components rather than modifying existing Readest code. This minimizes merge conflicts with upstream updates.

When planning new features:

- **Reader UI-only features:** Focus on `apps/readest-app/src/app/reader`, `apps/readest-app/src/components`, and the Zustand stores in `apps/readest-app/src/store`.
- **Web app backend/sync features:** Reference Supabase schema in `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md` and implement integration in `apps/readest-app/src/services`.
- **Library and persistence features:** Focus on `apps/readest-app/src/services`, `apps/readest-app/src/utils`, and `apps/readest-app/src/store` (especially `libraryStore`, `bookDataStore`, and `settingsStore`).
- **Desktop shell or native capabilities (not current focus):** These exist in `apps/readest-app/src-tauri` (Rust) for future desktop/mobile builds.

**Upstream Relationship:**
- **Origin:** Fork of https://github.com/readest/readest
- **Strategy:** Keep customizations separate to enable clean upstream merges
- **Modification Policy:** Only modify Readest files when absolutely necessary

Use this index as the entry point and extend it with more detailed architecture and component documentation as the project evolves.

