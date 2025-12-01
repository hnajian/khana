# 5. Show/Hide Header and Footer Widgets

**Added**: v0.9.23 (Commit 48074f0f, #620, #602)

## Overview

Provides granular control over reader interface elements, allowing users to show or hide header and footer widgets, customize progress indicators, and choose between distraction-free or information-rich reading experiences.

## Settings Location

**Path**: Settings Dialog → Layout Panel → Header & Footer section

**File**: `src/components/settings/LayoutPanel.tsx` (lines 603-692)

## Available Settings

### 1. Show Header (Toggle)

**Default**: Enabled in paginated mode, disabled in scrolled mode

```typescript
<input
  type="checkbox"
  checked={viewSettings.showHeader}
  onChange={(e) => updateSetting('showHeader', e.target.checked)}
/>
```

**When Enabled**:
- Section/chapter title displayed at top
- Fixed position header bar
- Minimum top margin enforced

**When Disabled**:
- No header bar
- More vertical space for content
- Cleaner reading interface

### 2. Show Footer (Toggle)

**Default**: Enabled in paginated mode, disabled in scrolled mode

```typescript
<input
  type="checkbox"
  checked={viewSettings.showFooter}
  onChange={(e) => updateSetting('showFooter', e.target.checked)}
/>
```

**When Enabled**:
- Progress indicators at bottom
- Page numbers or percentage
- Remaining time/pages (if enabled)
- Minimum bottom margin enforced

**When Disabled**:
- No footer bar
- Maximum vertical space
- Immersive reading experience

### 3. Show Remaining Time (Toggle)

**Default**: Enabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showRemainingTime}
  disabled={!viewSettings.showFooter}
  onChange={(e) => updateSetting('showRemainingTime', e.target.checked)}
/>
```

**Calculation** (`src/app/reader/components/FooterBar.tsx`):
```typescript
const calculateRemainingTime = (
  remainingPages: number,
  averageReadingSpeed: number  // pages per minute
): string => {
  const minutes = Math.round(remainingPages / averageReadingSpeed);

  if (minutes < 60) {
    return `${minutes} min left`;
  } else {
    const hours = Math.floor(minutes / 60);
    const mins = minutes % 60;
    return `${hours}h ${mins}m left`;
  }
};
```

**Display Format**:
- **Less than 1 hour**: "42 min left"
- **More than 1 hour**: "2h 15m left"
- **Scope**: Current chapter or entire book (configurable)

### 4. Show Remaining Pages (Toggle)

**Default**: Enabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showRemainingPages}
  disabled={!viewSettings.showFooter}
  onChange={(e) => updateSetting('showRemainingPages', e.target.checked)}
/>
```

**Display Format**:
- "123 pages left in chapter"
- "456 pages left in book"

### 5. Show Reading Progress (Toggle)

**Default**: Enabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showProgress}
  disabled={!viewSettings.showFooter}
  onChange={(e) => updateSetting('showProgress', e.target.checked)}
/>
```

**Components**:
- Progress bar (visual indicator)
- Numeric progress (page number or percentage)

### 6. Reading Progress Style (Radio/Select)

**Default**: "Page Number"

**Options**:
- **Page Number**: "12 / 345" (current page / total pages)
- **Percentage**: "3.5%" (percentage through book/chapter)

```typescript
<select
  value={viewSettings.progressStyle}
  disabled={!viewSettings.showProgress}
  onChange={(e) => updateSetting('progressStyle', e.target.value)}
>
  <option value="pageNumber">{_('Page Number')}</option>
  <option value="percentage">{_('Percentage')}</option>
</select>
```

**Implementation**:
```typescript
const getProgressDisplay = () => {
  if (viewSettings.progressStyle === 'percentage') {
    const percent = ((currentPage / totalPages) * 100).toFixed(1);
    return `${percent}%`;
  } else {
    return `${currentPage} / ${totalPages}`;
  }
};
```

### 7. Apply Also in Scrolled Mode (Toggle)

**Default**: Disabled

```typescript
<input
  type="checkbox"
  checked={viewSettings.showWidgetsInScrolledMode}
  onChange={(e) => updateSetting('showWidgetsInScrolledMode', e.target.checked)}
/>
```

**Purpose**: By default, header/footer are hidden in scrolled mode for distraction-free reading. This option forces them to display even in scrolled mode.

## Margin Behavior

**Header Margin Enforcement**:
```typescript
const getMinTopMargin = (): number => {
  if (!viewSettings.showHeader) return 0;

  const headerHeight = 44; // px
  const safeAreaTop = gridInsets.top;
  const requiredMargin = headerHeight - safeAreaTop;

  return Math.max(0, Math.round(requiredMargin / 4) * 4); // Round to nearest 4px
};
```

**Footer Margin Enforcement**:
```typescript
const getMinBottomMargin = (): number => {
  if (!viewSettings.showFooter) return 0;

  const footerHeight = 44; // px
  const safeAreaBottom = gridInsets.bottom;
  const requiredMargin = footerHeight - safeAreaBottom;

  return Math.max(0, Math.round(requiredMargin / 4) * 4);
};
```

## UI Components

**Header Component** (`src/app/reader/components/SectionInfo.tsx`):
```typescript
const SectionInfo = () => {
  const { viewSettings } = useReaderStore();

  if (!viewSettings.showHeader) return null;
  if (viewSettings.scrolled && !viewSettings.showWidgetsInScrolledMode) {
    return null;
  }

  return (
    <div className="section-info">
      {currentSection.title}
    </div>
  );
};
```

**Footer Component** (`src/app/reader/components/PageInfo.tsx`):
```typescript
const PageInfo = () => {
  const { viewSettings } = useReaderStore();

  if (!viewSettings.showFooter) return null;
  if (viewSettings.scrolled && !viewSettings.showWidgetsInScrolledMode) {
    return null;
  }

  return (
    <div className="page-info">
      {viewSettings.showProgress && <Progress />}
      {viewSettings.showRemainingTime && <RemainingTime />}
      {viewSettings.showRemainingPages && <RemainingPages />}
    </div>
  );
};
```

## Preset Configurations

**Minimal (Distraction-Free)**:
```typescript
{
  showHeader: false,
  showFooter: false,
  showWidgetsInScrolledMode: false
}
```

**Information-Rich**:
```typescript
{
  showHeader: true,
  showFooter: true,
  showRemainingTime: true,
  showRemainingPages: true,
  showProgress: true,
  progressStyle: 'pageNumber',
  showWidgetsInScrolledMode: true
}
```

**Balanced (Default)**:
```typescript
{
  showHeader: true,
  showFooter: true,
  showRemainingTime: true,
  showRemainingPages: false,
  showProgress: true,
  progressStyle: 'pageNumber',
  showWidgetsInScrolledMode: false
}
```

---
