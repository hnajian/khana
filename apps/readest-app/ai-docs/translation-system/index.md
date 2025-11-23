# Feature: Translation System

## Overview

The Translation System in Readest provides multi-provider translation capabilities with intelligent caching, quota management, and seamless fallback handling. Added in v0.9.32-0.9.43, the system supports 4 translation providers and 50+ languages with a two-tier caching strategy for optimal performance.

## Key Components

### Primary Files

- **`src/hooks/useTranslator.ts`** - Main translator hook orchestrating translation logic
- **`src/services/translators/cache.ts`** - Two-tier caching system (memory + IndexedDB)
- **`src/services/translators/providers/`** - Provider-specific implementations
  - `deepl.ts` - DeepL translator (authentication required)
  - `azure.ts` - Azure Translator (free, auto-token)
  - `google.ts` - Google Translate (free)
  - `yandex.ts` - Yandex Translate (free)
- **`src/app/reader/components/annotator/TranslatorPopup.tsx`** - Responsive translation UI

### Related Files

- **`src/services/translators/types.ts`** - Type definitions for translators and providers
- **`src/services/translators/preprocess.ts`** - Text preprocessing before translation
- **`src/services/translators/polish.ts`** - Post-translation text formatting
- **`src/pages/api/deepl/translate.ts`** - Backend translation proxy with KV cache
- **`src/app/reader/hooks/useTextTranslation.ts`** - Full-page text translation hook
- **`src/services/translators/lang.ts`** - Language code normalization

## Architecture

### Translation Flow

1. **Text Selection**: User selects text in the reader
2. **Preprocessing**: Optional text substitution for better context (e.g., "Cover" → "The Cover")
3. **Cache Check**: Query memory cache → IndexedDB cache
4. **Translation**: If cache miss, call selected provider
5. **Caching**: Store translation in both memory and IndexedDB
6. **Polishing**: Apply language-specific formatting rules
7. **Display**: Show translation in responsive popup

### Supported Translation Providers

| Provider | Authentication | Features | API Version |
|----------|----------------|----------|-------------|
| **DeepL** | Required (authRequired: true) | Quota tracking, daily usage limits | v2 (with v1 compatibility) |
| **Azure Translator** | Not required (auto-token) | 8-minute token expiry, auto-refresh | v3.0 |
| **Google Translate** | Not required | Platform-aware (Tauri HTTP for desktop) | N/A |
| **Yandex Translate** | Not required | Multiple service options (yandexgpt, etc.) | v2 |

#### Provider Details

**DeepL** (`providers/deepl.ts`):
- Supports both Free and Pro plans
- Uses Bearer token authentication
- Language normalization for API compatibility
- Quota exceeded detection with fallback to Azure
- API Endpoints:
  - Free: `https://api-free.deepl.com/v2/translate`
  - Pro: `https://api.deepl.com/v2/translate`

**Azure Translator** (`providers/azure.ts`):
- Auto-fetched token from Microsoft edge service
- Token expires after 8 minutes
- API: `https://api-edge.cognitive.microsofttranslator.com/translate` (v3.0)
- Language format: Full language codes (e.g., en-US, ja-JP)

**Google Translate** (`providers/google.ts`):
- Free API without authentication
- API: `https://translate.googleapis.com/translate_a/single`
- Platform support: Tauri HTTP for desktop, standard fetch for web

**Yandex Translate** (`providers/yandex.ts`):
- Free API without authentication
- API: `https://translate.toil.cc/v2/translate/`
- Service options: yandexgpt, yandextranslate, yandexcloud, yandexbrowser
- Default: Uses yandexgpt service

### Two-Tier Caching Strategy

**Cache Implementation** (`src/services/translators/cache.ts`, 594 lines):

#### Level 1: Memory Cache
- **Purpose**: Fast lookups for frequently translated texts
- **Storage**: In-memory JavaScript object
- **Features**:
  - Timestamps tracked for cache expiry
  - Instant access with zero latency
  - Cleared on app restart

#### Level 2: IndexedDB Cache
- **Purpose**: Persistent browser storage across sessions
- **Storage**: Browser IndexedDB
- **Database**: `TranslationCache` (v1)
- **Object Store**: `translations`
- **Indexes**:
  - `provider` - Filter by translator
  - `timestamp` - Age-based pruning
- **Expiry**: 90 days (default)

**Cache Key Structure**:
```typescript
`${provider}:${sourceLang}:${targetLang}:${text}`
```

**Cache Features**:
- **Batch Loading**: `loadCacheFromDB()` with provider/language filters
- **Auto-Pruning**: Configurable limits (max 10,000 entries, 10MB size, 30-day age)
- **Cache Statistics**: Track memory/DB cache sizes and entry counts
- **Smart Filtering**: Load only relevant translations on app startup

**Default Configuration**:
```typescript
preload: true
preloadOptions: { maxAge: 7 days, maxEntries: 1000 }
autoPrune: true
pruneInterval: 1 hour
pruneOptions: { maxAge: 30 days, maxEntries: 10000, maxSizeInBytes: 10MB }
```

### API Version Handling

