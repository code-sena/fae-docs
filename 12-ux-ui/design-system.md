# Design System - FaceAttendEDU

The design system of FaceAttendEDU is the shared visual and functional foundation between the Web and Mobile platforms. It is designed to be highly adaptable, supporting light/dark modes and accessibility profiles for different types of color blindness.

---

## Design Tokens

### Colors
The system uses dynamic theme generation (`generateTheme.js`) that adjusts colors based on the mode and the user's vision.

#### Base Palette (Dynamic)
- **Primary (Accent):** Based on the user's selection (Default: `#2196F3` / `hsl(217, 76, 52)`).
- **Neutrals and Backgrounds:**
  - **App Background:** Light: `#F8FAFC` | Dark: `#0F172A`
  - **Surface:** Light: `#FFFFFF` | Dark: `#1E293B`
  - **Elevated:** Light: `#FFFFFF` | Dark: `#334155`
  - **Overlay:** `rgba(0, 0, 0, 0.45)`

#### Semantic Colors
- **Success:** `#10B981` (Light variants generated through `lighten(35)` or `alpha(0.18)`).
- **Warning:** `#F59E0B`
- **Error/Danger:** `#EF4444`
- **Info:** Uses the Primary accent color.

#### Text and Borders
- **Primary Text:** Light: `#0F172A` | Dark: `#F1F5F9` (Adjusted dynamically for contrast).
- **Secondary Text:** Light: `#64748B` | Dark: `#94A3B8`
- **Disabled Text:** Light: `#94A3B8` | Dark: `#475569`
- **Inverse Text:** Light: `#FFFFFF` | Dark: `#0F172A`
- **Primary Borders:** Light: `#E2E8F0` | Dark: `#334155`
- **Secondary Borders:** Light: `#CBD5E1` | Dark: `#475569`

### Typography
It follows a responsive scale defined in `typography.js`.

- **Font Family:** System sans-serif (Android/Web) / System (iOS).
- **Size Scale:**
  - **Display XL:** 52px | Weight: 800 | LH: 62px
  - **Display LG:** 36px | Weight: 800 | LH: 44px
  - **Heading 1:** 28px | Weight: 700 | LH: 36px
  - **Heading 2:** 22px | Weight: 700 | LH: 30px
  - **Body LG:** 16px | Weight: 400 | LH: 26px
  - **Body MD:** 14px | Weight: 400 | LH: 22px
  - **Body SM:** 12px | Weight: 400 | LH: 18px
  - **Button MD:** 15px | Weight: 700
  - **Caption:** 11px | Weight: 400

### Spacing
System based on 8px increments (`theme.config.js`):
- `xs`: 4px | `sm`: 8px | `md`: 16px | `lg`: 24px | `xl`: 32px | `xxl`: 48px | `xxxl`: 64px

### Borders and Shadows
- **Border Radius:** `sm`: 4px | `md`: 8px | `lg`: 12px | `xl`: 16px | `xxl`: 24px | `round`: 9999px.
- **Shadows:**
  - `sm`: Opacity 0.05 | Radius 2 | Elevation 1
  - `md`: Opacity 0.1 | Radius 4 | Elevation 2
  - `lg`: Opacity 0.15 | Radius 8 | Elevation 4
  - `xl`: Opacity 0.2 | Radius 16 | Elevation 8

---

## Components

### Buttons (`Button.js`)
- **Variants:**
  - **Primary:** Primary accent background, `textOnPrimary` text.
  - **Secondary:** Secondary brand color background.
  - **Outline:** Transparent background, primary accent border.
  - **Ghost:** No background or border, primary text.
  - **Danger:** Red error background (`#EF4444`), `errorText` text.
- **Sizes:** `sm` (min-h 36px), `md` (min-h 48px), `lg` (min-h 52px).
- **States:** `loading` (Shows `ActivityIndicator`), `disabled` (Opacity 0.5).

### Forms (`TextInput.js`)
- **Input Types:** Text, email, password, number, phone.
- **Validation Feedback:**
  - **Error:** Red border, red label, and error message below the field. Includes a **"Shake"** animation.
  - **Success:** Green border and `check-circle` icon on the right.
  - **Warning:** Orange border and label.
  - **Progress:** Subtle progress bar in the primary color along the border.

### Feedback and Notifications
- **Toasts:** Supports `success`, `error`, `warning`, and `info` types with automatic fade-out.
- **Alerts:** Contextual alerts with icons and a close button.
- **Modals:** `BaseModal` and `ConfirmModal` for critical actions with style variants (default, danger, warning).
- **Loading:** Implementation of `Loader.js`, `Skeleton.js`, and `ProgressBar.js`.

### Data Tables (`CoursesTable.js`, `UsersTable.js`)
- **Features:** Pagination, sorting, and advanced filtering via `FilterBar`.

---

## UX Patterns

### Principles
- **Critical Confirmation:** Irreversible actions (e.g., delete) must require the `ConfirmModal` in the "danger" variant.
- **Immediate Feedback:** Real-time validation with color changes and visual animations.
- **Empty States:** Use `EmptyState.js` to guide the user when there is no data.

### Error Handling (`apiClient.js`)
- **Network Errors:** Reported as `NetworkError` with user-friendly messages about the gateway.
- **Timeouts:** Managed via `AbortController`.
- **Unauthorized (401):** The global `unauthorizedHandler` clears the local session and redirects to login automatically.
- **API Errors:** `ApiError` wrapper that includes `status`, `code`, and `message`.

---

## Accessibility Guide

- **Contrast:** Use of the `tinycolor2` library in `generateTheme.js` to ensure a minimum contrast ratio of **WCAG AA (4.5:1)**.
- **Vision Modes:** Specific presets for **Deuteranopia, Protanopia, Tritanopia, and Achromatopsia** (`colorPresets.js`).
- **Interactive Elements:** Clear focus indicators (the border changes to `text.primary` when focused).
- **Form Labels:** All inputs have explicit labels for screen reader compatibility.
