# PlotWise — Server-Hosted Books + Mobile Reading Design

> Replaces Section 3.3 (EPUB Storage) of `PLOTWISE_MULTIUSER_DESIGN.md`  
> Covers: how EPUBs live on the server, how the reader gets them, mobile flow

---

## 1. The Clarified Vision

```
Admin (you) uploads ORV.epub to the server
       ↓
Server runs ingestion → builds character graph in PostgreSQL
       ↓
ORV is now "available" on PlotWise
       ↓
You open PlotWise on your phone
→ Add ORV to your library
→ PlotWise fetches the EPUB from server → epub.js renders it in your browser
→ You read chapter 47
→ You ask "who is Dokja?" → client sends query to server → server gates at ch.47 → answer
```

No file imports. No downloads. No local management. Open the PWA, pick a book, read.

---

## 2. Why This Is Actually Simpler Than Model A

Model A (users bring their own EPUB) required:

- Users to source their own EPUB file
- A manual import step into Flow
- A sync bridge between Flow's IndexedDB and the server's chapter tracker

This model (server-hosted EPUBs) requires:

- Admin uploads once
- Frontend fetches EPUB from server on demand
- Server is the single source of truth for everything

The only addition is storing EPUB files on the server and serving them through a protected endpoint. Everything else in the architecture is the same or simpler.

---

## 3. What Changes in the Architecture

### 3.1 EPUB Storage on the Server

EPUBs are stored as files on the server. Two options depending on scale:

**Option A — Server Disk (for personal/small scale)**

```
/data/
  epubs/
    lord-of-the-mysteries.epub
    omniscient-readers-viewpoint.epub
    the-beginning-after-the-end.epub
  books/
    lord-of-the-mysteries/
      plotwise.db (or PostgreSQL)
      lotm.faiss
```

Fine for personal use and a handful of books. Costs nothing extra.

**Option B — Cloudflare R2 (for deployed, persistent storage)**

```
R2 bucket: plotwise-books/
  epubs/lord-of-the-mysteries.epub
  epubs/omniscient-readers-viewpoint.epub
```

R2 is free for up to 10GB storage and free egress. LotM is ~20MB. You can store 500 webnovels for free. Use this when you move off a local machine.

**For v1: start with server disk. Switch to R2 when you deploy.**

### 3.2 New Endpoint: Serve EPUB + Auth Model

**The PlotWise Books Flow:**

1. User clicks "PlotWise books" button in Flow UI
2. If not logged in → show login screen (email + password)
3. User logs in → server issues JWT token
4. Token stored in browser (localStorage or cookie)
5. User sees available books (LotM, ORV, TBATE)
6. User clicks "Add ORV to library" → `POST /api/library/orv`
7. ORV added to user's server-synced library
8. User opens ORV → fetches from `/api/books/orv/epub`
9. Progress synced to server on every chapter change
10. Later: open on laptop → same JWT (or re-login) → resume at same chapter

**Auth Mechanism: JWT (JSON Web Token)**

```
GET /api/books/{book_id}/epub
    Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
    Response: EPUB file as application/epub+zip
              (or a short-lived pre-signed URL if using R2)
```

Auth is required. Server validates:
1. JWT is valid (signed with server secret)
2. User ID extracted from JWT
3. User has this book in their library (`user_libraries` table)

Only then: serve the EPUB file.

```python
@router.get("/books/{book_id}/epub")
async def serve_epub(book_id: str, user=Depends(get_current_user), db=Depends(get_db)):
    # get_current_user: extracts user_id from JWT
    # If JWT missing or invalid: raises 401 Unauthorized
    
    # verify user has this book in their library
    library_entry = db.execute("""
        SELECT 1 FROM user_libraries
        WHERE user_id = ? AND book_id = ?
    """, [user.id, book_id]).fetchone()

    if not library_entry:
        raise HTTPException(403, "Book not in your library")

    epub_path = get_epub_path(book_id)  # from books table or config
    return FileResponse(epub_path, media_type="application/epub+zip")
```