**DeepL API v1/v2 Compatibility** (`src/pages/api/deepl/translate.ts`):

The system supports both DeepL API v1 and v2 with automatic detection and transformation:

**Version Detection**:
```typescript
const isV2Api = apiUrl.endsWith('/v2/translate');
```

**Language Code Compatibility**:
```typescript
const LANG_V2_V1_MAP: Record<string, string> = {
  'ZH-HANS': 'ZH',        // v2 Chinese Simplified → v1
  'ZH-HANT': 'ZH-TW',     // v2 Chinese Traditional → v1
};
```

**Request Format Differences**:

| Aspect | v2 API | v1 API |
|--------|--------|--------|
| Text Field | Array: `[text]` | String: `text` |
| Source Lang | Full codes (ZH-HANS) | Short codes (ZH) with mapping |
| Target Lang | Full codes (ZH-HANT) | Mapped codes (ZH-TW) |

**Handling Logic**:
1. Client sends v2-compatible format
2. Backend detects API version
3. Transforms request/response as needed
4. Maintains backward compatibility with v1 APIs

### Quota Management

**DeepL Quota Tracking**:
- Tracks daily usage per user plan (free/pro)
- Server-side tracking with `UsageStatsManager`
- Client-side localStorage backup: `translationDailyUsage`
- Quota exceeded triggers fallback to Azure translator
- Toast notification: "Daily translation quota reached. Upgrade your plan..."

**Fallback Strategy**:
1. User selects DeepL as provider
2. If quota exceeded, automatically fall back to Azure
3. Show toast notification explaining the quota limit
4. Continue translation seamlessly without user intervention

## AI Agent Modification Guidelines

### Adding a New Translation Provider

To add support for a new translation service:

1. **Create provider file** (`src/services/translators/providers/newtranslator.ts`):
   ```typescript
   import type { TranslationProvider } from '../types';

   export const newtranslatorProvider: TranslationProvider = {
     name: 'NewTranslator',
     authRequired: false, // or true if needs API key

     async translate({
       text,
       sourceLang,
       targetLang,
       authToken,
       signal
     }) {
       const response = await fetch('https://api.newtranslator.com/translate', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
           ...(authToken && { 'Authorization': `Bearer ${authToken}` })
         },
         body: JSON.stringify({
           text,
           source: sourceLang,
           target: targetLang
         }),
         signal
       });

       const data = await response.json();
       return data.translations;
     }
   };
   ```

2. **Register provider** in `src/services/translators/providers/index.ts`:
   ```typescript
   export { newtranslatorProvider } from './newtranslator';
   ```

3. **Update useTranslator hook** (`src/hooks/useTranslator.ts`):
   ```typescript
   import { newtranslatorProvider } from '@/services/translators/providers';

   const availableTranslators = [
     deeplProvider,
     azureProvider,
     googleProvider,
     yandexProvider,
     newtranslatorProvider // Add here
   ];
   ```

4. **Add to UI** in `TranslatorPopup.tsx`:
   ```typescript
   <option value="newtranslator">NewTranslator</option>
   ```

### Implementing Text Preprocessing Rules

To add custom preprocessing for better translation context:

1. **Edit `src/services/translators/preprocess.ts`**:
   ```typescript
   const DEFAULT_SUBSTITUTIONS: Record<string, string> = {
     'Cover': 'The Cover',
     'Dedication': 'Dedication Page',
     'Acknowledgements': 'The Acknowledgements',
     'Chapter': 'Book Chapter',  // New rule
     'Section': 'Book Section'   // New rule
   };
   ```

2. **Add language-specific preprocessing**:
   ```typescript
   export function preprocessText(text: string, sourceLang?: string): string {
     let processed = text;

     // Apply default substitutions
     Object.entries(DEFAULT_SUBSTITUTIONS).forEach(([key, value]) => {
       processed = processed.replace(new RegExp(`\\b${key}\\b`, 'g'), value);
     });

     // Language-specific preprocessing
     if (sourceLang === 'en') {
       processed = processed.replace(/\bDr\./g, 'Doctor');
     }

     return processed;
   }
   ```

### Adding Language-Specific Polishing

To improve translation output formatting:

1. **Edit `src/services/translators/polish.ts`**:
   ```typescript
   export function polishTranslation(
     text: string,
     targetLang: string
   ): string {
     let polished = basicPolish(text);

     // Language-specific polishing
     switch (targetLang) {
       case 'zh':
       case 'zh-hans':
       case 'zh-hant':
         return polishChinese(polished);
       case 'ja':
         return polishJapanese(polished);
       case 'ko':  // Add Korean
         return polishKorean(polished);
       default:
         return polished;
     }
   }

   function polishKorean(text: string): string {
     // Korean-specific formatting rules
     return text.replace(/\s+([,.!?])/g, '$1');
   }
   ```

### Extending Cache Configuration

To customize caching behavior:

