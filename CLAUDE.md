# CLAUDE.md

This project refactors the frontend panel for Marna Proxy (paid version). My colleague develops Nova Proxy (free version), implements frontend changes, shares updates with me, and I apply relevant changes to Marna Proxy. All changes require clear commit messages for tracking.

Guidance for Claude Code (claude.ai/code) working with this repository.

## Overview

Marna Proxy frontend panel — static HTML/CSS/JS deployed via GitHub Pages and served from Cloudflare Worker's `ASSETS` binding. Derived from the open-source Nova Proxy project. Admins manage proxy; end-users get configs via `user/index.html`. Version 4.0.0.

**No build step.** No npm, bundler, or framework. Edit and deploy files directly.

## Custom Features for Marna Project

### Branding & Naming Rules
1. Replace all non-critical occurrences of **"Nova"** with **"Marna"** wherever doing so does not introduce functional, compatibility, migration, or upstream synchronization issues.
2. Replace all Persian occurrences of **"نوا"** with **"مارنا"** under the same conditions.
3. Do not rename identifiers, storage keys (`localStorage` keys like `nova_lang`, `nova_theme`), API fields, or compatibility-related values if doing so would break existing functionality, user data, or upstream merge compatibility.
4. When introducing new UI labels, comments, documentation, or user-facing text, always use the **Marna** brand.

### Commercial Project Rules
1. MarnaProxy is a customized commercial project and must not be presented as a free service.
2. Remove or update any user-facing text, documentation, comments, panel content, or descriptions that explicitly advertise the service as free.
3. Preserve upstream functionality while removing unnecessary references to free-service marketing language.

### Customization Preservation Policy
1. Marna-specific customizations always take priority over cosmetic upstream changes.
2. During upstream merges, preserve Marna branding, commercial-project modifications, and RTL/i18n customizations unless explicitly instructed otherwise.
3. Any upstream update that conflicts with Marna customizations must be reviewed before merging.

## File map

**Path** | **Role**
`login/index.html` | Admin login with 2FA
`admin/index.html` | Admin SPA dashboard
`user/index.html` | Public config hub with QR codes
`install/index.html` | Legacy setup wizard (KV + D1)
`install/install.html` | Setup wizard v3.1+ (D1-only)
`nova-admin.js` | Admin panel logic (~1259 lines)
`qrcode.min.js` | QR generation
`version.json` | Version info
`logo*.svg`, `logo.png`, `flag-iran.svg` | Brand assets

## Admin SPA architecture

`admin/index.html` is a single-page app without URL routing. Views are `<div class="view" id="view-*">` shown/hidden via `active` class. Sidebar links with `data-p` attributes map to view IDs. Click handler at `nova-admin.js:438` lazy-loads data.

**Views:** home, distribution, network, cleanips, users, settings, activity, logs, guide.

Each section saves independently. Save handlers gather form fields into global `cfg`, call `saveConfig()` which POSTs to `/admin/config.json`.

## API surface

All endpoints same-origin. Cache-busting: `?_t=<timestamp>`.

**Config:** `GET|POST /admin/config.json`
**Network:** `GET|POST /admin/network-settings.json`
**IPs:** `GET|POST /admin/ADD.txt`
**Status:** `GET /admin/system.json`
**WARP:** `GET|POST /admin/warp.json`
**Users:** `GET|POST /admin/users.json`
**Domains:** `GET /admin/domains[?check=1]`
**Logs:** `GET /admin/log.json`
**Announcements:** `GET /admin/announcement`
**Security:** `GET /admin/security/{status,reveal,2fa-setup}`; `POST /admin/security/{change-password,2fa-enable,2fa-disable}`
**Fleet:** `GET /admin/central/stats`; `POST /admin/central/announcement`
**Scanner:** `GET /admin/bestip`, `/whoami`
**Notifications:** `POST /admin/cf.json`, `/tg.json`
**Mirror:** `POST /admin/publish-mirror`
**Public sub:** `GET /sub?token=<token>&{b64|clash|sb|surge|quanx|loon}`

## i18n & theming

**Translations:** `T` object in `nova-admin.js:10` with `T.en` and `T.fa`. Access via `t(key)` with fallback to `T.en`. Language in `localStorage` key `nova_lang` (or `nova-lang`). RTL/LTR via `document.documentElement.dir`.

**Theme:** CSS custom properties on `:root`. Dark default; light via `html[data-theme=light]`. Toggle via `data-theme`. Stored in `localStorage` key `nova_theme` (or `nova-theme`).

## Key globals & helpers in nova-admin.js

`$(id)` — getElementById
`cfg` — global config from `/admin/config.json`
`lang` / `theme` — current language and theme
`t(key)` — translate
`toast(msg, err)` — notification
`copyText(str)` — clipboard copy
`esc(str)` — HTML-escape
`setTg(id, on)` — toggle switch
`PANEL_VERSION`, `PORTs`, `ISPS`, `FORMATS` — constants

## Auth

Cookie-based sessions. Detects expiry via `r.redirected || r.url.indexOf('/login') > -1`, shows "Session expired" toast. `ADMIN` env var = recovery password. 2FA uses TOTP.

## Constraints

- **No framework.** Vanilla JS only.
- **No npm/bundler.** Files served directly.
- **Inline CSS.** Each HTML has `<style>` block.
- **RTL-aware.** Support LTR and RTL. Use logical properties (`inset-inline-start`, `margin-inline-start`, `padding-inline-end`).
- **Worker domain only.** Relative API paths.
- **Session expiry.** Check pattern from `loadUsersPage()` and `saveUsers()`.

## Git Workflow — Mandatory

### Commit Rules
- Every meaningful change must be committed separately.
- Commit messages must be concise, descriptive, and written in English.
- Commit format:
```
<type>: <short description>
<optional body — explain WHY, not just WHAT>
```
- Types: `feat`, `fix`, `perf`, `refactor`, `upstream`, `config`, `style`, `i18n`

### Branch Strategy
- `main` — Deployable production branch
- `upstream` — Tracks Nova Panel frontend updates (merge-only, no manual modifications)
- Use a dedicated branch for every major feature or fix.

## Applying Upstream Updates — Standard Process

When the Nova Proxy frontend (Nova Panel) releases updates, follow these steps:

1. **Fetch upstream:**
   ```bash
   git checkout upstream
   git pull upstream main   # or fetch the latest Nova Panel release
   git commit -m "upstream: fetch Nova Panel <version>"
   ```

2. **Three-way merge:**
   ```bash
   git checkout main
   git merge upstream --no-commit
   ```
   Resolve conflicts while prioritizing preservation of Marna customizations (branding, commercial rules, RTL/i18n).

3. **Re-apply custom modifications:**
   - Keep useful upstream improvements and bug fixes.
   - Preserve Marna branding, commercial modifications, and RTL/i18n customizations.
   - Review all user-facing strings for branding compliance.

4. **Test before deployment:**
   - Verify admin panel functionality (all views load, save operations work).
   - Test user-facing pages (`user/index.html`, `login/index.html`).
   - Confirm RTL/LTR switching and theme toggling.
   - Validate API communication with the worker backend.

5. **Commit the merge:**
   ```bash
   git commit -m "upstream: merge Nova Panel <version> with Marna customizations"
   ```

6. **Push to remote:**
   ```bash
   git push origin main
   ```

> **Last Updated:** 2026-07-21
> **Base Version:** Nova Proxy V4.0.0