**Why JWT?**
- **Cross-device:** One login, token works on phone + laptop + tablet
- **Stateless:** Server doesn't need session store, just validates signature
- **PWA-friendly:** Works with offline and installable apps
- **Device-agnostic:** Progress synced via `user_id` in JWT, not device-specific state
### 3.3 Frontend: Fetch EPUB from Server, Pass to epub.js

**IMPORTANT: Flow's local library management is NOT stripped. Both workflows coexist.**

You can simultaneously have:
- Local EPUBs you imported (read from IndexedDB, progress local-only)
- Server EPUBs from PlotWise collection (read from server, progress synced)

No conflict, no interference.

---

**How server EPUBs work in the reader:**

Instead of Flow loading from local IndexedDB, PlotWise fetches the EPUB from the server and passes it to epub.js as an **ArrayBuffer** (binary data in memory):

```javascript
// Instead of: ePub("/local/file.epub")
// Do this for server books:

async function loadBookFromServer(bookId, token) {
  // Step 1: Fetch EPUB from server
  const response = await fetch(`/api/books/${bookId}/epub`, {
    headers: { Authorization: `Bearer ${token}` }
  });

  // Step 2: Convert response to binary (ArrayBuffer)
  const arrayBuffer = await response.arrayBuffer();
  // arrayBuffer = entire EPUB file (20MB) in raw bytes, stored in memory

  // Step 3: Pass to epub.js — it doesn't care where bytes came from
  const book = ePub(arrayBuffer);
  const rendition = book.renderTo("reader-container", {
    width: "100%",
    height: "100%"
  });

  // Step 4: Render
  rendition.display();

  // Step 5: Track chapter changes → sync to server
  rendition.on("relocated", async (location) => {
    const chapterIndex = location.start.index;
    await fetch('/api/reader/progress', {
      method: 'POST',
      headers: { Authorization: `Bearer ${token}` },
      body: JSON.stringify({ book_id: bookId, chapter_index: chapterIndex })
    });
  });
}
```

**What is ArrayBuffer?**

ArrayBuffer is just **binary data in memory**. When you fetch an EPUB from the server:

```
Server sends: [binary EPUB file bytes]
         ↓
fetch().arrayBuffer() converts to: ArrayBuffer (raw bytes in memory)
         ↓
ePub() reads: the raw bytes, doesn't care if they came from disk, network, or anywhere
         ↓
rendition.display() renders: the book in the browser
```

**Why ArrayBuffer instead of downloading a file?**

| Approach | UX | Storage | Offline |
|----------|----|---------| --------|
| Traditional download | Slow (save dialog, file on disk) | Uses device disk space | Works after download |
| ArrayBuffer streaming | Fast (instant in reader) | Memory only (~20MB) | Requires server |
| ArrayBuffer + Service Worker cache | Fast (after 1st load) | IndexedDB cache (~20MB per book) | Yes (future) |

For v1: use ArrayBuffer streaming. The EPUB loads instantly, stays in memory while you read, no disk usage.

**Could we cache it locally for offline?** Yes, but that's a future enhancement. Add Service Worker caching after v1 ships.

---

**What stays unchanged in Flow:**

- Local import/export still works
- Local EPUB reading still works
- IndexedDB still stores local books
- Offline mode still works (for local EPUBs)
- Flow's UI doesn't need changes
- Highlights and annotations still work locally

**What Flow adds (minimal):**

- A "PlotWise books" button in the UI
- When clicked: login → show server book collection
- Option to add to library, which syncs reading progress to server

=== wait why are we stripping flows local librar management ? i thought we are gonna work with that?? NO ? Will that no t work ? what is this araybuffer
### 3.4 Books Table Update

