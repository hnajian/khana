# khana - Product Requirements Document

**Author:** {{user_name}}
**Date:** {{date}}
**Version:** 1.0

---

## Executive Summary

An immersive, education-focused e-book reader that helps students move from shallow, distracted screen reading to deep, focused study sessions. The product centers on long-form textbook reading, with a clean, comfortable interface designed for hours of use without cognitive overload.

Students bring their own course materials (primarily textbook PDFs/EPUBs) and use the reader as their primary study surface: reading, marking up, and revisiting key concepts. Core experiences include high-quality text rendering, stable layout for textbooks, thoughtful typography, and navigation patterns tuned for dense, structured content (chapters, sections, figures, tables).

The product aligns with the broader goal of improving learning outcomes and study satisfaction for students who already have digital textbooks but dislike current reading experiences. By combining ergonomic reading, study tools, and light research helpers in one place, it turns “I have to read this textbook” into a more engaging, less painful workflow.

### What Makes This Special

Unlike generic e-book readers, this product is opinionated around textbook-style studying rather than casual reading. It treats highlights, notes, and inline lookup as first-class interactions, making it easy to move between reading, annotating, and revisiting key concepts without losing focus.

The combination of comfortable long-form reading, structured study tools, and quick research links inside a single workspace is what differentiates it from commodity PDF/EPUB readers.

---

## Project Classification

**Technical Type:** web_app
**Domain:** edtech
**Complexity:** medium

