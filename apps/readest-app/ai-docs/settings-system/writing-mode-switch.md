# Vertical/Horizontal Layout Switch

**Added**: January 2025 (commit 3ad26d9d)

## Overview

The vertical/horizontal layout switch allows users to manually override text direction for CJK (Chinese, Japanese, Korean) books.

**Location**: `src/app/reader/components/settings/LayoutPanel.tsx` (Writing Mode section)

## Writing Modes

| Mode | Value | Description | Icon |
|------|-------|-------------|------|
| **Auto** | `'auto'` | Uses document's natural writing direction | `MdOutlineAutoMode` |
| **Horizontal** | `'horizontal-tb'` | Forces horizontal text (left-to-right, top-to-bottom) | `MdOutlineTextRotationNone` |
| **Vertical** | `'vertical-rl'` | Forces vertical text (right-to-left, top-to-bottom) | `MdOutlineTextRotationDown` |

## Feature Characteristics

**Visibility**: Only shown for CJK books (detected via language code)

```typescript
const langCode = getBookLangCode(bookData.bookDoc?.metadata?.language);
const isCJKBook = langCode === 'zh' || langCode === 'ja' || langCode === 'ko';
```

**Type Definition**:

```typescript
export interface BookLayout {
  writingMode: string; // 'auto' | 'horizontal-tb' | 'vertical-rl'
  vertical: boolean;   // Auto-detected from document
  // ... other properties
}
```

**CSS Application** (`src/utils/style.ts`):

```css
html, body {
  ${writingMode === 'auto' ? '' : `writing-mode: ${writingMode};`}
}
```

## Differences from Existing `vertical` Property

| Aspect | `vertical` (existing) | `writingMode` (new) |
|--------|----------------------|---------------------|
| **Source** | Auto-detected from document | User-controlled setting |
| **Type** | Boolean | String enum |
| **Purpose** | Layout calculations (e.g., footnote positioning) | Override document text direction |
| **Persistence** | Not persisted | Saved per book |
| **When Set** | On document load | User selection in settings |

## Persistence

- **Per-Book Only**: Not available in global settings
- **Storage**: Saved in `BookConfig.viewSettings.writingMode`
- **Default**: `'auto'` (defined in `src/services/constants.ts`)

## Use Case

Useful when:
- Book has incorrect metadata about text direction
- User prefers alternate reading direction (e.g., horizontal for traditionally vertical text)
- Testing different reading modes for CJK content

---

**Related**: [index.md](./index.md) (Main settings system documentation)
