# Architecture

## Translation Flow

1. **Text Selection**: User selects text in the reader
2. **Preprocessing**: Optional text substitution for better context (e.g., "Cover" → "The Cover")
3. **Cache Check**: Query memory cache → IndexedDB cache
4. **Translation**: If cache miss, call selected provider
5. **Caching**: Store translation in both memory and IndexedDB
6. **Polishing**: Apply language-specific formatting rules
7. **Display**: Show translation in responsive popup

## Supported Translation Providers

| Provider | Authentication | Features | API Version |
|----------|----------------|----------|-------------|
| **DeepL** | Required (authRequired: true) | Quota tracking, daily usage limits | v2 (with v1 compatibility) |
| **Azure Translator** | Not required (auto-token) | 8-minute token expiry, auto-refresh | v3.0 |
| **Google Translate** | Not required | Platform-aware (Tauri HTTP for desktop) | N/A |
| **Yandex Translate** | Not required | Multiple service options (yandexgpt, etc.) | v2 |

### Provider Details

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

## Two-Tier Caching Strategy

**Cache Implementation** (`src/services/translators/cache.ts`, 594 lines):

### Level 1: Memory Cache
- **Purpose**: Fast lookups for frequently translated texts
- **Storage**: In-memory JavaScript object
- **Features**:
  - Timestamps tracked for cache expiry
  - Instant access with zero latency
  - Cleared on app restart

### Level 2: IndexedDB Cache
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

## API Version Handling

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

## Quota Management

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
