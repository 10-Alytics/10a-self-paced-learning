# SPLMS Tracker — GitHub Pages Deployment

Self-Paced Learning Management System tracker for 10Alytics. This repo contains exactly
one file that needs to be live: `index.html`. Everything else is documentation.

Owner: Prince Ubong (ubong@10alytics.org)

---

## 1. Files in this repo

| File | Purpose |
|---|---|
| `index.html` | The entire app — UI, styling, and logic in one static file. This is what GitHub Pages serves. |
| `README.md` | This file. |

There is no build step, no `package.json`, no server. It is a static HTML file that runs
entirely in the visitor's browser.

---

## 2. Go live on GitHub Pages (one-time, ~3 minutes)

1. Create a new GitHub repository (or use an existing one), e.g. `10alytics/splms-tracker`.
2. Upload `index.html` to the root of the repo (drag-and-drop on github.com works, or `git push`).
3. In the repo: **Settings → Pages**.
4. Under "Build and deployment", set **Source: Deploy from a branch**, **Branch: main**, folder **/ (root)**. Save.
5. GitHub gives you a URL like `https://10alytics.github.io/splms-tracker/` within a minute or two. That's the live tracker.

Any time you push a new `index.html` to `main`, GitHub Pages redeploys automatically —
usually live within a minute.

---

## 3. How the data source works — and how it refreshes

The tracker ships with a **fixed snapshot** of participant data baked into the file
(the `BUNDLE` constant near the top of the `<script>` block). That snapshot never changes
on its own — it's just what the page shows if you don't connect a live source, or if the
live source is briefly unreachable.

To make entries **actually refresh** without you having to rebuild or redeploy anything,
connect the page to your live Google Sheet using **Publish to the web**, which is a
built-in Google Sheets feature that exposes a read-only, always-current CSV link for a
sheet — no Apps Script, no API keys, no server required.

### One-time setup

1. Open your **SPLMS_Data_Model** Google Sheet (the one with `ParticipantDB`, `Programmes`,
   and `Settings` tabs).
2. **File → Share → Publish to web.**
3. In the dialog: pick the **ParticipantDB** sheet from the dropdown, format **Comma-separated
   values (.csv)**, click **Publish**. Copy the URL it gives you.
4. Repeat for the **Programmes** sheet, and again for the **Settings** sheet. You'll end up
   with three CSV URLs.
5. Open `index.html`, find the `DATA_SOURCE` block near the top of the `<script>` section:
   ```js
   const DATA_SOURCE = {
     participantsCsvUrl: '', // paste ParticipantDB CSV URL here
     programmesCsvUrl: '',   // paste Programmes CSV URL here
     settingsCsvUrl: '',     // paste Settings CSV URL here
   };
   ```
6. Paste the three URLs in, save, commit, and push. GitHub Pages redeploys automatically.

### What happens after that

Every time someone opens the live page, it fetches those three CSV links fresh (no
caching) and rebuilds the table from whatever is currently in the Sheet. Google refreshes
a published CSV automatically, typically within about five minutes of an edit landing in
the Sheet — so the flow is:

**Someone edits ParticipantDB → Google republishes the CSV (~5 min) → next page load pulls it in.**

No GitHub push, no rebuild, no redeploy needed for data changes — only for changes to
`index.html` itself (styling, features, the `DATA_SOURCE` URLs).

You'll see a small status pill in the top-right of the header confirming this:
- **Green "Live data — refreshed HH:MM"** — pulled fresh data successfully.
- **Amber "Demo data — data source not configured"** — `DATA_SOURCE` URLs are still blank.
- **Red "Could not reach data source"** — a URL is wrong, unpublished, or the network
  request failed; the page falls back to the last-known snapshot so it never shows a
  blank screen.

The CSV parser reads columns **by header name**, not by position, so you can reorder
columns in the Sheet freely. Renaming a header the page depends on (e.g. `Participant_ID`,
`Programme_Code`, `Modules_Completed`) will break that field — check the mapping table in
`index.html`'s `refreshLiveData_()` function if you rename anything.

---

## 4. Making it stable

- **Keep header names fixed.** The live-data mapping matches on exact column headers
  (`Participant_ID`, `Full_Name`, `Programme_Code`, etc.). Renaming a header silently
  drops that field back to blank/default rather than erroring — check the tracker after
  any Sheet restructuring.
- **Don't un-publish the Sheet.** Publish-to-web links stay valid until someone explicitly
  stops publishing (File → Share → Publish to web → Stop publishing) or unshares the Sheet.
  If that happens, the badge turns red and the page falls back to the snapshot — it degrades
  gracefully rather than breaking.
- **Local edits made directly on the tracker page (Add participant, inline edits) are
  browser-local only.** They're stored via `localStorage` on that visitor's device and are
  **never sent back to the Google Sheet.** Two different staff members looking at the
  tracker on two different computers will not see each other's manual edits — only what's
  actually in the Sheet. Treat in-page edits as a personal scratch layer, not as how you
  update the source of truth. Update the Sheet directly (or through the Apps Script
  automation, if you're running Phase 2) for anything that needs to be seen by everyone.
- **Privacy:** GitHub Pages sites are public on the free tier — anyone with the URL can
  view it, and it can be indexed by search engines unless you add a `robots.txt`/
  `noindex` tag. This tracker currently displays participant names and emails. Before
  sharing the live link outside the ops team, decide whether that's acceptable, or trim
  the visible fields (edit the row-rendering code in `index.html` to drop `email`), or move
  hosting somewhere with access control (e.g. a private Netlify/Vercel deployment with
  password protection, or GitHub Pages on a GitHub Enterprise plan, which supports private
  Pages sites).
- **CSV parsing failures are silent by design** (falls back to snapshot) so the page never
  shows a broken UI to end users — but that also means a bad URL can go unnoticed. Check
  the status badge periodically, or open the browser console (F12) where fetch errors are
  logged.

---

## 5. The recurring task (keeping it updated)

Once the one-time setup in Section 3 is done, **there is no recurring manual task for data
to stay current** — every page load re-pulls the Sheet automatically. Your only ongoing job
is the same one the SOP already asks for: **keep ParticipantDB up to date** (directly, or
via the Weekly Feedback Form + Apps Script automation if that's running). The tracker just
reflects whatever is in the Sheet.

If you'd rather not expose a live publish-to-web link (see the privacy note above) and
prefer a manual refresh cadence instead, here's that alternative task:

1. Download `ParticipantDB` as CSV from Google Sheets (File → Download → CSV), for the
   sheet only.
2. Open `index.html`, locate the `BUNDLE.participants` array, and replace it with the new
   data (a short conversion script can automate this — ask for one if you'd like it).
3. Commit and push. GitHub Pages redeploys with the new snapshot.

Do this as often as you want the public page to reflect reality — daily or weekly is
typical for a manual cadence. This is more work than Section 3's live link, but keeps
participant data off the public internet between refreshes.
