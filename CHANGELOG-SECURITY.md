# NEO PRO SISTEM — build / security patch (iter2-fixed)

## Security
- Admin passwords: scrypt (N=16384) with per-password salt. Legacy SHA256 accepted once and auto-migrated.
- Login checks **stored** hash + email (no more hardcoded dual passwords for auth).
- `ensureAdminCreds` no longer force-resets password to default.
- Bootstrap creds only used when DB is empty; override via `ADMIN_BOOTSTRAP_PASSWORD` / `ADMIN_BOOTSTRAP_EMAIL`.
- Session tokens 32 bytes; TTL cleanup (default 72h via `ADMIN_SESSION_TTL_HOURS`).
- Removed hardcoded OpenRouter API key from source (use env `OPENROUTER_API_KEY` or settings).
- Media upload: SVG blocked (XSS).
- HTML sanitizer: no `data:image/svg+xml`, tighter CSS filter, more dangerous tags stripped.

## Admin UX
- Removed tab «Блоки главной» (Вверх/Вниз) — duplicate of visual page-builder.
- «Продвижение»: dashboard of integrations (connected / not) + progress bar, then all fields.

## Run
```bash
cd workspace
npm ci   # or npm install
npm run dev
```
Admin: /admin  (bootstrap email/password until you change password in Settings)

