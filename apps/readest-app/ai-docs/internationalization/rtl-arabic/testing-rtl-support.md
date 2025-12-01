# Testing RTL Support

## Manual Testing Checklist

- [ ] Switch UI language to Arabic (`?lng=ar`)
- [ ] Verify UI mirrors completely
- [ ] Open an Arabic EPUB book
- [ ] Confirm book content displays RTL
- [ ] Check progress bar fills right-to-left
- [ ] Verify sidebar appears on the right
- [ ] Test navigation buttons are swapped
- [ ] Check annotations align to the right
- [ ] Verify TOC indentation reverses
- [ ] Test search results align properly
- [ ] Confirm book cards align to the right in library
- [ ] Verify modal dialogs mirror correctly

## Automated Tests

```typescript
// test/rtl.test.ts
import { getDirFromLanguage, getDirFromUILanguage } from '@/utils/rtl';

describe('RTL Detection', () => {
  it('detects Arabic as RTL', () => {
    expect(getDirFromLanguage('ar')).toBe('rtl');
    expect(getDirFromLanguage('ar-SA')).toBe('rtl');
  });

  it('detects English as LTR', () => {
    expect(getDirFromLanguage('en')).toBe('ltr');
  });

  it('handles unknown languages', () => {
    expect(getDirFromLanguage('')).toBe('auto');
    expect(getDirFromLanguage('xyz')).toBe('ltr');
  });
});
```
