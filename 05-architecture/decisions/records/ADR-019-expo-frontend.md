# ADR-019 — Frontend on Expo (Web + Mobile)

- **ID:** ADR-019
- **Date:** 2026-10-17
- **Status:** Accepted
- **Supersedes:** ADR-013 (Frontend Technology — React Native CLI, "not Expo")
- **Authors:** FaceAttend EDU Team

---

## Context

ADR-013 chose React Native **CLI, explicitly "not Expo"**, to keep control over native modules, plus `react-native-vision-camera` for face capture. The implementation went the other way on every point: both frontends are **Expo SDK 57** apps (`front-end/Web`, `front-end/Mobile`), the camera is `expo-camera`, and distribution for mobile goes through **EAS** (`eas.json`). The web client is not a separate React app as ADR-013 imagined: it is `expo export --platform web` served by nginx in a container (`front-end/Web/Dockerfile`), which also gives the admin/kiosk panel that ADR-013 never contemplated — the final system has two frontends, not one.

**Known constraints:**
- NFR-005: camera for facial recognition must work on Android (primary) and in browsers (admin/kiosk).
- Face capture, fingerprint (DigitalPersona web SDK vendored via `front-end/Web/scripts/copy-digitalpersona.mjs`) and WebSocket biometric attendance must run in the same app.
- Team of 3 with no dedicated frontend specialist — build and signing pipelines must not require local Android/iOS toolchains.
- i18n (ADR-017) and the design system live in both apps.

---

## Decision

**We decided:** Use **Expo (SDK 57) with EAS** for both frontends — React Native for Mobile and Expo Web for the browser client — keeping JavaScript, React Navigation and Context-based state as ADR-013 did.

**Justification:**

1. **One toolchain, two targets:** Expo Web produces the static admin/kiosk SPA (`expo export` → nginx), and the same SDK builds the mobile app — shared patterns, one upgrade path, no separate React web app to maintain.
2. **Camera without native builds:** `expo-camera` covers the facial capture flow (liveness challenge, frame capture) with permission handling per platform, without `react-native-vision-camera`'s native build requirements.
3. **EAS removes the mobile toolchain:** `eas.json` defines development/preview/production profiles; builds and signing happen on EAS, so no teammate needs a local Android SDK.
4. **Already the reality:** both apps run SDK 57 with ~45 viewmodels, MVVM (`ARQUITECTURA_MVVM.md`), jest suites and Docker/Web packaging built on Expo — reverting to RN CLI would discard all of it.

**Implementation:**
- `front-end/Web`: Expo Web export → nginx container (`Dockerfile`), SPA fallback, `/api/*` proxied to Kong, DigitalPersona SDK vendored at build time.
- `front-end/Mobile`: Expo + `expo-camera`, `expo-secure-store` for tokens, i18next (ADR-017), EAS profiles in `eas.json`.
- Both consume Kong at `EXPO_PUBLIC_API_URL` (baked at build time; runtime hostname inference in dev).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Expo + EAS (chosen)** | One SDK for web + mobile; no local native toolchains; camera plugin maintained; OTA updates | Expo couples upgrades to its release cadence; some native modules need a dev client | — (chosen) |
| React Native CLI (ADR-013) | Full native control | Needs local Android/iOS build environments; no web target — a second app would be required; contradicts the implemented code | Never implemented; would discard both working apps |
| React web-only | Simplest web story | Mobile distribution (EAS/store) lost; camera/liveness flows weaker in browsers | The mobile app is a core deliverable (students scan faces) |
| Flutter | Single mobile codebase | Dart; no Expo Web equivalent; team knows JS | Different language, no benefit for the web client |

---

## Consequences

**Positive:**
- One frontend stack for both apps; shared patterns, theming, i18n approach.
- Mobile builds and distribution do not depend on local toolchains (EAS).
- The web admin/kiosk is part of the same stack (Expo Web + nginx), already containerized.

**Negative / Trade-offs:**
- SDK upgrades are coupled to Expo's release train; SDK 57 pins React 19.2 / RN 0.86.
- The DigitalPersona fingerprint SDK is vendored manually (`copy-digitalpersona.mjs`) because it is not an npm dependency — an upgrade step outside Expo's model.
- Source is still JavaScript despite `typescript` being installed (`src/*.js`) — type safety remains pending.

**Impact on the system:**
- Affected services: `front-end/Web`, `front-end/Mobile`.
- Documents that must be updated: ADR-013 (marked Superseded), ADR-017 (i18n note: Mobile uses i18next, Web a custom hook), `12-ux-ui/navigation-map.md`, `10-devops/environments.md` (frontend build/publish).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Expo SDK upgrade breaks native modules | Medium | Medium | Pin SDK; upgrade deliberately between features |
| EAS free-tier limits (build minutes, OTA) | Medium | Low | Preview builds are occasional; production builds are rare |
| Web bundle size grows with face-api.js | Medium | Medium | Lazy-load biometric modules; nginx serves gzip |

---

## References

- Supersedes: ADR-013 → `05-architecture/decisions/records/ADR-013-frontend-technology.md`
- Code: `FULL/front-end/Web/package.json`, `FULL/front-end/Mobile/package.json`, `FULL/front-end/Mobile/eas.json`, `FULL/front-end/Web/Dockerfile`
- Related: ADR-017 (localization), ADR-007 (Kong as API edge)
