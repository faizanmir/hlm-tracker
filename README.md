# HLM Block Tracker

Day-by-day tracker for the 11-week HLM powerlifting block (5 Oct – 20 Dec 2026).
Static site on GitHub Pages; data in Firebase Firestore behind Google sign-in.

## Data model

```
users/{uid}/settings/main      { s, b, d }                       // 1RM inputs; all loads derive from these
users/{uid}/sessions/w4-mon    { sets: { "bench-main": { "0": {w,r,rpe,done} } }, warm, speed, trap, notes, done }
users/{uid}/weeks/w4           { bw, sleep, appetite, motivation, note }
```

Writes are field-level merges (`setDoc(..., {merge:true})` with only the changed fragment),
so two devices editing different sets of the same session don't overwrite each other.
Firestore's persistent local cache keeps the app usable offline; writes sync on reconnect.

## Setup (one-time, ~10 min)

1. **Firebase project** — https://console.firebase.google.com → Add project (Analytics off is fine).
2. **Firestore** — Build → Firestore Database → Create database → production mode → region `asia-south1` (Mumbai).
   Rules tab → paste `firestore.rules` → Publish.
3. **Auth** — Build → Authentication → Get started → Sign-in method → Google → Enable.
   Settings → Authorized domains → add `<your-github-username>.github.io`.
4. **Web app** — Project settings → Your apps → `</>` (Web) → register → copy the config object into `config.js`.
5. **GitHub Pages** — push this folder to a public repo → Settings → Pages → Deploy from branch → `main` / root.
   Site goes live at `https://<user>.github.io/<repo>/`.

## Local dev

```
python3 -m http.server 8080   # ES modules need http://, not file://
```
Add `localhost` to Firebase authorized domains (it is there by default).

## Export / import

Block tab → Your data → Export JSON (backup) / Import JSON (restore, overwrites matching documents).