**khana** is a customized fork of [Readest](https://github.com/readest/readest), an open-source cross-platform ebook reader. While Readest supports web, desktop (via Tauri v2), and mobile platforms, **khana's current development focus is exclusively on the web application** with Supabase backend for authentication, storage, and cross-device sync.

This web application is designed for consuming and studying existing textbook files. The core of the system is a rich reading and annotation experience, with cloud-based handling of imported files and student-generated notes, plus real-time sync across devices via Supabase. It supports EPUB and PDF document formats.

**Development Strategy:** Following the Open/Closed Principle from SOLID to minimize modifications to upstream Readest code, enabling clean merges when pulling updates from the upstream repository.

The domain is education technology focused on individual students rather than institutional LMS integrations—for now. Complexity is driven more by UX quality, annotation workflows, cloud sync behavior, and document handling than by deep backend or compliance needs, which keeps it in the medium range at this stage.

{{#if domain_context_summary}}

### Domain Context

{{domain_context_summary}}
{{/if}}

---

## Success Criteria

Success means students choose this as their primary way to study digital textbooks because it feels noticeably more comfortable and productive than using generic PDF/e-book readers.

- Students can read for long, focused sessions without feeling “lost in the document” or fighting the UI.  
- Highlighting and note‑taking become a natural part of reading, not a clumsy extra step.  
- Students can reliably return to important passages and notes when revising for exams or assignments.  
- Quick lookup (dictionary and web links) supports understanding without pulling them out of the reading flow.  

From a product perspective, success looks like:

- High repeat usage during study periods (students returning to the same book multiple times).  
- A meaningful portion of imported textbooks being actively annotated (not just opened once).  
- Positive qualitative feedback about comfort, focus, and study effectiveness.

{{#if business_metrics}}

### Business Metrics

Key business indicators include:

- Growth in active student accounts using the reader for textbooks.  
- Retention of students across academic terms, especially those with multiple active textbooks.  
- Depth of engagement per active user (e.g., number of sessions with highlights/notes, not just opens).  
- Conversion from casual reading to “study mode” behavior (annotations, revisits, and exam-period usage).
{{/if}}

---

## Product Scope

### MVP - Minimum Viable Product

The MVP focuses on a great web-based reading and studying experience for imported textbooks:

- **Web application** optimized for long-form, structured documents (chapters, sections, headings)
- **Supabase backend** for authentication, storage, and real-time sync (schema: `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md`)
- **Multi-device sync** for books, reading position, highlights, and notes via Supabase
- **Import** of local textbook files (PDF and EPUB formats) with upload to Supabase Storage
- **Comfortable reading UI:** sensible typography, themes (including dark mode), and distraction-minimized layout
- **Robust navigation:** table of contents, page/section navigation, and automatic remembering of last reading position per book
- **Core study tools:** text highlighting, inline notes attached to selections, and integrated dictionary lookup
- **Quick research helpers:** configurable "open in" links (e.g., Wikipedia, Google, Google Scholar) launched from inside the reader
- **Cloud-first storage** of imported books and annotations with Supabase, syncing across all user devices
- **Account authentication** via Supabase Auth with secure, user-scoped data access

### Growth Features (Post-MVP)

Growth focuses on deeper study workflows and spreading usage across devices:

- Powerful library features: tagging, search across books and notes, saved filters for courses or subjects.  
- AI-assisted features on top of existing notes and text (e.g., explain-this-section, generate study questions, summarize a chapter).  
- Better study surfaces: note views grouped by chapter, “all highlights for this exam,” and simple export of notes/citations.  
- Collaboration-oriented features such as sharing annotated copies or note bundles with classmates or study groups.

### Vision (Future)

Longer term, the product can evolve into a study operating system for textbook-based learning:

- Deep AI study companion: concept maps, adaptive practice, and guidance that sits on top of the student’s own materials.  
- Rich integrations: LMS platforms, campus systems, and potential publisher or store integrations where appropriate.  
- Advanced research workflows: cross-book concept linking, citation management, and better support for long, multi-source projects.  


---

{{#if domain_considerations}}

## Domain-Specific Requirements

{{domain_considerations}}

This section shapes all functional and non-functional requirements below.
{{/if}}

---

{{#if innovation_patterns}}

## Innovation & Novel Patterns

{{innovation_patterns}}

### Validation Approach

{{validation_approach}}
{{/if}}

---

{{#if project_type_requirements}}

## Web App Specific Requirements

For this product, the web application must support focused, long-form studying of imported textbooks in modern browsers with cloud-based sync via Supabase.

**Backend Architecture (Supabase)**

- PostgreSQL database with tables: `books`, `book_configs`, `book_notes`, `files` (schema: `readest.wiki/Supabase-Tables-Schema-for-Sync-API.md`)
- Row-level security (RLS) policies ensuring users only access their own data
- Supabase Auth for user authentication and session management
- Supabase Storage for book files and cover images
- Real-time sync capabilities for cross-device reading position and annotations

**Browser and Platform Expectations**

- Optimized for modern desktop browsers (Chrome, Edge, Safari, Firefox) with good behavior on laptop and desktop screens
- Reasonable support for mobile/tablet browsers, especially for quick reading or review, with a path to deeper mobile support later
- Installable as a Progressive Web App (PWA) so students can "add to home screen" and re-open the reader like a native app

**Responsive Layout and Reading Experience**

- Layout adapts cleanly from laptop/desktop down to tablet and larger phone screens, always prioritizing legibility and focus on the text
- Reading UI minimizes distracting chrome while keeping core controls (navigation, highlights, notes, search) easily accessible
- Multi-column or alternative layouts for wide screens may be considered only if they improve the reading experience for textbooks

**Performance and Perceived Responsiveness**

- Opening books, scrolling, and navigating between sections feels smooth, even for large textbook files
- Core interactions (highlight, add note, open dictionary, open quick link) respond quickly enough that they do not interrupt reading flow
- Supabase sync happens seamlessly in the background without blocking the reading experience
- Optimistic UI updates for annotations provide immediate feedback while syncing to backend

**SEO and Public Surface**

- The core reader is a logged-in, app-like experience; SEO is not a primary concern for the reading interface itself
- Public pages (e.g., marketing site, landing page) can be handled separately and are not a constraint on the reader architecture

**Accessibility Expectations**

- Basic accessibility from day one: readable contrast, scalable text, keyboard navigation of core actions, and no blocking interaction patterns
- The design should not make future deeper accessibility work (including stronger screen reader support) unnecessarily difficult

{{#if endpoint_specification}}

### API Specification

{{endpoint_specification}}
{{/if}}

{{#if authentication_model}}

### Authentication & Authorization

{{authentication_model}}
{{/if}}

{{#if platform_requirements}}

### Platform Support

{{platform_requirements}}
{{/if}}

{{#if device_features}}

### Device Capabilities

{{device_features}}
{{/if}}

{{#if tenant_model}}

### Multi-Tenancy Architecture

{{tenant_model}}
{{/if}}

{{#if permission_matrix}}

### Permissions & Roles

{{permission_matrix}}
{{/if}}
{{/if}}

---

{{#if ux_principles}}

## User Experience Principles

The reader should feel calm, focused, and reliable—more like a good physical study desk than a busy web app. The primary goal of the UX is to keep the student immersed in the text while making study actions (highlighting, notes, lookup) feel like natural extensions of reading, not context switches.

Key principles:

- Text-first: page layout, chrome, and controls all serve the reading experience; nothing competes visually with the text.  
- Low-friction study: creating and revisiting highlights/notes should be a near-effortless part of reading.  
- Predictability: navigation, offline behavior, and saving should feel trustworthy, with clear feedback when something important happens.  
- Gentle, not flashy: animations and visual effects are subtle, avoiding anything that feels like “appiness” over study focus.  

### Key Interactions

Core interaction patterns:

- Selecting text to trigger a small, focused toolbar for highlight, note, and dictionary lookup.  
- Simple navigation model: table of contents, previous/next section, and “jump back to last position” affordances.  
- A notes/highlights view that lets students quickly jump back into context in the book from their annotations.  
- Easy access to quick research links from the selected text without losing the current reading position.  
{{/if}}

---

## Functional Requirements

**Reading and Library**

- FR1: Students can create an account and sign in to access their personal library and study data.  
- FR2: Students can import supported e-book files (e.g., PDFs, and later EPUB) into their personal library.  
- FR3: Students can see a list of imported books with basic metadata (title and file name at minimum).  
- FR4: Students can open any imported book in the reader.  
- FR5: The system remembers the last reading position for each book and returns the student there when reopening it.  

**In-Reader Experience**

- FR6: Students can read textbooks in a comfortable, distraction-minimized reading view.  
- FR7: Students can navigate within a book using a table of contents (when present) and basic next/previous controls.  
- FR8: Students can jump to a specific location using page or section-like navigation appropriate to the book format.  
- FR9: Students can adjust basic reading settings such as theme (e.g., light/dark) and potentially text size.  

**Highlighting, Notes, and Review**

- FR10: Students can select text and create a highlight.  
- FR11: Students can select text and attach a note to that selection.  
- FR12: Students can view, edit, or delete existing highlights and notes.  
- FR13: Students can open a consolidated view of highlights and notes for a given book.  
- FR14: From the consolidated view, students can jump back into the book at the relevant location.  

**Dictionary and Quick Research**

- FR15: Students can look up the definition of a selected word or phrase via an integrated dictionary.  
- FR16: Students can open quick research links (e.g., search engines, Wikipedia, or similar) based on selected text.  
- FR17: Quick research actions open in a way that lets students return to the exact reading position without confusion.  

**Offline and PWA Behavior**

- FR18: Students can install the application as a PWA from supported browsers.  
- FR19: Once installed and after initial downloads, students can open previously imported books and continue reading offline.  
- FR20: Highlights and notes created while offline are stored locally and remain available on that device.  

**Account and Data Handling**

- FR21: Students can log out of their account.  
- FR22: Students can remove books from their library.  
- FR23: Students can export or delete their personal data (books and annotations) from the system, at least in a basic form.  

**Future-Oriented AI and Study Features (Post-MVP)**

- FR24: Students can ask basic clarification questions about selected text and receive an answer generated from that context.  
- FR25: Students can generate simple study aids (e.g., key-point summaries or practice questions) for a chapter or section.  

**Admin / System**

- FR26: The product team can configure which quick research providers are available by default.  
- FR27: The system can report high-level, privacy-respecting product metrics (e.g., number of active users, books with annotations) without exposing individual student content.  

---

## Non-Functional Requirements

{{#if performance_requirements}}

### Performance

The product should feel consistently responsive during core study workflows:

- Opening typical textbook-sized files and navigating between sections should complete without noticeable lag for students on common hardware and connections.  
- Interactions such as highlighting, adding notes, and opening the dictionary should respond quickly enough that they do not break reading flow.  
- PWA startup should be fast enough that students can quickly return to “where they left off” in their last-used book.  
{{/if}}

{{#if security_requirements}}

### Security

Because students will import course materials and create personal study notes, the system must handle data respectfully and securely:

- All communication between client and server uses secure transport.  
- Authentication and session handling follow modern best practices.  
- Access to books and notes is scoped to the owning student account, with no unintended sharing.  
- Any analytics or product metrics must avoid exposing individual student content or sensitive annotations.  
{{/if}}

{{#if scalability_requirements}}

### Scalability

The system should handle growth from early adopters to a broader student base without major rework:

- Architecture should support adding more storage and compute capacity as the number of students and books grows.  
- Core workflows (import, reading, annotations) should remain stable even as usage increases.  
{{/if}}

{{#if accessibility_requirements}}

### Accessibility

Accessibility is important to support a wide range of students:

- Core reading and navigation flows should be reachable via keyboard.  
- Text and UI elements should meet sensible contrast guidelines for typical themes.  
- The design should avoid patterns that significantly hinder screen readers or later, deeper accessibility work.  
{{/if}}

{{#if integration_requirements}}

### Integration

In early versions, integration needs are minimal:

- The system may offer simple account-based sign-in without tight coupling to institutional systems.  
- Future integrations with LMS or campus systems should be possible without rewriting core reader behavior.  
{{/if}}

{{#if no_nfrs}}
_No specific non-functional requirements identified for this project type._
{{/if}}

---

_This PRD captures the essence of {{project_name}} - {{product_value_summary}}_

_Created through collaborative discovery between {{user_name}} and AI facilitator._
