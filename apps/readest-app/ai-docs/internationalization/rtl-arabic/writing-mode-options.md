# Writing Mode Options

## Available Modes

**File**: `src/types/book.ts`

```typescript
export type WritingMode =
  | 'auto'           // Automatic detection from book metadata
  | 'horizontal-tb'  // Horizontal LTR, top-to-bottom
  | 'horizontal-rl'  // Horizontal RTL, top-to-bottom (Arabic default)
  | 'vertical-rl';   // Vertical RTL, right-to-left (CJK)
```

## Mode Selection UI

**File**: `src/app/reader/components/settings/LayoutPanel.tsx` (lines 382-419)

**UI Labels**:
- **Auto**: Uses the document's natural writing direction
- **Horizontal (LTR)**: Horizontal text, left-to-right, top-to-bottom
- **Vertical (RTL)**: Vertical text, right-to-left, top-to-bottom (for CJK)
- **RTL Direction**: Horizontal text, right-to-left, top-to-bottom (for Arabic/Hebrew)


## Configuration Storage

**File**: `src/types/book.ts`

```typescript
export interface BookLayout {
  writingMode: WritingMode;
  rtl: boolean;           // Derived from writingMode
  vertical: boolean;      // Derived from writingMode
  // ... other layout properties
}

export interface ViewSettings {
  layout: BookLayout;
  // ... other view settings
}
```

**Derivation Logic**:
```typescript
const rtl = writingMode === 'horizontal-rl' || writingMode === 'vertical-rl';
const vertical = writingMode === 'vertical-rl';
```
