# CLAUDE.md

This project refactors worker.js for Marna Proxy (paid version). My colleague develops Nova Proxy (free version), implements frontend changes, shares updates with me, and I apply relevant changes to Marna Proxy. All changes require clear commit messages for tracking.

Guidance for Claude Code (claude.ai/code) working with this repository.

## Project overview

Marna Proxy frontend panel — static HTML/CSS/JS from Cloudflare Worker's `ASSETS` binding. Admins manage proxy; end-users get configs via `user/index.html`. Version 4.0.0.

**No build step.** No npm, bundler, or framework. Edit and deploy files directly.

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