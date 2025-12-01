# Entry Points for Common Tasks

| Task | Primary Files | Steps |
|------|---------------|-------|
| **Add new RTL language** | `src/utils/rtl.ts`, `public/locales/{lang}/` | Update RTL_LANGUAGES set → Add translations → Test |
| **Customize RTL layout** | Component files, `src/utils/rtl.ts` | Import direction utilities → Apply conditional classes → Test |
| **Fix RTL alignment** | Component CSS, `src/styles/globals.css` | Replace fixed properties with logical → Test in RTL mode |
| **Override book direction** | `src/app/reader/components/settings/LayoutPanel.tsx` | Select writing mode → Apply to view settings → Persist |
| **Debug RTL issues** | Browser DevTools, component files | Inspect dir attribute → Check CSS logical properties → Verify flexbox direction |
