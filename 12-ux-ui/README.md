# 12 — UX/UI

> **What is this?** The user experience design: how the system looks, how it is navigated,
> and how it behaves from the end user's perspective.
>
> **FaceAttend EDU** — every statement below was validated against the actual front-end code
> on branch `develop`: `front-end/Web/` (React Native Web + React Navigation) and
> `front-end/Mobile/` (Expo). Two platforms, one shared design language.

## Why design comes before code

Changing a wireframe takes 5 minutes. Changing the code takes hours.
Changing the code in production with real users can cost days and reputation.

**Design first → implement later.**

In this project the order was inverted at times (code existed before documentation), so this
section was built **backwards from the shipped code** — which makes it verifiable: every token,
route and component listed here exists in a file you can open.

---

## Files in this section

| File | Content | Source of truth in code |
|------|---------|-------------------------|
| [`navigation-map.md`](./navigation-map.md) ⭐ | All screens, how they connect, role access matrix | `Web/src/navegation/appNavigator.js`, `AuthenticatedNavigator.js`, `linking.ts`, `Mobile/src/navigations/AppNavigator.js`, `Web/src/viewmodels/useRolePermissions.js` |
| [`wireframes.md`](./wireframes.md) | Low-fidelity ASCII structure of the main screens | `Web/src/view/...`, `Mobile/src/view/screens/...` |
| [`design-system.md`](./design-system.md) | Tokens (colors, spacing, radius, typography), component library, interaction patterns | `Web/src/core/config/theme.config.js`, `core/theme/presets.js`, `core/theme/generateTheme.js`, `context/ThemeContext.js`, `view/components/common/` |

---

## Summary: navigation map (`navigation-map.md`)

### Public area (no authentication)

**Web** (`appNavigator.js` + `linking.ts` deep links)
- `/` — Landing (`LandingView`: hero, stats, marketing)
  - `/login` — Login (`LoginView`: email/password, password-strength indicator, policy modal)
    - → `/dashboard` on success (JWT issued via Kong gateway)
  - `/register` — Signup (`SignupView`)
  - ❗ Gap: no `/recover-password` route on Web yet (the flow exists on Mobile and in 01-ms-identity)

**Mobile** (unauthenticated branch of the root stack)
- `HomesScreen` (login)
  - `ForgotPasswordScreen` → `VerifyCodeScreen` (recovery-code flow, matches `ResetPasswordUseCase`)
  - → `DashboardScreen` on success

### Private area — Role: admin
- **Dashboard** — KPI grid (`MetricsSummary`, `DailyBarChart`, `WeeklyTrend`, `AtRiskStudentsList`)
- **Users** — tab labeled *"Usuarios"*: Admin/Teacher/Student sub-sections, `UsersTable`, CSV import
- **Courses** — `RegisterCourseModal`, `ImportCoursesModal`, `CourseDetailModal`
- **Environments** — `EnvironmentCard` grid, `EnvironmentFormModal`, `ScheduleModal` — **admin-only tab**
- **Reports** — filters + charts + export, **all scope** (`canViewAllReports` is admin-only)
- **Settings** — `AdminSettings`: general / appearance / facial / notifications / security
- **Profile** — sidebar footer → logout

### Private area — Role: teacher
- Dashboard ✅ · Users relabeled **"Alumnos"** (view + register face only, no create/import)
- Courses ✅ manage · Environments ❌ hidden · Reports ✅ own scope · `TeacherSettings`

### Private area — Role: student
- Dashboard ✅ · Users relabeled **"Compañeros"** (read-only)
- Courses ✅ view only · Environments ❌ · Reports ❌ · `StudentSettings` (personalization only)

### Mobile private flows (beyond dashboard/menu)
- `Novedades` (news), `Profile`, `UpdatePhoto`, `RegisterFace`
- Facial-fail path: `FacialFail` → retry / manual justification
- Justifications: `MenuJustify` → `AddJustification` / `ConsultJustify` / `ValidJustifications` /
  `PendingJustificationScreen` (teacher validates)