```sql
-- Add storage location to books table
ALTER TABLE books ADD COLUMN epub_storage_path TEXT;
-- e.g., "/data/epubs/lord-of-the-mysteries.epub"
-- or "r2://plotwise-books/epubs/lord-of-the-mysteries.epub"

ALTER TABLE books ADD COLUMN epub_size_bytes INTEGER;
ALTER TABLE books ADD COLUMN cover_image_path TEXT;  -- for library UI
```

---

## 4. User Workflow: Accessing PlotWise Books (Confirmed Flow)

**The confirmed user flow:**

1. **Open Flow PWA** on phone or laptop
2. **Click "PlotWise books" button** in UI
3. **Not logged in?** → Show login form (email + password)
4. **Log in** → Server issues JWT token, stores in browser
5. **Browse available books** — see LotM, ORV, TBATE (all pre-ingested)
6. **Add ORV to library** → `POST /api/library/orv` with JWT
7. **ORV appears in separate "PlotWise Library"** section (distinct from local imports)
8. **Open ORV** → fetches EPUB from server as ArrayBuffer → renders in epub.js
9. **Read chapter 47** → progress synced to server on every chapter change
10. **Open Flow on laptop later** → same JWT (or re-login) → **resume at chapter 47**
11. **PlotWise library is the same across all devices** — email is the key, not device

**Key points:**
- ✅ Separate library per user (synced across devices)
- ✅ Device-agnostic (phone ↔ laptop ↔ tablet all share same progress)
- ✅ Local imports still work independently in the same session
- ✅ No conflict between local and server books

---

## 5. Admin Workflow: Adding a Book (Unchanged from Original)

This is the full flow for adding a new book to the platform:

```
1. Admin uploads EPUB via a protected admin endpoint
   POST /api/admin/books/upload
   Body: multipart form — epub file + metadata (title, author)
   → file saved to /data/epubs/{book_id}.epub
   → books row created with status = 'pending'

2. Ingestion triggered (async background task)
   POST /api/admin/books/{book_id}/ingest
   → Pass 1 runs (per-chapter extraction)
   → Pass 2 runs (identity merge resolution)
   → FAISS index built
   → books.ingestion_status = 'complete'

3. Book is now available
   books.is_public = true  (admin sets this when ready)
   → appears in GET /api/books/available
   → all users can add to library and read
```

All admin endpoints require `role = 'admin'` in the JWT. Regular users never touch these endpoints.

---

## 6. Mobile Reading Flow (End to End)

This is what actually happens when you open PlotWise on your phone and read ORV:

```
1. Open PlotWise PWA on phone (installed from browser)

2. Click "PlotWise books" button
   → Not logged in yet? Show login form

3. Login with your account (email + password)
   → JWT issued and stored in localStorage

4. Browse available books
   GET /api/books/available
   → See: LotM, ORV, TBATE with cover images and chapter counts

5. Add ORV to library
   POST /api/library/omniscient-readers-viewpoint
   → Creates user_libraries row with your user_id

6. ORV appears in your "PlotWise Library" (separate from local imports)

7. Open ORV
   GET /api/books/omniscient-readers-viewpoint/epub
   → Fetches EPUB as ArrayBuffer from server
   → epub.js renders it in browser
   → Progress loaded from server (reader_progress) → jumps to last chapter

8. Read chapter 47
   → On chapter change: POST /api/reader/progress { chapter_index: 47 }
   → Server updates reader_progress for your user_id

9. Ask "who is Han Sooyoung?"
   POST /api/query/resolve { book_id: "orv", query: "who is Han Sooyoung?" }
   → Server reads your reader_progress → current_chapter = 47
   → SpoilerGate filters to chapter ≤ 47
   → Answer returned in ~800ms via Groq

10. Close phone, open on laptop 1 hour later
    → Same JWT token still valid (or re-login if cleared)
    → Login/session automatic
    → Open ORV → server reads YOUR progress → resumes at chapter 47
    → Full state preserved across devices, zero setup
```

