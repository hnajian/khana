# Keyboard Shortcuts for Annotations

**Added**: v0.9.11 (commit #378)

## Overview

Keyboard shortcuts provide quick access to annotation features, significantly improving reading workflow efficiency. Users can trigger annotation actions without leaving the keyboard.

## Available Shortcuts

### Annotation Shortcuts

| Shortcut | Action | Context | Added |
|----------|--------|---------|-------|
| `H` | Toggle highlight | Text selected | v0.9.11 |
| `N` | Add/edit note | Text selected or highlight clicked | v0.9.11 |
| `D` | Delete highlight | Highlight clicked | v0.9.11 |
| `C` | Copy to notebook | Text selected | v0.9.11 |
| `T` | Translate selection | Text selected | v0.9.11 |
| `W` | Wikipedia lookup | Text selected | v0.9.11 |
| `S` | Text-to-speech | Text selected | v0.9.11 |

### Navigation Shortcuts (v0.9.83+)

| Shortcut | Action | Context | Added |
|----------|--------|---------|-------|
| `Space` | Navigate forward (next page) | Reader | v0.9.83 |
| `Shift+Space` | Navigate backward (previous page) | Reader | v0.9.83 (#2176) |
| `Ctrl/Cmd+[` | Navigate to previous section/chapter | Reader | v0.9.88 (#2291) |
| `Ctrl/Cmd+]` | Navigate to next section/chapter | Reader | v0.9.88 (#2291) |
| `Arrow Up` | Scroll up (improved smoothness) | Reader, scrolled mode | v0.9.88 (#2306) |
| `Arrow Down` | Scroll down (improved smoothness) | Reader, scrolled mode | v0.9.88 (#2306) |

### UI Control Shortcuts (v0.9.83+)

| Shortcut | Action | Context | Added |
|----------|--------|---------|-------|
| `Ctrl/Cmd+B` | Toggle bookmarks sidebar | Reader | v0.9.83 (#2170) |
| `Ctrl/Cmd+Shift+S` | Toggle sidebar visibility | Reader | v0.9.83 (#2170) |
| `Ctrl/Cmd+I` | Import books dialog | Library or Reader | v0.9.83 (#2170) |
| `Ctrl/Cmd+W` | Close current window | Reader | v0.9.83 (#2170) |

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

## GNOME Human Interface Guidelines Compatibility (v0.9.83, #2170)

The keyboard shortcuts have been updated to comply with GNOME Human Interface Guidelines for better Linux desktop integration:

- Consistent shortcut patterns across platforms
- Standard system shortcuts respected
- No conflicts with desktop environment shortcuts
- Cross-platform compatibility (Windows, macOS, Linux)

**Platform-Specific Behavior**:
- macOS: Uses `Cmd` key
- Windows/Linux: Uses `Ctrl` key
- Automatic detection and adaptation

**Implementation**:
```typescript
const getModifierKey = () => {
  return navigator.platform.includes('Mac') ? 'metaKey' : 'ctrlKey';
};

const handleKeyDown = (e: KeyboardEvent) => {
  const modKey = getModifierKey();

  if (e[modKey] && e.key === 'b') {
    toggleBookmarksSidebar();
    e.preventDefault();
  }
};
```

## Version 0.9.83 - 0.9.90 Updates

### Enhanced Navigation (v0.9.83-0.9.88)

**Shift+Space for Backward Navigation** (#2176):
- Complements existing Space key for forward navigation
- Consistent with browser behavior
- Works in both paginated and scrolled modes

**Section Navigation** (#2291):
- Jump to previous/next chapter or section
- Faster navigation in long books
- Keyboard-only chapter browsing

**Smooth Scrolling** (#2306):
- Improved Arrow Up/Down scrolling performance
- Reduced jank and stutter
- Better responsiveness in scrolled mode

### UI Control Shortcuts (v0.9.83, #2170)

**Bookmark Management**:
- Quick toggle bookmarks panel without mouse
- Consistent with other reader applications
- Keyboard-first workflow support

**Sidebar Toggle**:
- Hide/show sidebar to maximize reading space
- Distraction-free reading mode
- Quick access to TOC, search, and notes

**Import Books**:
- Open import dialog from anywhere
- Faster book management workflow
- Library building efficiency

**Close Window**:
- Standard Ctrl/Cmd+W behavior
- Consistent with browser tabs
- Clean application exit

## Accessibility

- Shortcuts only active when annotation popup is visible (for annotation shortcuts)
- Non-conflicting with browser native shortcuts
- Visual indicators help users discover shortcuts
- Keyboard-only navigation fully supported
- Cross-platform shortcut consistency
- Screen reader compatible

## Tips for Users

1. **Discover Shortcuts**: Hover over buttons to see keyboard shortcuts in tooltips
2. **Customize Workflow**: Combine shortcuts for efficient reading (e.g., Space → H → N for highlight and note)
3. **Platform Awareness**: Shortcuts automatically adapt to your operating system
4. **Learning Curve**: Start with basic navigation (Space, Shift+Space) then add annotation shortcuts

---

**Related**: [index.md](./index.md) (Main annotation system documentation)