- Attendance: `DisplayingAttendance`, `AttendanceReportScreen`
- Admin-only: `ManageUsersScreen`, `ManageEnviromentScreen`, `SchoolConfigurationScreen`

### Access matrix (from `useRolePermissions.getVisibleTabs()`)

| Screen / Tab | Guest | Student | Teacher | Admin |
|--------------|-------|---------|---------|-------|
| / (Landing) | ✅ | ✅ | ✅ | ✅ |
| /login, /register | ✅ | — | — | — |
| Dashboard | ❌ | ✅ | ✅ | ✅ |
| Users ("Compañeros"/"Alumnos"/"Usuarios") | ❌ | ✅ read-only | ✅ view+face | ✅ full CRUD + import |
| Courses | ❌ | ✅ view | ✅ manage | ✅ manage + import |
| Environments | ❌ | ❌ | ❌ | ✅ |
| Reports | ❌ | ❌ | ✅ own | ✅ all + notify-all |
| Settings | ❌ | ✅ personal | ✅ partial | ✅ full |

**Security rule found in code:** unknown role → `visibleTabs = []` (*"Vacío a propósito, no queremos fugas aunque alguien logre pasar las validaciones"*). UI gating is complemented server-side by Kong JWT + 02-ms-authorization checks (defense in depth).

**Architecture pattern enforced in code:** Navigator → Screen → View (routes register Screens only; Views never navigate). Web keeps a persistent `CollapsibleSidebar` outside the inner stack so state survives navigation; Mobile swaps public/private branches based on `isAuthenticated` (AsyncStorage token + `userRole`).

---

## Summary: wireframes (`wireframes.md`)

Low-fidelity ASCII sketches derived from the actual View code — structure only; colors and typography are governed by `design-system.md`. Included screens:
- **Web Login** — animated split layout (brand panel ⇄ form panel), password-strength bar, `PasswordPolicyModal` on weak-password attempt.
- **Web Authenticated shell** — persistent collapsible sidebar (Dashboard · Usuarios · Cursos · Ambientes · Reportes · Ajustes · user+logout footer) + `PageHeader` + active-screen content.
- **Web Dashboard** — row of `StatCard`s over `MetricsSummary`, then two-column chart grid (`DailyBarChart` / `WeeklyTrend`), then risk lists (`AtRiskStudentsList`, `PerfectAttendanceList`).
- **Web Users** — `UsersFilterBar` + `UsersTable` (paginated) + domain modals.
- **Web Environments** — `EnvironmentCard` grid + `EnvironmentFormModal` / `ScheduleModal`.
- **Web Settings** — one screen, six modal sections rendered per role.
- **Mobile key flows** — login, recovery code, dashboard, menu, register-face camera flow, facial-fail → manual justification.

---

## Summary: design system (`design-system.md`)

### Design principles found in code
1. **Single source of truth:** *"Nunca importar Colors directamente — siempre usar ThemeTokens"*; all color access goes through `useTheme()` → `theme.colors`.
2. **Accessibility-first:** semantic color slots are re-mapped per **color-vision deficiency mode** (base / deuteranopia / protanopia / tritanopia / achromatopsia), not just light/dark.
3. **Dynamic theming:** `generateTheme(accent, mode)` adapts the user-picked accent to the background for contrast (`adaptColorForTheme`); original kept as `primaryBase`.
4. **Responsive scaling:** font sizes scale via `useResponsive`; collapsible-sidebar layout.

### Colors — semantic slots × vision modes (`presets.js`)

| Slot | Meaning | base | deuteranopia | protanopia | tritanopia | achromatopsia |
|------|---------|------|--------------|------------|------------|----------------|
| primary | App color, main buttons | `#1983FC` | `#286FE2` | `#286FE2` | `#7048D8` | `#5A5A5A` |
| success | Confirmations, 2xx | `#19C687` | `#00A6A6` | `#007F8B` | `#1B9E77` | `#808080` |
| warning | Notices, intermediate | `#FFAB00` | `#E6A700` | `#F0A000` | `#E76F00` | `#B0B0B0` |
| error | Failures, critical | `#F04438` | `#D81B60` | `#7B2CBF` | `#D73027` | `#303030` |
| text | Typography control | `#000000` | `#000000` | `#000000` | `#000000` | `#000000` |

