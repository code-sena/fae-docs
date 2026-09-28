# ADR-017 — Localization/i18n Strategy

- **ID:** ADR-017
- **Date:** 2026-09-12
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

NFR-006 defines the localization requirements:
- Initial language: **Spanish**.
- The system must support language configuration (RF 4.2).
- Future languages may be added without redesigning the interface.
- Date and time formats must be clear for the institutional context.

ADR-001 decided English for all code, documentation, and API contracts. However, the **user-facing interface** must be in Spanish (and potentially other languages in the future). This creates two language layers:
1. **Internal/code:** English (ADR-001).
2. **UI/user-facing:** Spanish (initial), configurable.

The frontend is React Native (ADR-013). The i18n library choice is i18next + react-i18next.

**Known constraints:**
- NFR-006: "Initial language: Spanish."
- NFR-006: "The system must support language configuration, according to SRS RF 4.2."
- NFR-006: "Future languages may be added without redesigning the interface."
- React Native mobile app (ADR-013).
- Backend API responses are in English (ADR-001); UI translates to the user's language.

---

## Decision

**We decided:** Use **i18next + react-i18next** for frontend localization, with Spanish as the default language and a configuration-based language selection mechanism.

### Architecture

```
┌─────────────────────────────────────────────────┐
│ React Native App                                │
│                                                 │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │ UI Layer     │    │ i18next              │   │
│  │ (components) │◄───│  ├─ es/ (Spanish)    │   │
│  │              │    │  ├─ en/ (English)    │   │
│  │  t('key')    │    │  └─ {lang}/          │   │
│  └──────────────┘    └──────────────────────┘   │
│         │                    │                   │
│         ▼                    ▼                   │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │ Date/Number  │    │ Backend API          │   │
│  │ Formatting   │    │ (responses in EN)    │   │
│  │ (Intl API)   │    │                      │   │
│  └──────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Translation file structure

```
src/
├── locales/
│   ├── es/
│   │   ├── common.json        (shared UI strings)
│   │   ├── auth.json          (login, registration)
│   │   ├── attendance.json    (attendance flows)
│   │   ├── justification.json (justification flows)
│   │   └── settings.json      (configuration screens)
│   ├── en/
│   │   ├── common.json
│   │   ├── auth.json
│   │   ├── attendance.json
│   │   ├── justification.json
│   │   └── settings.json
│   └── index.ts               (i18next initialization)
```

### Translation key convention

> Note: values below are Spanish locale data (`es/attendance.json` example), not documentation prose. Per ADR-001, docs stay in English; runtime UI strings are localized.

```json
{
  "attendance": {
    "status": {
      "present": "Presente",
      "absent": "Ausente",
      "late": "Tardanza",
      "justified": "Justificado"
    },
    "scan": {
      "title": "Escanear rostro",
      "camera_permission": "Se requiere permiso de cámara",
      "no_face_detected": "No se detectó rostro. Intente de nuevo.",
      "match_success": "Asistencia registrada exitosamente",
      "match_failed": "No se pudo identificar. Use registro alternativo."
    }
  }
}
```

### Language configuration

- Default language: `es` (Spanish).
- User can change language via settings screen (RF 4.2).
- Language preference is stored in device local storage (AsyncStorage).
- No backend API needed for language selection — purely client-side.
- Backend API responses (error messages, status codes) remain in English; the frontend translates them.

### Date and time formatting

```typescript
// Use Intl API (built into React Native)
const formatDate = (date: Date, locale: string) =>
  new Intl.DateTimeFormat(locale, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(date);

// Spanish: "12 de septiembre de 2026"
// English: "September 12, 2026"
```

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **i18next + react-i18next (chosen)** | Framework-agnostic core; plugin ecosystem; lazy loading of translation files; interpolation and pluralization; well-documented | Slightly heavier than simpler alternatives; configuration can be complex | — (chosen) |
| react-intl (FormatJS) | ICU message format; excellent for dates/numbers; React-specific | More opinionated; less flexible for non-React contexts; smaller plugin ecosystem | i18next is more flexible for React Native |
| next-intl | Server-side i18n; modern API | Tied to Next.js; not applicable for React Native | Frontend is React Native, not Next.js |
| Custom i18n (simple JSON + Context) | Zero dependencies; full control | Must implement pluralization, interpolation, lazy loading manually; reinventing the wheel | i18next provides all this out of the box |

---

## Consequences

**Positive:**
- Spanish UI with English code/API — clean separation of concerns.
- i18next lazy loading keeps initial app bundle small.
- Adding a new language requires only adding a new locale directory and translation files.
- Date/number formatting via Intl API is locale-aware without additional libraries.

**Negative / Trade-offs:**
- Two language layers (English code, Spanish UI) require discipline — translation keys must not leak into code.
- Translation files must be maintained as the UI evolves.
- Backend error messages in English require frontend translation mapping.

**Impact on the system:**
- Affected modules: frontend (React Native app).
- Documents that must be updated:
  - `12-ux-ui/design-system.md` — reference i18next for string management
  - `12-ux-ui/navigation-map.md` — document translated screen names
  - `07-api/contracts/openapi/` — confirm API responses are in English; frontend handles translation
  - `14-training/technical-onboarding.md` — document i18n workflow for new contributors

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Translation keys become inconsistent between languages | Medium | Low | Use TypeScript types for translation keys; CI check that all locale directories have matching keys |
| Missing translations for new features | Medium | Low | i18next fallback chain (user lang → Spanish → English key); `t()` returns key if translation missing |
| Backend error messages not translatable | Low | Medium | Backend returns error codes (not messages); frontend maps codes to translated messages |

---

## References

- NFR-006 Usability/Localization → `04-requirements/non-functional.md`
- RF 4.2 Language configuration → `02-domain/domain-map.md`
- ADR-001 (English for code/docs) → `05-architecture/decisions/records/ADR-001-documentation-language.md`
- ADR-013 (React Native frontend) → `05-architecture/decisions/records/ADR-013-frontend-technology.md`
- i18next documentation: https://www.i18next.com/