**Result:** No file imports needed. No sync setup. Just login once per device, pick a book, read. Progress always synced.

## 7. Deployment Requirements

```
┌─────────────────────────────────────────────────────────┐
│  Vercel (free)                                          │
│  Next.js PWA — served globally, installable on mobile   │
└─────────────────────────────┬───────────────────────────┘
                              │ HTTPS API calls
┌─────────────────────────────▼───────────────────────────┐
│  Railway or Render (~$7/mo)                             │
│  FastAPI — handles auth, queries, progress, epub serve  │
│  /data/epubs/ — EPUB files on persistent disk           │
│  /data/faiss/ — FAISS indexes                           │
└─────────────────────────────┬───────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────┐
│  Neon or Railway PostgreSQL (free tier available)       │
│  All tables from multi-user schema                      │
└─────────────────────────────────────────────────────────┘
```

**Total cost: ~$7–10/month** for a fully working multi-device reading companion.

### When You Scale Up

If you add many books or users, swap server disk for R2:

```python
# Cloudflare R2 — boto3 compatible
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="https://{account_id}.r2.cloudflarestorage.com",
    aws_access_key_id=R2_ACCESS_KEY,
    aws_secret_access_key=R2_SECRET_KEY
)

def get_epub_presigned_url(book_id: str) -> str:
    return s3.generate_presigned_url(
        "get_object",
        Params={"Bucket": "plotwise-books", "Key": f"epubs/{book_id}.epub"},
        ExpiresIn=300  # 5 min URL, frontend fetches directly
    )
```

With presigned URLs, the EPUB bytes flow directly from R2 → user's browser, bypassing your server entirely. Faster, cheaper, scales to any number of users.

---

## 7. Copyright Note

Storing and serving EPUB files for webnovels like LotM and ORV is legally grey. Most are fan translations not officially licensed in English. For a private personal tool this is a practical non-issue. If PlotWise ever becomes public-facing:

- Store only EPUBs you have rights to, or
- Return to Model A (users bring their own EPUB, server stores only the extracted character graph)
- The narrative intelligence layer (character graph, events, relationships) is your original work product and is unambiguously yours

The architecture supports switching between these two models by changing only the epub-serving endpoint. The entire AI layer is unaffected.

---

## 8. Complete Change Summary

Changes from `PLOTWISE_MULTIUSER_DESIGN.md`:

| What                 | Change                                                                        |
| -------------------- | ----------------------------------------------------------------------------- |
| EPUB storage         | Server disk → `/data/epubs/` or Cloudflare R2                                 |
| `books` table        | Add `epub_storage_path`, `cover_image_path`, `epub_size_bytes`                |
| New endpoint         | `GET /api/books/{book_id}/epub` — JWT auth-protected file serve               |
| New endpoint         | `POST /api/admin/books/upload` — admin only, JWT required                     |
| Frontend reader      | Fetch server EPUB as ArrayBuffer → pass to epub.js                           |
| Flow's local storage | **Completely preserved.** Local EPUBs still work independently                |
| PlotWise library     | **Separate from Flow's local library.** Both coexist in same session           |
| User login           | Required for server-hosted books (enables cross-device sync)                  |
| Progress tracking    | Server-synced for PlotWise books; local for imported EPUBs                    |
| Offline support      | Yes for local EPUBs; future: cache server EPUBs in Service Worker             |
| Deployment           | Required for cross-device (Railway + Vercel + Neon)                          |

**IMPORTANT CORRECTIONS:**
- ✅ Flow's local library management is **NOT stripped** — it continues to work as-is
- ✅ PlotWise books appear in a **separate library section**, not replacing Flow's
- ✅ Both workflows (local + server) work simultaneously with no conflict

Everything else — ingestion pipeline, character graph, identity resolution, spoiler gate, all query endpoints — unchanged.