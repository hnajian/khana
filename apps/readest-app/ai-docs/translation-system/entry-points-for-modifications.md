# Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add translation provider | `providers/newtranslator.ts` | `useTranslator.ts`, `TranslatorPopup.tsx` |
| Modify caching strategy | `cache.ts` | `useTranslator.ts` |
| Update preprocessing | `preprocess.ts` | `useTranslator.ts` |
| Add polishing rules | `polish.ts` | `useTranslator.ts` |
| Customize UI | `TranslatorPopup.tsx` | `lang.ts` |
| Backend proxy changes | `pages/api/deepl/translate.ts` | Provider files |
| Quota management | `useTranslator.ts` | `pages/api/deepl/translate.ts` |
