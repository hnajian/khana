# 8. Separate Header/Footer Visibility for Reading Modes

**Added**: v0.9.32 (Commit ccd467eb, #859)

## Overview

This feature allows independent control of header and footer visibility for paginated and scrolled modes, enabling users to have different UI configurations for each reading mode.

## Settings Structure

**Type Definition** (`src/types/book.ts`):
```typescript
export interface ViewSettings {
  // Paginated mode widgets
  showHeader: boolean;
  showFooter: boolean;

  // Scrolled mode widgets (separate controls)
  showHeaderInScrolled: boolean;
  showFooterInScrolled: boolean;

  // OR use the existing approach:
  showWidgetsInScrolledMode: boolean;  // Apply paginated settings to scrolled

  // ... other settings
}
```

## Default Behavior

**Paginated Mode** (Default):
- Header: Shown
- Footer: Shown
- Full UI with section title and progress

**Scrolled Mode** (Default):
- Header: Hidden (unless `showWidgetsInScrolledMode: true`)
- Footer: Hidden (unless `showWidgetsInScrolledMode: true`)
- Distraction-free scrolling experience

## Settings Location

**Path**: Settings Dialog → Layout Panel → Header & Footer section

**UI Implementation**:
```typescript
<div className="header-footer-settings">
  <h4>{_('Header & Footer Visibility')}</h4>

  {/* Paginated Mode */}
  <fieldset>
    <legend>{_('Paginated Mode')}</legend>
    <label>
      <input
        type="checkbox"
        checked={viewSettings.showHeader}
        onChange={(e) => updateSetting('showHeader', e.target.checked)}
      />
      {_('Show Header')}
    </label>
    <label>
      <input
        type="checkbox"
        checked={viewSettings.showFooter}
        onChange={(e) => updateSetting('showFooter', e.target.checked)}
      />
      {_('Show Footer')}
    </label>
  </fieldset>

  {/* Scrolled Mode */}
  <fieldset>
    <legend>{_('Scrolled Mode')}</legend>
    <label>
      <input
        type="checkbox"
        checked={viewSettings.showWidgetsInScrolledMode}
        onChange={(e) => updateSetting('showWidgetsInScrolledMode', e.target.checked)}
      />
      {_('Show Header & Footer in Scrolled Mode')}
    </label>
  </fieldset>
</div>
```

## Implementation Logic

**Conditional Rendering** (`src/app/reader/components/ReaderContent.tsx`):
```typescript
const shouldShowHeader = () => {
  if (viewSettings.scrolled) {
    return viewSettings.showWidgetsInScrolledMode;
  }
  return viewSettings.showHeader;
};

const shouldShowFooter = () => {
  if (viewSettings.scrolled) {
    return viewSettings.showWidgetsInScrolledMode;
  }
  return viewSettings.showFooter;
};

return (
  <div className="reader-content">
    {shouldShowHeader() && <HeaderBar />}
    <BookView />
    {shouldShowFooter() && <FooterBar />}
  </div>
);
```

## Use Cases

**Paginated Mode**:
- Traditional book-like experience
- Header shows chapter title
- Footer shows page numbers and progress
- Information-rich reading

**Scrolled Mode**:
- Distraction-free scrolling
- Minimal UI for immersive reading
- More like reading a web article
- Maximum vertical space

---
