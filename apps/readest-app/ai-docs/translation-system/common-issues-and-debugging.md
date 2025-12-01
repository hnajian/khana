# Common Issues and Debugging

## Problem: Translation not cached

- Check if cache is enabled: `cache.isEnabled()`
- Verify IndexedDB is accessible in browser DevTools
- Check cache key format matches: `provider:sourceLang:targetLang:text`
- Ensure cache is not full (check pruning configuration)

## Problem: Provider fallback not working

- Verify provider order in `availableTranslators` array
- Check `authRequired` flag and token availability
- Verify quota tracking is functioning
- Check network errors in browser console

## Problem: Quota exceeded but still using provider

- Verify localStorage quota tracking is updated
- Check server-side quota tracking endpoint
- Ensure fallback logic is triggered properly
- Verify toast notification appears

## Problem: Language not supported

- Check language code in `TRANSLATOR_LANGS` array
- Verify provider supports the language
- Check language code normalization in `lang.ts`
- Review provider documentation for supported languages
