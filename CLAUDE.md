# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Marna Proxy frontend panel — static HTML/CSS/JS served from a Cloudflare Worker's `ASSETS` binding. Administrators manage their proxy through this panel; end-users get subscription configs through `user/index.html`. Version 4.0.0.

**No build step.** No npm, no bundler, no framework. Edit files directly — they're deployed as-is.

## File map

| Path | Role |
|---|---|
| `login/index.html` | Admin login (POSTs to `/login`, supports 2FA, redirects to `/admin`) |
| `admin/index.html` | Admin SPA dashboard. Inline CSS + loads `qrcode.min.js` + inline `nova-admin.js` |
| `user/index.html` | Public end-user config hub — subscription formats, QR codes, per-app deep links, radar scanner |
| `install/index.html` | Legacy setup wizard (KV + D1 + password) |
| `install/install.html` | Setup wizard v3.1+ (D1-only, no KV step) |
| `nova-admin.js` | Entire admin panel application logic (~1259 lines vanilla JS) |
| `qrcode.min.js` | QR code generation (global `QRCode` constructor) |
| `version.json` | `{ version, released, notes, worker_url }` |
| `logo*.svg`, `logo.png`, `flag-iran.svg` | Static brand assets |

## Admin SPA architecture

`admin/index.html` is a single-page app with no URL router. Views are `<div class="view" id="view-*">` elements, shown/hidden by toggling an `active` class. The sidebar inlines `data-p` attributes on `<a>` links; the click handler in `nova-admin.js:438` maps `data-p` to the matching view ID and lazy-loads data for that page.

Page views: `home` (dashboard), `distribution` (subscriptions), `network` (routing/DNS/WARP), `cleanips` (IP scanner), `users`, `settings` (protocol/transport/TLS/security/hosts/mirror), `activity` (notifications/Telegram), `logs`, `guide`.

Each section has its own save button — there is no single "save all." Each save handler gathers form fields into the global `cfg` object, then calls `saveConfig()` which POSTs `cfg` as JSON to `/admin/config.json`.

## API surface

All endpoints are on the same origin (the Worker). Cache-busting via `?_t=<timestamp>` on GETs.

**Core config:** `GET /admin/config.json`, `POST /admin/config.json` (JSON body = full `cfg` object)
**Network:** `GET|POST /admin/network-settings.json` (routing, DNS, WARP settings)
**Custom IPs:** `GET|POST /admin/ADD.txt` (plain text, one `ip:port#name` per line)
**System status:** `GET /admin/system.json` (colo, KV status, worker usage, traffic)
**WARP:** `GET|POST /admin/warp.json` (registration, license, WoW)
**Users:** `GET|POST /admin/users.json` (`{ multiUser, users[] }`)
**Domains:** `GET /admin/domains`, `GET /admin/domains?check=1` (health check)
**Logs:** `GET /admin/log.json` (array of events)
**Announcements:** `GET /admin/announcement` (central broadcast)
**Security:** `GET /admin/security/status`, `/reveal`, `/2fa-setup`; `POST /admin/security/change-password`, `/2fa-enable`, `/2fa-disable`
**Fleet management:** `GET /admin/central/stats`, `POST /admin/central/announcement`
**Scanner:** `GET /admin/bestip?loadIPs=<src>&port=<p>`, `GET /admin/whoami`
**Notifications:** `POST /admin/cf.json` (Cloudflare token), `POST /admin/tg.json` (Telegram bot)
**Mirror:** `POST /admin/publish-mirror`
**Subscription (public):** `GET /sub?token=<token>&b64|&clash|&sb|&surge|&quanx|&loon`

## i18n & theming

**Translations:** Single `T` object in `nova-admin.js:10` with `T.en` and `T.fa` keys. Accessed via `t(key)` which falls back to `T.en`. Language stored in `localStorage` key `nova_lang` (login/install pages use `nova-lang`). RTL/LTR set via `document.documentElement.dir`.

**Theme:** CSS custom properties (`--bg`, `--card`, `--tx`, `--ac`, `--bd`, etc.) on `:root`. Dark is default; light theme via `html[data-theme=light]` overrides. Toggled by `data-theme` attribute. Stored in `localStorage` key `nova_theme` (or `nova-theme` on standalone pages).

## Key globals & helpers in nova-admin.js

- `$(id)` — `document.getElementById(id)`
- `cfg` — global config object loaded from `/admin/config.json`
- `lang` / `theme` — current language (`'en'|'fa'`) and theme (`'light'|'dark'`)
- `t(key)` — translate a key
- `toast(msg, err)` — show temporary notification
- `copyText(str)` — clipboard copy + toast
- `esc(str)` — HTML-escape a string
- `setTg(id, on)` — toggle a switch element's `.on` class
- `PANEL_VERSION` — version of the dashboard files, shown in sidebar
- `PORTs` — `[443, 2053, 2083, 2087, 2096, 8443]`
- `ISPS` — array of `{ name, fa, asn }` for Iranian operators
- `FORMATS` — array of `{ name, fa, icon, q }` for subscription formats

## Auth

Cookie-based session (server-side). The panel detects expired sessions by checking `r.redirected || r.url.indexOf('/login') > -1` on fetch responses, then shows a "Session expired" toast. The `ADMIN` Cloudflare env var acts as a recovery password. 2FA uses TOTP (Google Authenticator compatible).

## Constraints

- **No framework.** Keep everything vanilla JS. No new dependencies.
- **No npm, no bundler.** Files are served directly. No build step to add.
- **CSS is inline.** Each HTML page has its own `<style>` block. No external stylesheets (except Google Fonts).
- **RTL-aware.** Every UI change must work in both LTR (English) and RTL (Farsi). Use logical properties (`inset-inline-start`, `margin-inline-start`, `padding-inline-end`) and test both directions.
- **Worker domain only.** The panel is served from the Worker's own domain. No hardcoded external origins for API calls — all relative paths.
- **Session expiry.** Wrap sensitive write operations with the session-expiry check pattern used in `loadUsersPage()` and `saveUsers()`.
