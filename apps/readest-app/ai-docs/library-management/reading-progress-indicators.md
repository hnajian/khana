# Reading Progress Indicators

**Added**: Commit 46fdea35

## Overview

Visual progress indicators display reading completion percentage for each book in the library view. This helps users quickly identify partially-read books and track reading progress across their collection.

**Location**: `src/app/library/components/ReadingProgress.tsx`, `src/app/library/components/BookItem.tsx`

## Implementation

**Component Structure:**
```typescript
interface ReadingProgressProps {
  book: Book;
}

const ReadingProgress: React.FC<ReadingProgressProps> = ({ book }) => {
  const progressPercentage = useMemo(() => getProgressPercentage(book), [book]);

  if (progressPercentage === null || Number.isNaN(progressPercentage)) {
    return null;
  }

  return (
    <div className='text-neutral-content/70 flex justify-between text-xs'>
      <span>{progressPercentage}%</span>
    </div>
  );
};
```

**Progress Calculation:**
- Reads from `book.progress` array: `[currentPage, totalPages]`
- Calculates percentage: `(currentPage / totalPages) * 100`
- Clamps between 0-100%
- Returns `null` if progress data unavailable

**Display Behavior:**
- Shows in book card footer (both grid and list modes)
- Updates when book progress changes
- Hidden if no progress data exists
- Memoized to prevent unnecessary recalculations

**Integration:**
- Displayed in `BookItem` component (line 113)
- Positioned left-aligned in card footer
- Shares footer space with cloud sync indicators
- Responsive text sizing based on view mode

## Modification Guidelines

**To change progress display format:**
```typescript
// Show as fraction instead of percentage
<span>{book.progress[0]} / {book.progress[1]}</span>

// Show with completion status
<span>
  {progressPercentage}% {progressPercentage === 100 ? '✓' : ''}
</span>
```

**To add visual progress bar:**
```typescript
<div className='w-full bg-gray-200 rounded'>
  <div
    className='bg-blue-500 h-1 rounded'
    style={{ width: `${progressPercentage}%` }}
  />
</div>
```

**To filter books by progress:**
```typescript
// In libraryStore
getBooksByProgress: (min: number, max: number) => {
  const { library } = get();
  return library.filter(book => {
    const percentage = getProgressPercentage(book);
    return percentage !== null && percentage >= min && percentage <= max;
  });
};
```

---

**Related**: [index.md](./index.md) (Main library management documentation)
