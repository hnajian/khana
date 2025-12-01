# AI Agent Modification Guidelines

## Adding a New Translation Provider

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

## Implementing Text Preprocessing Rules

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

## Adding Language-Specific Polishing

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

## Extending Cache Configuration

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

## Improving Quota Management

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
