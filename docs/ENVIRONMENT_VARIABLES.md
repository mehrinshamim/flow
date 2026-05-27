# Environment Variables Guide

Flow uses `.env.local` files for app-specific configuration. Each app has its own file.

---

## Quick Setup

```bash
# From the repo root
cp apps/reader/.env.local.example apps/reader/.env.local
cp apps/website/.env.local.example apps/website/.env.local
```

The defaults work out of the box for local development. **No edits needed to get started.**

---

## Reader (`apps/reader/.env.local`)

```env
NEXT_PUBLIC_DROPBOX_CLIENT_ID=
DROPBOX_CLIENT_SECRET=
NEXT_PUBLIC_WEBSITE_URL=http://localhost:7117
```

### Variable Breakdown

| Variable | Required? | Default | What It Does |
|---|---|---|---|
| `NEXT_PUBLIC_WEBSITE_URL` | **Yes** | `http://localhost:7117` | URL of the marketing website. The reader links back to it (e.g., branding, navigation). The default value works for local dev — no changes needed. |
| `NEXT_PUBLIC_DROPBOX_CLIENT_ID` | **No** | _(empty)_ | OAuth Client ID for Dropbox cloud sync. Only needed if you're working on the sync feature. |
| `DROPBOX_CLIENT_SECRET` | **No** | _(empty)_ | OAuth Client Secret for Dropbox. Paired with the Client ID above. Server-side only (no `NEXT_PUBLIC_` prefix). |

### Do I need the Dropbox variables?

**No, not for general development.** The reader works fully without them — you can:

- Open and read ePub files
- Highlight and annotate
- Search within books
- Customize typography and themes
- Use all UI features

The Dropbox variables **only** enable the cloud sync feature, which lets users:

1. Back up their reading progress and annotations to Dropbox
2. Sync their library across devices

### How to set up Dropbox (if needed)

1. Go to [Dropbox App Console](https://www.dropbox.com/developers/apps/create)
2. Choose **Scoped access** → **Full Dropbox**
3. Name your app (e.g., `flow-dev`)
4. Copy the **App key** → paste as `NEXT_PUBLIC_DROPBOX_CLIENT_ID`
5. Copy the **App secret** → paste as `DROPBOX_CLIENT_SECRET`
6. Under **OAuth 2** settings, add `http://localhost:7127/api/callback/dropbox` as a redirect URI

### `NEXT_PUBLIC_` prefix explained

Next.js uses this naming convention to control variable exposure:

- **`NEXT_PUBLIC_*`** → Bundled into the browser JavaScript. Visible to anyone inspecting the page. Used for values the frontend needs (like the Dropbox Client ID for initiating OAuth).
- **No prefix** → Server-side only. Only accessible in API routes and `getServerSideProps`. Used for secrets (like `DROPBOX_CLIENT_SECRET`) that must never reach the browser.

---

## Website (`apps/website/.env.local`)

```env
NEXT_PUBLIC_APP_URL=http://localhost:7127
```

### Variable Breakdown

| Variable | Required? | Default | What It Does |
|---|---|---|---|
| `NEXT_PUBLIC_APP_URL` | **Yes** | `http://localhost:7127` | URL of the reader app. The website's "Open App" / "Try Now" buttons link to this. The default works for local dev. |

---

## Summary

| For local development | Do this |
|---|---|
| Just run the apps | Copy the example files. **Done.** No edits needed. |
| Work on Dropbox sync | Copy the example files + register a Dropbox app + fill in the two Dropbox variables. |
| Deploy to production | Replace `localhost` URLs with your real domain(s). Fill in Dropbox credentials if you want sync. |
