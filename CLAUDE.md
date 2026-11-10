# CLAUDE.md — HLM Block Tracker: hosting handoff

You are picking this up from a previous Claude session. The app is **built and tested**; your job is to
**provision Firebase and host it on GitHub Pages**, then verify it end to end. Don't redesign or refactor the app.

## What this is

A single-page web app for tracking Faizan's 11-week HLM powerlifting block day by day
(block runs Mon 5 Oct 2026 → Sun 20 Dec 2026). It was converted from his spreadsheet
`HLM_11_Week_Program_v2.xlsx`. For every session it shows the prescribed loads and logs
kg / reps / RPE / done per set, plus bar speed, left-trap notes, session notes, a weekly
recovery log (bodyweight, sleep/appetite/motivation 1–5) and the Week 11 max estimate.

Decisions already made by the user — do not reopen them:
- **Hosting:** GitHub Pages, **new public repo**.
- **Backend:** Firebase **Firestore**.
- **Auth:** **Google sign-in** (Firebase Auth); data locked to the signed-in user's UID.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app (HTML + CSS + one `<script type="module">`). Imports the Firebase JS SDK v10.12.2 from `www.gstatic.com` and `./config.js`. |
| `config.js` | `export const firebaseConfig = {...}`. **Currently placeholders (`REPLACE_ME`)**. While it holds placeholders the app shows a "Finish setup" screen. |
| `firestore.rules` | Security rules: only `users/{uid}/**` is readable/writable, and only by that uid. Everything else is denied. |
| `firebase.json` | Points the Firebase CLI at `firestore.rules` (used for `firebase deploy --only firestore:rules`). |
| `.nojekyll` | Stops GitHub Pages from running Jekyll. |
| `README.md` | Human-readable setup notes (the same steps as below, in short). |

## Architecture (enough to debug, not to rewrite)

- **Program logic is pure JS, derived from 1RMs.** `buildDay(week, dayIndex, orm)` reproduces the sheet's
  formulas (`MROUND(1RM × %, 2.5)`, TM factor 0.9, backoff 0.85, deload 40/50/60% of TM, Week 11 ramp
  40/60/75/85/90/95% + 2.5 kg bench / +5 kg squat and deadlift). The secondary-lift loads (tempo squat, tempo bench, Larsen bench,
  paused deadlift) are a hardcoded `SEC` table. Prescribed loads are **never stored**; only the 1RM inputs are.
  These values were checked against the spreadsheet for weeks 1, 3, 4, 7, 9 and 11. **Don't touch them.**
- **Data model (Firestore):**
  ```
  users/{uid}/settings/main     { s, b, d, updatedAt }                  // 1RM inputs: squat, bench, deadlift
  users/{uid}/sessions/w4-mon   { sets: { "<exerciseId>": { "<setIndex>": {w, r, rpe, done} } },
                                  warm: {"0":true,...}, speed: {...}, trap: {...}, notes, done, updatedAt }
  users/{uid}/weeks/w4          { bw, sleep, appetite, motivation, note, updatedAt }
  ```
  Session IDs: `w{1..11}-{mon|tue|wed|thu|fri|sat|sun}`. At most 77 session docs plus 11 week docs plus 1 settings doc.
- **Writes** go through `patch(col, id, fragment)`. It merges the fragment locally, debounces for 500 ms per document, then calls
  `setDoc(ref, fragment, {merge: true})` with **only the changed fields**. That way two devices editing different sets of
  the same day don't clobber each other.
- **Reads** use one `onSnapshot` per collection under `users/{uid}`. A document with an unsent local edit skips
  incoming snapshots. The page doesn't redraw while an input has focus (`requestRender` waits).
- **Offline:** `initializeFirestore(..., {localCache: persistentLocalCache({tabManager: persistentMultipleTabManager()})})`.
- **Auth:** `signInWithPopup(GoogleAuthProvider)`. The popup is used on purpose, not a redirect: redirect sign-in on a third-party
  `github.io` domain breaks under browser storage partitioning.
- **Backup:** Block tab → "Your data" → Export JSON / Import JSON (batch write) / Sign out.
- The glue code was tested against in-memory stubs of the Firebase SDK: fragment writes, snapshot round-trip, gate screens.
  **It has never run against a real Firebase project.** Treat the first real run as the real test.

## Runbook

Run these on the user's Mac. Ask before anything that costs money or can't be undone.

