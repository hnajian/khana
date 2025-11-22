# Custom CSS Editor Enhancements

**Added**: January 2025 (commit 2c9fe8e4)

## Overview

The Custom CSS editor received significant UX improvements in commit 2c9fe8e4, transforming it from an immediate-apply system to a draft-based editor with explicit validation and Apply button.

**Location**: `src/app/reader/components/settings/MiscPanel.tsx` (Misc settings panel)

## Key Improvements

| Feature | Before | After |
|---------|--------|-------|
| **Validation** | Regex-based, limited errors | Dedicated `cssValidate()` utility with detailed errors |
| **Application** | Immediate on every keystroke | Explicit "Apply" button |
| **State Management** | Direct state updates | Draft state + saved state separation |
| **Error Feedback** | Generic messages | Specific error messages per validation rule |
| **UX** | Confusing immediate changes | Clear save workflow |

## CSS Validation (`src/utils/css.ts`)

**Validation Rules**:
1. Comment removal (`/* ... */`)
2. Brace balancing (`{` equals `}`)
3. Rule structure (selector + declarations)
4. Selector validation (non-empty)
5. Declaration validation (non-empty)
6. Property validation (proper format with colons)

**Error Messages**:
- "Empty CSS"
- "Unbalanced curly braces"
- "Invalid CSS structure"
- "Missing selector"
- "Missing declarations for selector: {selector}"
- "Invalid property: {property}"

## Usage Pattern

```typescript
const [draftStylesheet, setDraftStylesheet] = useState(userStylesheet);
const [draftStylesheetSaved, setDraftStylesheetSaved] = useState(true);
const [error, setError] = useState<string | null>(null);

const handleUserStylesheetChange = (e) => {
  const css = e.target.value;
  setDraftStylesheet(css);
  setDraftStylesheetSaved(false);

  const { isValid, error } = cssValidate(css);
  setError(error);
};

const applyStyles = () => {
  if (error) return;

  const formatted = cssbeautify(draftStylesheet, {
    indent: '  ',
    openbrace: 'end-of-line',
  });

  setViewSettings({ userStylesheet: formatted });
  setDraftStylesheet(formatted);
  setDraftStylesheetSaved(true);
};
```

## Button State Management

- **Hidden**: When `draftStylesheetSaved === true` (no changes)
- **Disabled**: When `error !== null` (validation failed)
- **Enabled**: When changes exist and validation passes

---

**Related**: [index.md](./index.md) (Main settings system documentation)
