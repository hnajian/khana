# Keyboard Shortcuts for Annotations

**Added**: v0.9.11 (commit #378)

## Overview

Keyboard shortcuts provide quick access to annotation features, significantly improving reading workflow efficiency. Users can trigger annotation actions without leaving the keyboard.

## Available Shortcuts

| Shortcut | Action | Context | Added |
|----------|--------|---------|-------|
| `H` | Toggle highlight | Text selected | v0.9.11 |
| `N` | Add/edit note | Text selected or highlight clicked | v0.9.11 |
| `D` | Delete highlight | Highlight clicked | v0.9.11 |
| `C` | Copy to notebook | Text selected | v0.9.11 |
| `T` | Translate selection | Text selected | v0.9.11 |
| `W` | Wikipedia lookup | Text selected | v0.9.11 |
| `S` | Text-to-speech | Text selected | v0.9.11 |

## Implementation

**Location**: `src/app/reader/components/annotator/Annotator.tsx`

**Event Handler**:
```typescript
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    if (!showAnnotPopup) return;

    switch (e.key.toLowerCase()) {
      case 'h':
        handleHighlight();
        break;
      case 'n':
        handleAddNote();
        break;
      case 'd':
        handleDeleteHighlight();
        break;
      case 'c':
        handleCopyToNotebook();
        break;
      case 't':
        handleTranslate();
        break;
      case 'w':
        handleWikipediaLookup();
        break;
      case 's':
        handleTextToSpeech();
        break;
    }
  };

  window.addEventListener('keydown', handleKeyDown);
  return () => window.removeEventListener('keydown', handleKeyDown);
}, [showAnnotPopup]);
```

## Visual Indicators

Keyboard shortcuts are displayed in the UI using `<kbd>` tags (commit #421):

```typescript
<button>
  Highlight <kbd>H</kbd>
</button>
```

This renders as a visually distinct keyboard key indicator, improving discoverability.

## Customization

**To add a new keyboard shortcut**:

1. Add new case in `handleKeyDown`:
   ```typescript
   case 'e':
     handleExport();
     break;
   ```

2. Update UI button to show shortcut:
   ```typescript
   <PopupButton onClick={handleExport}>
     Export <kbd>E</kbd>
   </PopupButton>
   ```

3. Document in shortcuts table above

**To modify existing shortcuts**:

1. Change the key in switch statement
2. Update UI labels
3. Test for conflicts with browser shortcuts (avoid Cmd/Ctrl combinations)

## Accessibility

- Shortcuts only active when annotation popup is visible
- Non-conflicting with browser native shortcuts
- Visual indicators help users discover shortcuts
- Keyboard-only navigation fully supported

---

**Related**: [index.md](./index.md) (Main annotation system documentation)