### 0. Preflight
```bash
node -v; npm -v; git --version
firebase --version || npm i -g firebase-tools
gh --version && gh auth status          # install via `brew install gh` and `gh auth login` if missing
```

### 1. Ask the user before pushing anything public
The repo **and** the hosted site are public. `index.html` contains the full program, including the **Rules tab →
"Open items"** card. That card mentions a left-trap issue, a medication course (ketorolac) and last bodyweight.
Ask the user whether to **keep or delete that card** before the first push. To delete it, remove the
`<div class="card"><h3>Open items</h3>…</div>` block inside `renderRules()`. Commits also carry the user's
git author email; ask if they want a GitHub noreply address instead.

### 2. Firebase project
```bash
firebase login
# The project ID must be globally unique, e.g. hlm-tracker-<random 4-6 chars>
firebase projects:create <PROJECT_ID> --display-name "HLM Tracker"
firebase use --add <PROJECT_ID>          # writes .firebaserc
```

### 3. Firestore (Mumbai region)
```bash
firebase firestore:databases:create "(default)" --location=asia-south1 --project <PROJECT_ID>
firebase deploy --only firestore:rules --project <PROJECT_ID>
```
If `firestore:databases:create` isn't available in the installed CLI version, create the database in the console instead
(Build → Firestore Database → Create → production mode → `asia-south1`), then deploy the rules.

### 4. Web app config → `config.js`
```bash
firebase apps:create WEB "HLM Tracker" --project <PROJECT_ID>
firebase apps:sdkconfig WEB <APP_ID> --project <PROJECT_ID>
```
Copy `apiKey, authDomain, projectId, storageBucket, messagingSenderId, appId` into `config.js`, keeping the
`export const firebaseConfig = {...}` shape. These values identify the project and are **not secrets**; committing them is normal.
Access is enforced by the rules plus sign-in.

### 5. Google sign-in (the user does this in the console; the CLI can't)
Firebase console → Authentication → Get started → Sign-in method → **Google → Enable** (choose a support email) → Save.
Then go to Authentication → Settings → **Authorized domains** → add `<github-username>.github.io`.
(`localhost` is already listed.) Walk the user through it and wait for confirmation.

### 6. Local smoke test before pushing
```bash
python3 -m http.server 8080   # ES modules need http://, not file://
```
Open http://localhost:8080 → Sign in with Google → tick one set → check that the Firestore console shows
`users/<uid>/sessions/w?-???` holding only the fields you touched.

### 7. GitHub repo and Pages
```bash
git init -b main && git add . && git commit -m "HLM block tracker"
gh repo create hlm-tracker --public --source . --push
gh api -X POST repos/{owner}/hlm-tracker/pages -f "source[branch]=main" -f "source[path]=/"
gh api repos/{owner}/hlm-tracker/pages --jq .html_url    # usually https://<user>.github.io/hlm-tracker/
```
The first Pages build takes about a minute.

### 8. Verify on the live URL (desktop, then phone)
1. The sign-in screen appears; Google popup sign-in succeeds. An `auth/unauthorized-domain` error means step 5's domain wasn't added.
2. Train tab opens on today's week and day. Ticking an empty set fills in the prescribed kg and reps; the status shows Saving… → Saved.
3. Open the same day on a second device: the tick appears live.
4. In an incognito window signed into a different Google account, the logs are empty (the rules isolate users).
5. Block tab → Export JSON downloads a file.

## Alternative, only if the user asks
Firebase Hosting instead of Pages: add `"hosting": {"public": ".", "ignore": ["firebase.json", "**/.*", "CLAUDE.md", "README.md"]}`
to `firebase.json` and run `firebase deploy --only hosting`. You get a `*.web.app` URL, and its domain is authorized for sign-in automatically.

## Open question for the user (not a blocker)
The spreadsheet has **Overhead Press** on Thursdays (and in the deloads). The user earlier mentioned permanently replacing OHP
with a **Machine Shoulder Press**. Ask whether to rename it. If yes, change the `'ohp'` exercise name in `buildDay()`
(three places: normal Thursday, deload Thursday, test Thursday) and keep the id `ohp` so any data already logged still lines up.

## Don't
- Change load percentages, rounding, the `SEC` table or exercise IDs (stored data is keyed by exercise ID).
- Switch auth to redirect, loosen `firestore.rules`, or add an email allowlist to the rules (the repo is public).
- Add a build step or framework. It's deliberately a static, no-build site.
