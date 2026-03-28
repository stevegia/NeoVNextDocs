# Session Changelog: Security Hardening & UI Polish (2026-03-26)

> Summary of changes from the March 2026 session covering GCP security hardening,
> desktop chrome improvements, and frontend UI polish.

---

## 1. GCP Security Hardening

### Dedicated Service Accounts (least-privilege IAM)

Replaced the default Compute Engine service account (which had broad `Editor` role) with
two purpose-built service accounts:

| Service Account | Purpose | IAM Roles |
|-----------------|---------|-----------|
| `neovlab-api-invoker@<PROJECT>.iam.gserviceaccount.com` | API backend calls Cloud Run GPU workers | `roles/run.invoker` |
| `neovlab-gpu-worker@<PROJECT>.iam.gserviceaccount.com` | GPU worker runtime identity | `roles/run.developer`, `roles/cloudsql.client`, `roles/storage.objectUser` |

**Files changed:**
- [deploy-visual-tagging.sh](docker/deploy-visual-tagging.sh) -- `--service-account` flag added
- [GoogleCloudGpuPlatformAdapter.cs](src/backend/NeoVLab.Api/Services/GpuPlatform/GoogleCloudGpuPlatformAdapter.cs) -- OIDC fallback for user-account auth
- [launchSettings.json](src/backend/NeoVLab.Api/Properties/launchSettings.json) -- `GOOGLE_APPLICATION_CREDENTIALS` env var for CloudDev profile

### Cloud SQL SSL Enforcement

- Enabled `requireSsl` on the Cloud SQL instance
- Updated connection string from `SslMode=Disable` to `SslMode=Require;Trust Server Certificate=true`

**Files changed:**
- [appsettings.CloudDev.json](src/backend/NeoVLab.Api/appsettings.CloudDev.json)

### OIDC Token Generation Fallback

The `GoogleCloudGpuPlatformAdapter` now handles both service-account and user-account
authentication flows:

1. Try Application Default Credentials (ADC) -- works for service accounts
2. Try `gcloud auth print-identity-token --audiences=<URL>` -- works for service accounts via CLI
3. Fallback: `gcloud auth print-identity-token` (no `--audiences`) -- works for user accounts

This fixes the 403 error when running locally with `gcloud auth login` (user credentials).

### Secret Key Protection

- Added `*-key.json` to `.gitignore` to prevent accidental commit of GCP service account keys

**Files changed:**
- [.gitignore](.gitignore)

---

## 2. WPF Desktop Chrome

### Glowing Border Halo
- `BorderThickness=1.5`, `BlurRadius=32`, `Opacity=0.45`, `Margin=10`
- Border color syncs with React theme accent via WebView2 bridge IPC

### Brand Title in Title Bar
- "Neo V nExT" rendered in WPF title bar with custom "Neo Sci-Fi" font
- `CharacterSpacing=150` + `DropShadowEffect` glow
- React `TopBar` hides logo when running inside WebView2

### Accent Color Sync Fix
- Fixed `JSON.stringify` double-encoding bug in `webview.postMessage()`
- Theme colors now update `Window.Resources` (not `Application.Current.Resources`)

### WebView2 Rounded Corners Fix
- `DefaultBackgroundColor = Transparent` on WebView2 control
- CSS `border-radius: 8px; overflow: hidden` on `html`, `body`, `#root`, `.aero-bg`
- Changed `.aero-bg::after` from `position: fixed` to `position: absolute`

### Custom Resize Corner
- Win32 P/Invoke (`WM_SYSCOMMAND + SC_SIZE + HTBOTTOMRIGHT`) for custom resize grip
- Diagonal line glyph with glow effect in bottom-right corner

**Files changed:**
- [MainWindow.xaml](src/desktop/NeoVLab.Desktop/MainWindow.xaml)
- [MainWindow.xaml.cs](src/desktop/NeoVLab.Desktop/MainWindow.xaml.cs)
- [TopBar.tsx](src/frontend/src/components/layout/TopBar.tsx)

---

## 3. Frontend UI Polish

### Glass Surface System
- New `[data-glass-surface]` CSS attribute selector with 5 variants: `none`, `clear`, `warm`, `cool`, `accent`
- Settings UI with visual picker cards
- Preference persisted and synced to `document.documentElement.dataset`

**Files changed:**
- [aero-theme.css](src/frontend/src/styles/aero-theme.css)
- [PreferencesContext.tsx](src/frontend/src/contexts/PreferencesContext.tsx)
- [usePreferences.ts](src/frontend/src/hooks/usePreferences.ts)
- [SettingsAppearanceSection.tsx](src/frontend/src/components/SettingsAppearanceSection.tsx)

### Custom Wallpaper Picker
- Desktop-only "Browse" button triggers WPF file picker via bridge IPC
- Data URL stored in `localStorage`, rendered by `BackgroundManager`
- "Scene Library" renamed to "Wallpaper Library"

**Files changed:**
- [SettingsAppearanceSection.tsx](src/frontend/src/components/SettingsAppearanceSection.tsx)
- [BackgroundManager.tsx](src/frontend/src/components/layout/BackgroundManager.tsx)

### Video Player & Media Detail
- Replaced Unicode/emoji icons with lucide-react: Play, Pause, VolumeX, Volume1, Volume2, Maximize, Minimize2
- Video container changed to `flex-1 h-full object-contain` to match sidebar height

**Files changed:**
- [PlayerControls.tsx](src/frontend/src/components/media/PlayerControls.tsx)
- [VideoPlayer.tsx](src/frontend/src/components/media/VideoPlayer.tsx)
- [MediaDetailPage.tsx](src/frontend/src/pages/MediaDetailPage.tsx)

### Ingest Page
- Fixed drive label double-colon ("C: :") -- removed extra `:` from JSX
- Added `font-mono` to drive letters, split free/total space into secondary/muted typography
- Fixed page container and glass-panel surface, H1 size, picker row alignment

**Files changed:**
- [DriveList.tsx](src/frontend/src/components/ingest/DriveList.tsx)
- [DropZone.tsx](src/frontend/src/components/ingest/DropZone.tsx)
- [ScanDirectoryList.tsx](src/frontend/src/components/ingest/ScanDirectoryList.tsx)
- [IngestPage.tsx](src/frontend/src/pages/IngestPage.tsx)

### Artifact Browser Page
- Fixed 8 design philosophy violations: opacity values, border colors, Unicode chevrons,
  hover states, table headers, file path `font-mono`, etc.

**Files changed:**
- [ArtifactBrowserPage.tsx](src/frontend/src/pages/ArtifactBrowserPage.tsx)

### Smoke Tests & Jobs Page
- Inlined SD and WAN smoke test components via lazy `Suspense`
- Removed duplicate `GpuBadge`, global status filter chips, generation progress sections

**Files changed:**
- [SmokeTestsPage.tsx](src/frontend/src/pages/SmokeTestsPage.tsx)
- [JobsPage.tsx](src/frontend/src/pages/JobsPage.tsx)

---

## 4. Infrastructure Notes

### GPU Strategy
- Single L4 GPU in `us-east4` handles all workload types (ComfyUI, visual tagging, transcription)
- `max-instances=1`, `min-instances=0` (scale-to-zero)
- Quota increase pending approval -- once approved, bump `max-instances` first

### Deferred Items
- Database audit logging (deferred until SQL instance scales up)
- Animation system CSS rules (`data-animation` attribute exists but has no effect yet)
- Multi-service GPU architecture (separate services per job type) deferred until quota increase
