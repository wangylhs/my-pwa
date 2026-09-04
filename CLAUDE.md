# Project notes for Claude

This is a small personal **Timesheet PWA**. See `README.md` for the user-facing overview.

## Shape

- Single-file vanilla JS app at `docs/index.html` (HTML, inline CSS, ESM `<script type="module">` — no build step, no frameworks).
- Service worker at `docs/sw.js`. Cache name is `timesheet-v3` and **must be bumped on every shipped change to `docs/`**, otherwise installed PWAs serve a stale shell.
- PWA manifest at `docs/manifest.webmanifest`. Icons in `docs/icons/`.
- Firestore rules at `firestore.rules`, configured by `firebase.json` and bound to project `timesheet-pwa-b30e2` in `.firebaserc`.
- Hosted on GitHub Pages from the `docs/` folder. No CI, no other tooling.

## Tech in play

- **Firebase Auth** (Email/Password + Google) gates the app.
- **Cloud Firestore** stores per-user data at `users/{email}` (doc ID is the lowercased email so both auth methods land on the same data).
- Firebase project: `timesheet-pwa-b30e2`. Web SDK loaded from `gstatic.com` CDN, version pinned at `10.13.0` in `index.html` imports.
- `apiKey` in `index.html` is a public project identifier, not a secret. Don't try to hide it.

## Local dev

```
cd docs && python3 -m http.server 8000
```

Open <http://localhost:8000>. `localhost` is allowlisted in Firebase Auth by default.

## Deploy

Commit and push. GitHub Pages picks up `docs/` automatically. Wait ~30s, hard-refresh the live URL to activate the new service worker.

Firestore rules are a separate deployment:

```
firebase deploy --only firestore:rules
```

## Recipes (things you will actually be asked to do)

### Add or remove an allowlisted user
Edit BOTH places — they must match or sign-in succeeds but data access fails:
1. `ALLOWED_EMAILS` array in `docs/index.html`.
2. `isAllowed()` function in `firestore.rules`, then deploy with `firebase deploy --only firestore:rules`.

### Change anything in `docs/`
Bump the cache name in `docs/sw.js` (`timesheet-v3` → `v4` etc.) in the same commit.

### Change the data shape
The per-user doc shape is `{ months: { [yyyy-mm]: { rows: [...] } }, monthOrder: [...], email, updatedAt }`. The same shape is mirrored in `localStorage` under key `timesheet:v1`. Keep both in sync.

## Don'ts

- Don't reintroduce Google Sheets sync. We removed ~250 lines of it; Firestore replaces it.
- Don't add a framework or a build step. Single-file vanilla is a feature.
- Don't switch the Firestore doc key away from email. UID-keyed docs would split data across auth methods (Google vs. email/password produce different UIDs for the same person).
- Don't add `localStorage`/`sessionStorage`-backed config to the auth flow — Firebase Auth handles its own session persistence.