1. **Update cache initialization** in `useTranslator.ts`:
   ```typescript
   const cache = new TranslationCache({
     preload: true,
     preloadOptions: {
       maxAge: 14 * 24 * 60 * 60 * 1000, // 14 days (instead of 7)
       maxEntries: 5000,                   // Increase from 1000
       providers: ['deepl', 'azure'],      // Only preload these
       languages: ['en', 'es', 'fr']       // Only preload these
     },
     autoPrune: true,
     pruneInterval: 2 * 60 * 60 * 1000,    // 2 hours
     pruneOptions: {
       maxAge: 60 * 24 * 60 * 60 * 1000,   // 60 days
       maxEntries: 50000,                   // Increase capacity
       maxSizeInBytes: 50 * 1024 * 1024    // 50MB
     }
   });
   ```

2. **Implement selective cache clearing**:
   ```typescript
   // Clear cache for specific provider
   await cache.clear({ provider: 'deepl' });

   // Clear cache for specific language pair
   await cache.clear({
     sourceLang: 'en',
     targetLang: 'es'
   });

   // Clear expired entries only
   await cache.prune({ maxAge: 7 * 24 * 60 * 60 * 1000 });
   ```

### Improving Quota Management

To enhance quota tracking and limits:

1. **Add per-provider quota limits**:
   ```typescript
   const PROVIDER_QUOTAS: Record<string, number> = {
     deepl: 500000,      // 500K characters/month
     azure: 2000000,     // 2M characters/month
     google: 100000,     // 100K characters/month
     yandex: Infinity    // Unlimited
   };
   ```

2. **Track usage in localStorage**:
   ```typescript
   interface UsageStats {
     provider: string;
     date: string;
     characters: number;
   }

   function trackUsage(provider: string, text: string) {
     const today = new Date().toISOString().split('T')[0];
     const key = `usage_${provider}_${today}`;
     const currentUsage = Number(localStorage.getItem(key) || 0);
     const newUsage = currentUsage + text.length;

     localStorage.setItem(key, String(newUsage));

     if (newUsage >= PROVIDER_QUOTAS[provider]) {
       // Trigger fallback
       switchToFallbackProvider();
     }
   }
   ```

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add translation provider | `providers/newtranslator.ts` | `useTranslator.ts`, `TranslatorPopup.tsx` |
| Modify caching strategy | `cache.ts` | `useTranslator.ts` |
| Update preprocessing | `preprocess.ts` | `useTranslator.ts` |
| Add polishing rules | `polish.ts` | `useTranslator.ts` |
| Customize UI | `TranslatorPopup.tsx` | `lang.ts` |
| Backend proxy changes | `pages/api/deepl/translate.ts` | Provider files |
| Quota management | `useTranslator.ts` | `pages/api/deepl/translate.ts` |

## Common Issues and Debugging

### Problem: Translation not cached

- Check if cache is enabled: `cache.isEnabled()`
- Verify IndexedDB is accessible in browser DevTools
- Check cache key format matches: `provider:sourceLang:targetLang:text`
- Ensure cache is not full (check pruning configuration)

### Problem: Provider fallback not working

- Verify provider order in `availableTranslators` array
- Check `authRequired` flag and token availability
- Verify quota tracking is functioning
- Check network errors in browser console

### Problem: Quota exceeded but still using provider

- Verify localStorage quota tracking is updated
- Check server-side quota tracking endpoint
- Ensure fallback logic is triggered properly
- Verify toast notification appears

### Problem: Language not supported

- Check language code in `TRANSLATOR_LANGS` array
- Verify provider supports the language
- Check language code normalization in `lang.ts`
- Review provider documentation for supported languages

## Dependencies

- **IndexedDB**: Persistent translation cache storage
- **React hooks**: useTranslator, useTextTranslation
- **Zustand**: State management for translation settings
- **Tauri HTTP**: Desktop platform API calls
- **Cloudflare Workers KV**: Server-side caching (optional)

## Performance Considerations

- **Cache preloading**: Load frequently used translations on startup
- **Batch translation**: Translate multiple texts in single API call
- **Auto-pruning**: Prevent cache from growing indefinitely
- **Lazy translation**: Use IntersectionObserver for full-page translation
- **Provider selection**: Use free providers when possible to conserve quota

## Extension Compatibility

**Support for Translation Extensions** (v0.9.36, commit #901):

The system supports integration with browser extensions like LingKuma and Immersive Translate:

- **Iframe Accessibility**: Minimal sandbox restrictions to allow extension injection
- **Message Passing**: Iframe events forwarded to parent for extension communication
- **Foliate-js Updates**: Submodule updated with more accessible iframe attributes

This allows users to use their preferred translation extension alongside Readest's built-in translation system.

## Supported Languages

**50+ languages** including:

- **Major**: English, French, German, Spanish, Italian, Portuguese, Russian
- **Asian**: Chinese (Simplified/Traditional), Japanese, Korean, Thai, Hindi, Bengali
- **European**: Polish, Czech, Hungarian, Romanian, Bulgarian, Croatian, Swedish, Danish, Finnish, Norwegian, Greek
- **Middle Eastern**: Arabic, Farsi, Hebrew, Turkish
- **Other**: Dutch, Indonesian, Vietnamese, Ukrainian, etc.

---

**For implementation details, see commits b83972e3, 4523437e, 93876a73, 68409db8, 38e24da9, 8979827c from the commit history.**