`DEFAULT_ACCENT = "#286FE2"`, `DEFAULT_MODE = "light"`. Generated theme structure (`generateTheme.js`): `brand{primary, primaryBase, primaryLight, primaryDark}`, `background{app, surface, elevated, overlay}`, `text{primary, secondary, disabled, inverse, onBrand}`, `border{primary, secondary, focus, error}`, `status{success/warning/danger/info + Light variants}`, `interactive{hover, pressed, disabled}`.

### Spacing (`theme.config.js` DESIGN_TOKENS.spacing)

| Token | Value | Use |
|-------|-------|-----|
| xs | 4 | Icon padding |
| sm | 8 | Compact gaps |
| md | 16 | Default element gap |
| lg | 24 | Section spacing |
| xl | 32 | Card padding |
| xxl | 48 | Page gutters |
| xxxl | 64 | Hero sections |

Radius (`borderRadius`): none 0 · sm 4 · md 8 · lg 12 · xl 16 · xxl 24 · round 9999

### Typography (`fontSize` / `fontWeight`)

| Level | Size | Weight | Use |
|-------|------|--------|-----|
| xs | 11 | 400–500 | Captions, badges |
| sm | 12 | 400 | Secondary labels |
| md | 14 | 400 | Body default |
| lg | 16 | 400/500 | Emphasized body |
| xl | 18 | 500–600 | Subsection titles |
| xxl | 20 | 600 | Card titles |
| xxxl | 24 | 600–700 | Page titles |
| display | 32 | 700–800 | Landing hero |

System font stack (no custom webfont loaded); weights normal 400 → extrabold 800; sizes scale with `useResponsive`.

### Component library (`view/components/common/`)

| Family | Components | Notes |
|--------|-----------|-------|
| buttons | `Button`, `IconButton` | Variants driven by `theme.colors.interactive` |
| inputs | `TextInput`, `PasswordInput`, `TextArea`, `Select`, `Autocomplete`, `Checkbox`, `RadioGroup`, `Switch`, `Slider`, `DatePicker`, `TimePicker`, `FileUpload`, `ColorPicker` | `ColorPicker` feeds accent customization; `FileUpload` powers CSV import |
| layout | `CollapsibleSidebar` (+ `SidebarNav/Header/Footer/Item/Divider`), `FormRow` | Persistent-sidebar pattern |
| lists | `List`, `UsersTable`, `CoursesTable`, `UsersFilterBar`, `LoadingFooter` | Pagination mirrors API `limit/offset` + `x-total-count` |
| filters | `UserFilters`, `ColumnFilterDropdown`, `ContextualFilterDropdown`, `AttendanceFilterInput` | |
| modals | `BaseModal`, `ConfirmModal` + domain modals (RegisterCourse, ImportCourses, CourseDetail, EnvironmentForm, Schedule, FaceRegistration, ImportStudents, StudentDetail) | |
| feedback | `Alert`, `Toast`, `Loader`, `Skeleton`, `EmptyState`, `ProgressBar`, `Tooltip`, `PushNotification` | Push component consumes 08-ms-notification |
| badges | `Badge`, `StatusBadge`, `FloatingBadge` | Colors from `theme.colors.status` |
| cards | `Card`, `StatCard`, `StatsCard` | Dashboard KPIs |
| charts | `BarChart`, `DailyBarChart`, `WeeklyTrend` | Data palette in `core/constants/dataColors.js` |
| navigation | `Navbar`, `Tabs`, `Accordion`, `Drawer`, `PageHeader`, `ThemeToggle`, `LanguageSelector` | |
| animation | `AnimatedDropdown`, `AuthAnimatedLayout` | Login/signup transitions |

