# ADR-028 — Mobile client: MVVM with hooks, SecureStore session, Expo push notifications

- **ID:** ADR-028
- **Date:** 2026-10-26
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-019 (Expo), ADR-021 (notification service), ADR-023 (opaque sessions)
- **Implementation note (verified 2026-10-26):** `FULL/front-end/Mobile/src/{view,viewmodels,services,storage}`, `08-ms-notification/src/infrastructure/push/expoPush.ts`.

---

## Context

ADR-019 chose Expo but no ADR records how the app is layered, where the session lives, or how learners learn of attendance events.

---

## Decision

**We decided:**
1. **MVVM:** `view/screens/*` hold only JSX and visual state; each screen calls a hook in `viewmodels/use<Screen>ViewModel.js` (state, actions such as capture/submit, loading/error); `services/*Service.js` perform HTTP via the shared API client. The codebase is JavaScript (JS), not TypeScript, and ViewModels are React hooks rather than classes — an `AttendanceViewModel.ts` class does not exist; the equivalent is `useDisplayingAttendanceViewModel.js` + `services/AttendanceService.js`.
2. **Session storage:** the session token is kept in `expo-secure-store` (Keychain/Keystore) on native; web falls back to AsyncStorage (`storage/TokenStorage.js`). Legacy AsyncStorage tokens migrate on read.
3. **Push:** `PushService.js` registers the Expo push token; `08-ms-notification` sends via Expo Push with localised title/body (e.g. “Low attendance”).
4. **No mock data in screens:** services call the real API; mocks exist only under `__tests__`.

---

## Alternatives

| Alternative | Reason for discarding |
|---|---|
| Redux/MobX global store | Excess for team size |
| Logic inside screens | Untestable, 800-line components |
| AsyncStorage for token | Plaintext on device (audit S-11) |

---

## Consequences

- Viewmodels and services are unit-testable (`__tests__`).
- **Open items before closing:** `usesCleartextTraffic: true` in `app.json`, `eas.json` pointing to `http://localhost:8000`, hard-coded LAN IP in `src/config/env.js` (audit S-12, S-13). The push content for “attendance marked” and “late after tolerance” must be confirmed in `attendanceAlerts`; today the verified templates are alert-type messages (low attendance etc.).
- Expo Push routes through Expo/Google/Apple servers: international transfer (audit L-06).
