# Key Components

## Primary Files

- **`src/hooks/useTranslator.ts`** - Main translator hook orchestrating translation logic
- **`src/services/translators/cache.ts`** - Two-tier caching system (memory + IndexedDB)
- **`src/services/translators/providers/`** - Provider-specific implementations
  - `deepl.ts` - DeepL translator (authentication required)
  - `azure.ts` - Azure Translator (free, auto-token)
  - `google.ts` - Google Translate (free)
  - `yandex.ts` - Yandex Translate (free)
- **`src/app/reader/components/annotator/TranslatorPopup.tsx`** - Responsive translation UI

## Related Files

- **`src/services/translators/types.ts`** - Type definitions for translators and providers
- **`src/services/translators/preprocess.ts`** - Text preprocessing before translation
- **`src/services/translators/polish.ts`** - Post-translation text formatting
- **`src/pages/api/deepl/translate.ts`** - Backend translation proxy with KV cache
- **`src/app/reader/hooks/useTextTranslation.ts`** - Full-page text translation hook
- **`src/services/translators/lang.ts`** - Language code normalization
