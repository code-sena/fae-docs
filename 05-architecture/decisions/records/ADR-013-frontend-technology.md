# ADR-013 — Frontend Technology

- **ID:** ADR-013
- **Date:** 2026-09-12
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

The system requires a client application for end users (students, instructors, administrators). NFR-005 defines compatibility with multiple browsers and devices. NFR-006 requires the interface to be intuitive, responsive, and accessible. The `12-ux-ui/` directory contains a design system and navigation map but no technology decision.

The frontend must:
1. Run on Android mobile devices (primary use case: students scanning faces for attendance).
2. Support responsive design for web browsers (NFR-005).
3. Integrate with the camera for facial recognition (NFR-005: "camera permission denial gracefully").
4. Support i18n/localization (NFR-006: Spanish initial, future languages).
5. Communicate with the backend via Kong API Gateway (ADR-007).

**Known constraints:**
- NFR-005 lists Android as the primary mobile OS (latest supported version).
- The system is for a single educational institution — not a consumer app.
- Team of 3 with no dedicated frontend specialist.
- Camera access is required for facial recognition attendance.

---

## Decision

**We decided:** Use **React Native** with **JavaScript** as the frontend framework, deployed as a mobile application for Android.

**Justification:**

1. **Camera access:** React Native provides native camera access via `react-native-camera` or `expo-camera`, which is required for facial recognition attendance (NFR-005).
2. **Android coverage:** React Native compiles to native Android (and iOS) applications, covering the primary mobile requirement.
3. **Responsive design:** React Native's Flexbox-based layout system naturally supports different screen sizes.
4. **Shared logic:** If a web application is needed later, React Native Web allows sharing code between mobile and web.
5. **Ecosystem:** React Native has the largest ecosystem of UI component libraries, camera libraries, and community support for mobile development.
6. **JavaScript consistency:** JavaScript is consistent with the TypeScript backend (TypeScript is a superset of JavaScript).

**Implementation:**
- React Native CLI (not Expo) for maximum control over native modules.
- Navigation: React Navigation (de facto standard for React Native).
- State management: React Context + useReducer (lightweight for this scope).
- HTTP client: Axios or fetch API for REST calls to Kong API Gateway.
- Camera: `react-native-vision-camera` for facial recognition capture.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **React Native + JavaScript (chosen)** | Native camera access; large ecosystem; responsive; JS consistency with backend; Android + iOS | Larger app size than pure native; JavaScript (not TypeScript) reduces type safety | — (chosen) |
| Flutter | Beautiful UI; single codebase for Android/iOS; excellent camera plugin | Dart language (team must learn); smaller ecosystem than React Native; different from backend JS | Team knows JavaScript; Dart is an additional language to learn |
| React (web only) | Simpler than React Native; responsive design via CSS; browser-based | No native camera access (browser camera API is limited); requires PWA for offline; Android browser limitations for facial recognition | Camera API limitations make web-based facial recognition unreliable |
| Angular + Ionic | WebView-based; HTML/CSS UI; TypeScript | Hybrid app (not native); camera access is less reliable; performance issues for real-time video processing | WebView limitations for camera-intensive use cases |
| Native Android (Kotlin/Java) | Best camera performance; native Android APIs | Requires learning Kotlin/Java; no iOS support; different language from backend; higher development cost | Single-platform; different language; higher learning curve |

---

## Consequences

**Positive:**
- Native camera access for facial recognition attendance.
- Large ecosystem of React Native libraries and community support.
- If iOS support is needed later, the same codebase compiles for iOS.
- JavaScript consistency with the backend reduces context-switching for the team.

**Negative / Trade-offs:**
- JavaScript (not TypeScript) reduces compile-time type safety. Mitigated by PropTypes or JSDoc annotations.
- React Native app size is larger than pure native Android apps.
- React Native debugging tools are less mature than browser DevTools.

**Impact on the system:**
- Affected modules: all (frontend consumes all backend APIs).
- Documents that must be updated:
  - `12-ux-ui/navigation-map.md` — reference React Navigation structure
  - `12-ux-ui/design-system.md` — reference React Native UI component library (e.g., React Native Paper)
  - `07-api/contracts/openapi/` — confirm API contracts are mobile-consumable
  - `10-devops/local-setup.md` — document React Native development setup (Android SDK, emulator)

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Camera integration issues across Android devices | Medium | High | Test on multiple Android devices; use `react-native-vision-camera` which handles device differences; provide alternative registration flow per NFR-005 |
| JavaScript type-safety issues | Medium | Low | Use PropTypes or convert to TypeScript later; code review catches type errors |
| React Native version upgrade breaks native modules | Medium | Low | Pin React Native version; test upgrades in dev before staging |

---

## References

- NFR-005 Compatibility → `04-requirements/non-functional.md`
- NFR-006 Usability → `04-requirements/non-functional.md`
- Design system → `12-ux-ui/design-system.md`
- Navigation map → `12-ux-ui/navigation-map.md`
- API contracts → `07-api/contracts/openapi/`
