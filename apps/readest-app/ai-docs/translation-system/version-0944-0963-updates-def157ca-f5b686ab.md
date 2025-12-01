# Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

## Bilingual Translation (v0.9.49, #1240)

**Major Feature**: Support for full book translation to bilingual format.

**Overview**: Users can now translate entire books or sections into bilingual format, showing both original and translated text side-by-side.

**Key Features**:
- Full book translation with progress tracking
- Side-by-side bilingual display
- Translation caching for performance
- Automatic language pair detection

**Files Updated**:
- `src/app/reader/hooks/useTextTranslation.ts` - Full-page translation hook
- `src/app/reader/components/annotator/TranslatorPopup.tsx` - Bilingual translation UI
- `src/services/translators/` - Provider support for book-length translations

**Usage**:
1. Open translation panel
2. Select "Translate Full Book" or "Translate Section"
3. Choose target language
4. Translation processes in background with progress indicator
5. View bilingual text with toggle between original/translated/both

## TOC Translation (v0.9.50, #1273)

**Feature**: Table of contents is now also translated when full book translation is enabled.

**Implementation**:
- TOC items translated using same provider as book content
- Translations cached to avoid repeated API calls
- Maintains original TOC structure and navigation links

**Benefits**:
- Complete bilingual reading experience
- Better navigation in translated books
- Consistent language throughout interface

## Hide Original Text in Translation (v0.9.62, #1511)

**Feature**: Option to show/hide original text when viewing translations.

**UI**: Toggle button in translator popup and settings panel.

**Modes**:
- **Both**: Show original and translation side-by-side (default)
- **Translation Only**: Hide original text, show only translation
- **Original Only**: Hide translation, show only original

**File**: `src/app/reader/components/annotator/TranslatorPopup.tsx`

**Settings**: Preference saved per book in BookConfig.

## Translation Header Hide on Mobile (v0.9.50, #1274)

**Feature**: Header bar automatically hides on mobile platforms when translation is enabled to maximize reading space.

**Implementation**:
- Platform detection for mobile devices
- Auto-hide header when translation panel is active
- Swipe gesture to restore header temporarily

**Files**:
- `src/app/reader/components/Header.tsx`
- `src/services/environment.ts` - Platform detection

## Daily Translation Quota Management (v0.9.50-0.9.62, #1275, #1349, #1363)

**Enhanced Quota System**: More sustainable DeepL API usage with daily quotas.

**Features**:
- Free plan: 500K characters/month
- Pro plan: Custom limits based on subscription
- Daily quota tracking per user
- Toast notification when quota exceeded: "Daily translation quota reached. Upgrade your plan or use a different translator."
- Automatic fallback to Azure Translator
- Quota reset at midnight UTC

**Implementation** (`src/pages/api/deepl/translate.ts`):
```typescript
const DAILY_QUOTAS = {
  free: 500_000 / 30,      // ~16K chars/day
  pro: 2_000_000 / 30,     // ~66K chars/day
  premium: Infinity
};

const checkQuota = async (userId: string, plan: string) => {
  const usage = await getUsageToday(userId);
  const limit = DAILY_QUOTAS[plan];

  if (usage >= limit) {
    throw new QuotaExceededError('Daily translation quota reached');
  }
};
```

**User Experience**:
1. User translates text
2. If quota exceeded, toast notification appears
3. Translator automatically switches to Azure
4. User can continue translating without interruption
5. Next day, DeepL quota resets

## Translation Lazy Loading (v0.9.48, #1282)

**Optimization**: Translation observer now lazy loads to reduce initial page load time.

**Implementation**:
- Translation functionality loaded on-demand
- Intersection Observer for visible text translation
- Reduces bundle size and initial load time
- Better performance on low-end devices

**File**: `src/app/reader/hooks/useTextTranslation.ts`

## Translation Post-Processing (v0.9.48, #1245)

**Feature**: Improved punctuation spacing in translated text for better readability.

**Language-Specific Rules**:
- **Chinese/Japanese**: Remove spaces before punctuation marks
- **English**: Ensure space after punctuation
- **French**: Space before/after guillemets (« »)
- **Korean**: Remove spaces around punctuation

**File**: `src/services/translators/polish.ts`

**Example**:
```typescript
// Before: "Hello , how are you ?"
// After:  "Hello, how are you?"

// Before: "你好 ， 你好吗 ？"
// After:  "你好，你好吗？"
```

## Responsive Translator Popup (v0.9.43, #1160)

**Enhancement**: Translator popup now fully responsive on mobile devices.

**Improvements**:
- Adaptive layout for small screens
- Touch-friendly buttons and controls
- Better positioning on tablets
- Swipe gestures for dismissal

---
