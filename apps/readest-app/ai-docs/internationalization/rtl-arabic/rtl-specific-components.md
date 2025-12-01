# RTL-Specific Components

## 1. Sidebar Navigation

**File**: `src/app/reader/components/sidebar/Sidebar.tsx`

**RTL Behavior** (#504, #519):
- Sidebar position switches sides (left in LTR, right in RTL)
- Tab navigation icons mirror
- TOC indentation reverses
- Search results align to the right
- Bookmark list aligns to the right

**Conditional RTL Application**:
```typescript
// Only apply RTL to sidebar for mandatory RTL languages
const sidebarDir = MANDATORY_RTL_LANGS.includes(uiLang) ? 'rtl' : 'ltr';
```

**Mandatory RTL Languages**: `ar`, `he`, `fa`, `ur`

**Rationale**: Some languages (like Arabic) require strict RTL layout, while others may prefer LTR UI even if the book is RTL.

## 2. Progress Bar

**File**: `src/app/reader/components/FooterBar.tsx` (#498)

**RTL Behavior**:
- Progress bar fills from right to left in RTL books
- Page numbers display in reversed order (e.g., "100 of 1" instead of "1 of 100")
- Navigation buttons swap positions

**Implementation**:
```typescript
const isRTL = bookLayout.rtl;

<div className={`progress-bar ${isRTL ? 'rtl' : 'ltr'}`}>
  {isRTL ? (
    <>{totalPages} {_('of')} {currentPage}</>
  ) : (
    <>{currentPage} {_('of')} {totalPages}</>
  )}
</div>
```

## 3. Notebook and Annotations

**File**: `src/app/reader/components/notebook/NotebookView.tsx` (#535)

**RTL Behavior**:
- Annotation list aligns to the right
- Note text aligns to the right
- Timestamps align to the left (reversed)
- Action buttons swap positions

**Conditional Rendering**:
```typescript
const dir = getDirFromUILanguage();

<div className="notebook-container" dir={dir}>
  {/* Content automatically mirrors in RTL */}
</div>
```

## 4. Go Back/Forward Buttons

**File**: `src/app/reader/components/FooterBar.tsx` (#551)

**RTL Behavior**:
- Button order consistent with reading direction
- In RTL: Forward (→) on left, Back (←) on right
- In LTR: Back (←) on left, Forward (→) on right

**Implementation**:
```typescript
const buttons = isRTL
  ? [forwardButton, backButton]
  : [backButton, forwardButton];
```
