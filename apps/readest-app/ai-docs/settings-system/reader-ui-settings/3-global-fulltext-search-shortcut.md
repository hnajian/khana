# 3. Global Fulltext Search Shortcut

**Added**: v0.9.27 (Commit 01ad18ca, #750)

## Overview

The Global Fulltext Search Shortcut provides a standard `Ctrl+F` (Windows/Linux) or `Cmd+F` (macOS) keyboard shortcut to open the search interface, making Readest more intuitive for users familiar with browser search functionality.

## Keyboard Shortcut

- **Windows/Linux**: `Ctrl + F`
- **macOS**: `Cmd + F`

## Implementation

**File**: `src/app/reader/hooks/useBookShortcuts.ts`

```typescript
import { useShortcuts } from '@/hooks/useShortcuts';

export const useBookShortcuts = () => {
  const shortcuts = useShortcuts();

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      // Ctrl+F or Cmd+F
      if ((e.ctrlKey || e.metaKey) && e.key === 'f') {
        e.preventDefault();
        shortcuts.onToggleSearchBar();
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, []);
};
```

**Shortcut Handler** (`src/hooks/useShortcuts.ts`):
```typescript
const showSearchBar = () => {
  eventDispatcher.dispatch('search', { term: '' });
};

export const useShortcuts = () => ({
  onToggleSearchBar: showSearchBar,
  // ... other shortcuts
});
```

## Event Flow

1. User presses `Ctrl+F` / `Cmd+F`
2. Event is captured by `useBookShortcuts`
3. Browser's default find-in-page is prevented
4. `showSearchBar()` is called
5. 'search' event is dispatched
6. Search sidebar tab is activated
7. Search input receives focus

## Context-Aware Behavior

**When Focus Is On**:
- **Book content**: Opens search sidebar
- **Input/textarea**: Allows browser default (except note editor)
- **Note editor**: Opens search sidebar (special case)

```typescript
const isInputFocused = () => {
  const activeElement = document.activeElement;
  const tagName = activeElement?.tagName.toLowerCase();

  if (tagName === 'input' || tagName === 'textarea') {
    // Exception: note editor
    if (activeElement?.classList.contains('note-editor')) {
      return false;
    }
    return true;
  }
  return false;
};

const handleKeyDown = (e: KeyboardEvent) => {
  if (isInputFocused()) return;  // Don't interfere with input fields

  if ((e.ctrlKey || e.metaKey) && e.key === 'f') {
    e.preventDefault();
    showSearchBar();
  }
};
```

## Cross-Platform Support

**Modifier Key Detection**:
```typescript
const isMac = navigator.platform.toUpperCase().indexOf('MAC') >= 0;
const modifierKey = isMac ? 'Cmd' : 'Ctrl';

// Display in UI
<kbd>{modifierKey}+F</kbd>
```

## Accessibility

- **Keyboard-only users**: Can quickly access search without mouse
- **Screen readers**: Announces "Search activated" when triggered
- **Focus management**: Search input receives focus immediately

## Related Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl/Cmd + F` | Open search |
| `Ctrl/Cmd + G` | Find next (browser default) |
| `Ctrl/Cmd + Shift + G` | Find previous (browser default) |
| `Esc` | Close search sidebar |

---
