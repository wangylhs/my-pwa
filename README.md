# Timesheet PWA

A small installable web app for tracking daily work hours. Built as a single HTML file plus a service worker, hosted free on GitHub Pages, with sign-in and cross-device sync via Firebase.

Live URL: <https://wangylhs.github.io/my-pwa/>

## Features

- Per-month timesheet with Start / End / Duty / Lunch fields and auto-calculated hours.
- Works offline once installed; edits queue up and sync when back online.
- Sign in with **email + password** or **Google**. Both methods land on the same data (keyed by email).
- Cross-device sync via Firestore — same email, same data on every device.
- Local copy is always kept in `localStorage` as a cache, so the app stays responsive offline.
- Allowlist gate: only emails listed in the source code (and mirrored in security rules) can sign in.

## Architecture

```
docs/                         <- served by GitHub Pages
  index.html                  <- entire app: HTML, styles, ESM script
  sw.js                       <- service worker (cache name: timesheet-v3)
  manifest.webmanifest        <- PWA manifest
  icons/
    icon-192.png
    icon-512.png
README.md
```

No build step. Open `docs/index.html` directly or serve `docs/` with any static file server.

### Cloud services

- **Firebase Authentication** — Email/Password + Google providers.
- **Cloud Firestore** — single document per user at `users/{email}` containing `{ months, monthOrder, email, updatedAt }`.

Firebase project: `timesheet-pwa-b30e2`. The `apiKey` in `index.html` is a public project identifier, not a secret — security is enforced by Firestore rules and the Auth allowlist.

### Data shape

Each user's Firestore doc:

```json
{
  "months": {
    "2026-04": { "rows": [ { "date": "2026-04-01", "day": "Wed", "start": "08:15", "end": "16:30", "duty": "Bake", "lunch": "0.5" }, ... ] },
    "2026-05": { ... }
  },
  "monthOrder": ["2026-04", "2026-05"],
  "email": "user@example.com",
  "updatedAt": <serverTimestamp>
}
```

Local cache uses the same shape under `localStorage` key `timesheet:v1`.

## Allowlist (adding or removing users)

Two places must agree, or sign-in will succeed but data access will be blocked:

1. **`docs/index.html`** — the `ALLOWED_EMAILS` array (search for it near the top of the `<script type="module">`). This is the UX gate.
2. **Firestore security rules** in Firebase Console → Firestore → Rules tab. Edit the `isAllowed()` function. This is the real security boundary.

After editing, commit/push the HTML change AND click Publish on the rules tab.

## Local development

```bash
cd docs
python3 -m http.server 8000
# open http://localhost:8000
```

`localhost` is already on Firebase Auth's authorized domains by default, so sign-in works locally without extra setup.

When you change `index.html` or `sw.js`, bump the cache name in `sw.js` (`timesheet-v3` → `v4`, etc.) so installed PWAs pick up the new shell.

## Deployment

Push to `main`. GitHub Pages serves `docs/` automatically (configured in repo Settings → Pages). Allow ~30 seconds, then hard-refresh the live URL to activate the new service worker.

## Security model

- **Auth allowlist** — only listed emails can sign in successfully (UX) or read/write data (rules).
- **Per-user isolation** — Firestore rules require `request.auth.token.email == {email}` in the doc path; one user cannot read or write another user's doc.
- **Public API key** — Firebase web API keys are intended to be public. They identify the project, not authorize access. All access control lives in Auth + Firestore rules.

If you need to revoke a user, remove their email from both the `ALLOWED_EMAILS` array and the rules `isAllowed()` list, redeploy, and republish rules. Their existing Firestore doc is left in place; if you also want to delete it, do so manually in the Firestore console.

## License

Personal project, no license specified. Not intended for redistribution.
