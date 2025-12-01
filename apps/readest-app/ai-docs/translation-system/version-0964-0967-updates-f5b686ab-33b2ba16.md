# Version 0.9.64 - 0.9.67 Updates (f5b686ab → 33b2ba16)

## Yandex Translator Integration (v0.9.67, #1652)

**Feature**: Added Yandex Translator as a new translation provider option.

**Overview**: Yandex Translator provides free translation services without authentication requirements, offering an alternative to DeepL, Azure, and Google Translate.

**Implementation**: Already documented in the provider section above.

**Key Features**:
- Free API without authentication
- Multiple service options (yandexgpt, yandextranslate, yandexcloud, yandexbrowser)
- Support for 50+ languages
- API endpoint: `https://translate.toil.cc/v2/translate/`

**File**: `src/services/translators/providers/yandex.ts`

## TOC Translation Fix (v0.9.65, #1610)

**Fix**: Resolved issue where table of contents (TOC) translation was not working properly.

**Problem**: When full book translation was enabled, the TOC items were not being translated, leading to inconsistent language display.

**Solution**: Updated translation hooks to ensure TOC items are translated along with book content.

**Files Updated**:
- `src/app/reader/hooks/useTextTranslation.ts` - TOC translation logic
- `src/app/reader/components/sidebar/TOCView.tsx` - TOC rendering with translations

**Impact**: Complete bilingual reading experience with translated navigation.

## Skip Pre, Code, and Math Tags (v0.9.67, #1698)

**Enhancement**: Improved translation quality by skipping code blocks and mathematical expressions.

**Rationale**:
- Code blocks (`<pre>`, `<code>`) should not be translated as they contain programming syntax
- Mathematical expressions (`<math>`, MathML) should remain in their original form
- Translating these elements can break functionality and readability

**Implementation**: Added tag filtering in translation preprocessing:

```typescript
const SKIP_TAGS = ['pre', 'code', 'math', 'script', 'style'];

function shouldTranslate(element: Element): boolean {
  const tagName = element.tagName.toLowerCase();
  return !SKIP_TAGS.includes(tagName);
}
```

**Files Updated**:
- `src/services/translators/preprocess.ts` - Tag filtering logic
- `src/app/reader/hooks/useTextTranslation.ts` - Element selection for translation

**Benefits**:
- Better translation quality for technical books
- Preserves code syntax and mathematical notation
- Reduces unnecessary API calls and quota usage

---

**Last Updated**: Documentation for commits through 33b2ba16 (November 2025, v0.9.67)
**Related Documents**: [text-to-speech](../text-to-speech/index.md), [annotation-system](../annotation-system/index.md), [settings-system](../settings-system/index.md)
**For implementation details, see commits b83972e3, 4523437e, 93876a73, 68409db8, 38e24da9, 8979827c, 9d3fc078, 219edb9f, b9add62b from the commit history.**
