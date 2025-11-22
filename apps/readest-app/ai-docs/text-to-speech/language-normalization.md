# Language Code Normalization

**Added**: v0.9.18 (Commit #457)

## Overview

Language Code Normalization improves TTS voice selection by normalizing language codes across different metadata formats. This ensures that books with varied language metadata formats can automatically match the correct TTS voice.

## Problem Solved

Books may have language metadata in various formats:
- ISO 639-1: `en`, `zh`, `ja`
- ISO 639-2: `eng`, `chi`, `jpn`
- BCP 47: `en-US`, `zh-CN`, `ja-JP`
- Regional variants: `en-GB`, `pt-BR`, `zh-TW`

TTS voices use BCP 47 codes, leading to mismatches when book metadata uses different formats.

## Normalization Logic

**Location**: `src/utils/language.ts`

```typescript
function normalizeLangCode(code: string): string {
  if (!code) return 'en-US';

  // Convert ISO 639-2 to ISO 639-1
  const iso639_2to1 = {
    'eng': 'en',
    'chi': 'zh',
    'jpn': 'ja',
    'fra': 'fr',
    'deu': 'de',
    'spa': 'es',
    // ... full mapping
  };

  const normalized = iso639_2to1[code] || code;

  // Ensure BCP 47 format
  if (normalized.length === 2) {
    const regionMap = {
      'en': 'en-US',
      'zh': 'zh-CN',
      'ja': 'ja-JP',
      'fr': 'fr-FR',
      'de': 'de-DE',
      'es': 'es-ES',
      // ... default regions
    };
    return regionMap[normalized] || `${normalized}-${normalized.toUpperCase()}`;
  }

  return normalized;
}
```

## Voice Matching Algorithm

1. Book language code normalized to BCP 47 format
2. Filter TTS voices by normalized code
3. Fallback to base language (e.g., `en` for `en-GB`)
4. Final fallback to default voice (`en-US`)

**Example Flow**:
```typescript
// Book has metadata: language="eng" (ISO 639-2)
normalizeLangCode("eng") → "en-US"
// Finds Edge TTS voice: "en-US-AvaNeural"

// Book has metadata: language="zh" (ISO 639-1)
normalizeLangCode("zh") → "zh-CN"
// Finds Edge TTS voice: "zh-CN-XiaoxiaoNeural"

// Book has metadata: language="pt-BR" (BCP 47 regional)
normalizeLangCode("pt-BR") → "pt-BR"
// Finds Edge TTS voice: "pt-BR-FranciscaNeural"
```

## Impact

- Better automatic voice selection for books with varied metadata
- Supports more book metadata formats
- Consistent voice matching across platforms
- Reduced need for manual voice selection

## AI Modification Guidelines

**To add a new language code mapping**:

1. Add ISO 639-2 to ISO 639-1 mapping in `iso639_2to1`:
   ```typescript
   'rus': 'ru',  // Russian
   'ara': 'ar',  // Arabic
   ```

2. Add default region mapping in `regionMap`:
   ```typescript
   'ru': 'ru-RU',
   'ar': 'ar-SA',
   ```

3. Test with books that have the language code in metadata

---

**Related**: [index.md](./index.md) (Main TTS feature documentation)