**Domain feature components:** `dashboard/` (MetricsSummary, AtRiskStudentsList, PerfectAttendanceList, TopPerformingGroups, InstructorAttendanceList, WeeklyAttendanceView), `hero/` (Landing), `auth/` (`PasswordStrengthIndicator`, `PasswordPolicyModal`), `students/`, `environments/`.

### Interaction patterns
- **Roles shape language, not just visibility:** same tab labeled *Usuarios / Alumnos / Compañeros* per role (`TAB_LABELS_BY_ROLE`) — context-aware copy is an official pattern.
- **Settings IA:** one screen, six modal sections (general, appearance, facial [threshold sliders], notifications, security [2FA, session timeout], personalization), rendered per role.
- **Facial-recognition failure path (Mobile):** capture fail → `FacialFailScreen` → manual justification.
- **Theming persistence:** `themeStorage.js` (localStorage/AsyncStorage) + `ThemeContext`.
- **i18n:** own pipeline (`core/utils/i18n`) with `en/fr/pt/de.json`; Mobile has parallel `LanguageService` + `LanguageSelector`.

---

## Known gaps (honest findings from the code review)

1. Web has no `/recover-password` deep link although the backend flow (reset password + recovery code) exists and Mobile implements it.
2. Spanish locale files are not present in the Web translations folder — ES strings are hardcoded in several components (e.g., sidebar labels). Needs decision.
3. Mobile screen names mix English/Spanish routes (`Novedades`, `MenuJustify`) — naming convention should be documented.
4. No Figma/Balsamiq artifacts committed — wireframes live as ASCII in `wireframes.md`.
5. Tokens live in JS, not CSS variables — document mapping if a Tailwind/web-token export is added.
6. No formal WCAG contrast audit despite the strong color-blind palette system — worth linking to ISO 25010 usability findings.

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `04-requirements/user-stories.md` → what flows exist | Screens implementing each HU (Dashboard, Users, Courses, Environments, Reports, Settings) |
| `02-domain/entities-and-rules.md` → what data to display | Fields in wireframes (School, Course, Session, AttendanceRecord, BiometricTemplate) |
| `09-microservices/` → what APIs the frontend consumes | Data arriving at each screen via Kong gateway (:8000, JWT + rate-limit 100/min): 01-identity (auth), 03-academic (schools/courses), 04-scheduling (sessions), 05-attendance (records/reports), 06-biometric (enrollment/capture), 07-configuration (settings/appearance) |
| `11-quality/` (ISO 25010 usability report) | Accessibility-first design decisions (vision-mode palettes, contrast adaptation) |

---

## Questions this section must answer

- **How many screens does the system have?** Web: 3 public (Landing, Login, Register) + 6 private areas (Dashboard, Users, Courses, Environments, Reports, Settings) with role-dependent tabs. Mobile: login + recovery-code flow + dashboard + ~15 menu-driven screens (Novedades, Profile, UpdatePhoto, RegisterFace, FacialFail, justification suite, attendance views, admin management). Exact routes in `navigation-map.md`.
- **How does each type of user navigate?** Through the access matrix driven by `useRolePermissions.getVisibleTabs()`: admin sees everything; teacher sees *"Alumnos"* instead of *"Usuarios"*, manages courses, loses Environments and global reports; student gets a reduced read-only experience (*"Compañeros"*, view-only courses, personal settings only). Unknown roles get zero tabs by design.
- **What visual components are repeated?** The common library under `view/components/common/`: `Button`/`IconButton`, paginated tables (`UsersTable`, `CoursesTable`), `BaseModal` + 8 domain modals, `CollapsibleSidebar`, `StatCard`/`Card`, status badges, charts, and auth animation layout — fully catalogued in `design-system.md`.
- **What is the system's visual language?** Token-driven and accessibility-first: semantic color slots (primary/success/warning/error/text) re-mapped per color-vision deficiency mode, dynamic accent adaptation for contrast, 7-step spacing scale, 8-level responsive typography on the system font stack, and context-aware copy that changes per role.