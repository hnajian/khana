# 7. Prev/Next Section Navigation

**Added**: v0.9.43 (Commit 7c464b9a, #1195, #1125)

## Overview

The Prev/Next Section Navigation feature adds dedicated buttons in the footer bar for quick navigation between book sections/chapters. This complements existing page navigation by allowing users to jump directly to the previous or next section without using the table of contents.

## UI Location

**Position**: Footer bar (reader interface)
**Buttons**:
- Previous Section (⏮ icon)
- Next Section (⏭ icon)

## Implementation

**File**: `src/app/reader/components/FooterBar.tsx`

```typescript
const FooterBar = () => {
  const { sections, currentSectionIndex } = useBookNavigation();
  const hasPrevSection = currentSectionIndex > 0;
  const hasNextSection = currentSectionIndex < sections.length - 1;

  const goToPrevSection = () => {
    if (hasPrevSection) {
      const prevSection = sections[currentSectionIndex - 1];
      navigateToSection(prevSection.href);
    }
  };

  const goToNextSection = () => {
    if (hasNextSection) {
      const nextSection = sections[currentSectionIndex + 1];
      navigateToSection(nextSection.href);
    }
  };

  return (
    <div className="footer-bar">
      <button
        onClick={goToPrevSection}
        disabled={!hasPrevSection}
        aria-label="Previous section"
      >
        <FaStepBackward />
      </button>

      {/* Page info and progress */}

      <button
        onClick={goToNextSection}
        disabled={!hasNextSection}
        aria-label="Next section"
      >
        <FaStepForward />
      </button>
    </div>
  );
};
```

## Button Behavior

**Previous Section Button**:
- Navigates to the start of the previous chapter/section
- Disabled when on first section
- Gray/dimmed appearance when disabled

**Next Section Button**:
- Navigates to the start of the next chapter/section
- Disabled when on last section
- Gray/dimmed appearance when disabled

## Section Detection

**TOC-Based** (`src/app/reader/hooks/useBookNavigation.ts`):
```typescript
const getCurrentSectionIndex = (currentCfi: string): number => {
  const toc = book.getTOC();

  // Find current section based on CFI
  for (let i = 0; i < toc.length; i++) {
    const section = toc[i];
    if (isCfiBefore(currentCfi, section.cfi)) {
      return Math.max(0, i - 1);
    }
  }

  return toc.length - 1;
};
```

## Keyboard Shortcuts

While not part of the initial implementation, suggested shortcuts:
- `Alt + ←`: Previous section
- `Alt + →`: Next section

## Use Cases

**Quick Chapter Navigation**:
- Finish reading a chapter and immediately jump to next
- Return to previous chapter for reference
- Skip to specific parts without opening TOC

**Linear Reading Flow**:
- Continue reading without interruption
- Natural progression through book structure
- Maintain reading momentum

## Visual Design

**Button Styling**:
```css
.section-nav-button {
  padding: 8px 12px;
  background: transparent;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.section-nav-button:hover:not(:disabled) {
  background: var(--hover-bg);
  border-color: var(--hover-border);
}

.section-nav-button:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
```

---
