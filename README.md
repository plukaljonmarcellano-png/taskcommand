# TaskCommand v2

Professional-grade task, project, and workflow management for finance & audit teams.
Single-user, cloud-synced via Firebase — served as a single static `index.html`.

---

## What's new in v2

**Security**
- Firebase Authentication (Google Sign-In) — no more open database
- Per-user data path (`/users/{uid}`) — your data is only readable by you
- Proper security rules included (`firebase-rules.json`)
- Base64 file uploads removed (they were leaking to public DB); attachments are now external links (Drive / SharePoint / S3 URLs)

**New features**
- **Dashboard view** — KPIs, aging analysis, completion trend, workload, upcoming deadlines
- **Calendar view** — monthly grid with tasks on due dates
- **Reports view** — printable status report, CSV export, JSON backup, audit trail
- **Recurring tasks** — daily / weekly / biweekly / monthly / month-end / quarterly / yearly
- **Time tracking** — actual vs. estimated hours with variance
- **Bulk actions** — multi-select tasks and move / delete in one click
- **Saved views** — save any filter combination as a named view
- **Command palette** — `⌘K` / `Ctrl+K` for keyboard-first navigation
- **Keyboard shortcuts** — `/` search, `n` new task, `1-6` view switch, `?` help
- **Activity log / audit trail** — every change timestamped
- **Undo toast** on delete
- **Light / dark theme** toggle
- **Debounced writes** — no more full-tree overwrite on every keystroke
- **Auto-migration** from v1 data (offered on first login)

---

## First-time setup (~10 min)

### 1. Firebase project

If you don't already have one, create a project at [console.firebase.google.com](https://console.firebase.google.com).

### 2. Enable Realtime Database

- **Build → Realtime Database → Create Database**
- Pick any region
- Start in **locked mode** (we'll deploy custom rules in step 4)

### 3. Enable Google Sign-In

- **Build → Authentication → Get started**
- **Sign-in method** tab → click **Google** → **Enable**
- Add your GitHub Pages domain (`plukaljonmarcellano-png.github.io`) under **Authorized domains** if it's not already there

### 4. Deploy the security rules

- **Build → Realtime Database → Rules** tab
- Paste the contents of `firebase-rules.json` (from this repo) — replace whatever is there
- Click **Publish**

The rules do three things:
- Deny all access by default (`.read: false, .write: false` at root)
- Allow each authenticated user to read/write only their own `/users/{uid}` node
- Allow the app to read your old v1 `/taskcommand` node once for one-time migration (you can tighten this after migration by removing the `taskcommand` block)

### 5. Get your Firebase config

- **⚙ Project Settings → General → Your apps → Web (`</>`)**
- If you don't have a web app registered yet, register one (nickname: anything)
- Copy the entire `firebaseConfig` block — you'll paste it into the app on first load

### 6. Deploy the app

**Option A — GitHub Pages (recommended, matches your current setup):**

```bash
# In your local clone of plukaljonmarcellano-png/taskcommand
git pull
cp index.html README.md firebase-rules.json .   # replace with the v2 files
git add index.html README.md firebase-rules.json
git commit -m "TaskCommand v2 — auth, per-user data, dashboard, calendar, reports"
git push
```

GitHub Pages will pick it up in ~30 seconds. Your existing URL keeps working.

**Option B — anywhere else:** `index.html` is entirely self-contained. Drop it on any static host.

### 7. First launch

- Open the app
- If your Firebase config is not stored locally, paste it into the setup screen
- Sign in with Google
- If v1 data exists in your DB, the app will offer to import it (recommended)
- After successful migration, you can delete the `/taskcommand` node in Firebase Console and remove the `taskcommand` block from `firebase-rules.json` for tighter security

---

## Migrating from v1

The app automatically detects your old v1 data at `/taskcommand` in RTDB on first login and offers a one-click import. Base64 file attachments are stripped for security (they were the largest risk vector); re-add them as external URLs after migration.

To clean up after migration:
1. Firebase Console → **Realtime Database → Data**
2. Hover the `taskcommand` node → click the red `×` → confirm delete
3. Update `firebase-rules.json` — remove the `"taskcommand"` block — and republish

---

## Data model

Everything is scoped under `/users/{your-uid}/`:

```
/users/{uid}/
  tasks/[]        — id, title, notes, section, priority, taskStatus, domain,
                    assignee, project, startDate, due, estimatedHours, actualHours,
                    recurrence, subtasks[], comments[], files[] (URL links only),
                    createdAt, updatedAt, completedAt
  projects/[]     — id, name, domain, status
  people/[]       — display names
  domains/[]      — name, color
  savedViews/[]   — id, name, filter
  activity/[]     — id, type, taskId?, detail, time (last 500 events)
  settings/{}     — reserved for future use
```

---

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `⌘K` / `Ctrl+K` | Command palette |
| `/` | Focus search |
| `n` | New task |
| `1` – `6` | Switch view (Dashboard / Board / List / Calendar / Projects / Reports) |
| `?` | Show shortcut help |
| `Esc` | Close modal or clear focus |
| `⌘↵` / `Ctrl+↵` | Submit forms & comments |

---

## Security posture

| Risk | Mitigation |
|---|---|
| Public database access | Auth required; per-UID rules |
| Unauthenticated tampering | `auth != null` guard on every path |
| Base64 file dump into RTDB | Removed — external URL references only |
| Runaway writes | Debounced saves (400 ms) |
| Data loss | JSON backup export in Settings, activity log capped at 500 events |
| Config leakage | `firebaseConfig` values are public-by-design; DB is protected by rules, not by hiding the config |

**Recommended hardening (optional next steps):**
1. Enable **Firebase App Check** with reCAPTCHA v3 — blocks non-browser access
2. Set up **budget alerts** in Google Cloud Billing (RTDB is on Spark free tier for typical solo use, but Blaze projects should have a cap)
3. Rotate the DB URL periodically if you suspect it has leaked to bots
4. Enable **Cloud Logging** on RTDB for a real audit trail beyond the in-app one

---

## Troubleshooting

**"Sign in failed: auth/unauthorized-domain"**
Add your domain (e.g. `plukaljonmarcellano-png.github.io`) to Firebase Console → Authentication → Settings → Authorized domains.

**"Permission denied" after sign-in**
The security rules haven't been deployed. Repeat step 4.

**Sync stuck on "Syncing…"**
Check the browser console. Most common causes: rules not yet propagated (wait 60 s), DB URL typo, no network.

**Recurring tasks not generating**
They generate at most once per calendar day on app open, from a *completed* recurring task. Re-open the app to trigger.

**Old v1 data doesn't show up for import**
The rules must temporarily allow read on `/taskcommand` for authenticated users. This is included in the shipped `firebase-rules.json`. Verify Rules tab shows the `taskcommand` block.

---

## Version

v2.0.0 — 2026-07-18

Built as a single self-contained HTML file (React 18 + Firebase 10 via CDN, no build step).
Approximately 138 KB gzipped ~40 KB.
