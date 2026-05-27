# PlotWise Design Clarifications — Chat Log

**Date:** 27 May 2026  
**Context:** Review of `3_current-plotwise-server-epub-design.md`

---

## User Questions

### 3.2 — What is the auth here??

> From section 3.2, endpoint `GET /api/books/{book_id}/epub` — what auth mechanism is being used?

### 3.3 — Wait, why are we stripping Flow's local library management? I thought we are gonna work with that?? NO? Will that NOT work? What is this ArrayBuffer?

> The document says "strip out the local library management" and replace it with server-hosted books. But the intent is:
> - Flow's local management should continue to work as-is
> - PlotWise adds a server layer that's a *collection* you can import to your library
> - These should coexist, not replace each other
> - What does ArrayBuffer mean in this context?

### 4 — "Flow will work as is, this will be like a collection available to import to your library"

> Confirming: both the local Flow workflow AND the PlotWise server workflow should work side-by-side.

---

## Answers

### 3.2 — The Auth Mechanism

The auth is **JWT (Bearer token)**. When you call the EPUB endpoint:

```javascript
GET /api/books/{book_id}/epub

Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The server extracts your `user_id` from the JWT, then checks two things:

1. **Is this token valid?** (signed with the server's secret key)
2. **Does this user have this book in their library?** (checks `user_libraries` table for `user_id` + `book_id`)

If both pass → serve the EPUB file. If not → `403 Forbidden`.

**Why JWT instead of sessions?**
- PWAs work across devices and offline
- Sessions tied to single server are fragile
- JWT is self-contained: server can verify it without a session store
- Works for multi-device reading (login once, token travels with you)

**Example implementation:**

```python
@router.get("/books/{book_id}/epub")
async def serve_epub(book_id: str, user=Depends(get_current_user), db=Depends(get_db)):
    # get_current_user extracts user_id from JWT
    # If JWT is invalid or missing, raises 401 Unauthorized
    
    # Check if user has this book in their library
    library_entry = db.execute("""
        SELECT 1 FROM user_libraries
        WHERE user_id = ? AND book_id = ?
    """, [user.id, book_id]).fetchone()

    if not library_entry:
        raise HTTPException(403, "Book not in your library")

    epub_path = get_epub_path(book_id)
    return FileResponse(epub_path, media_type="application/epub+zip")
```

---

### 3.3 — Flow's Local Library ≠ Stripping. Coexistence.

**The document is misleading. Here's the correction:**

#### What the document says (wrong):
> "Flow's local storage features (IndexedDB import, export, offline cache) are no longer needed for the primary use case. The Flow fork becomes lighter — keep epub.js's rendering and pagination, strip out the local library management."

#### What should happen (correct):
Flow's local management stays **completely intact**. PlotWise adds a *parallel* layer, not a replacement.

```
Flow's IndexedDB (unchanged)
├─ Stores EPUBs you imported locally
├─ Stores reading position for local EPUBs
├─ Stores highlights, annotations, bookmarks
└─ Works offline

PlotWise Server Layer (new, additive)
├─ Maintains a collection of pre-ingested books (ORV, LotM, TBATE)
├─ You can add these books to your "PlotWise library" while logged in
├─ These books are read from the server (as ArrayBuffer)
├─ Reading progress synced to server for spoiler gating
└─ Works when online

Both coexist. A Flow session can simultaneously have:
✓ Local EPUB imported by you (reads from IndexedDB, progress is local-only)
✓ Server EPUB from PlotWise collection (reads from server, progress is synced)
✓ No conflict, no interference
```

**The key insight:** PlotWise doesn't *replace* Flow's library. It *extends* it with a pre-curated collection. Flow's local import/export/offline still work exactly as they do now.

---

#### What is ArrayBuffer?

ArrayBuffer is **binary data in memory**. When the server sends you an EPUB:

```javascript
// Step 1: Fetch EPUB from server
const response = await fetch('/api/books/orv/epub', {
  headers: { Authorization: `Bearer ${token}` }
});

// Step 2: Convert response to binary
const arrayBuffer = await response.arrayBuffer();
// arrayBuffer = entire EPUB file (20MB) in raw binary form
// Think: File object bytes, but in memory instead of on disk

// Step 3: Pass to epub.js
const book = ePub(arrayBuffer);
// epub.js doesn't care where the bytes came from
// Could be: local disk, IndexedDB, HTTP fetch, or anywhere
// It just needs the binary blob

// Step 4: Render
const rendition = book.renderTo("reader-container");
rendition.display();
```

**Why ArrayBuffer instead of downloading a file?**

| Approach | UX | Storage | Offline |
|----------|----|---------| --------|
| Download file | Slow (download dialog, save to disk) | Uses local disk space | Works after download |
| ArrayBuffer (streaming) | Fast (instant in reader) | Stays in memory | No (requires server) |
| ArrayBuffer + Service Worker cache | Fast (after first load) | IndexedDB cache (~20MB per book) | Yes (future feature) |

For v1: Use ArrayBuffer streaming. The EPUB streams from server → memory → epub.js renders it immediately. No download, no storage burden on your device.

**Could we cache it locally?** Yes, with a Service Worker:

```javascript
// Service Worker (future enhancement)
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/books/') && event.request.url.includes('/epub')) {
    event.respondWith(
      caches.open('epubs-v1').then((cache) => {
        return cache.match(event.request).then((response) => {
          // If cached, serve from cache (offline works)
          if (response) return response;
          
          // Otherwise fetch from server and cache for next time
          return fetch(event.request).then((freshResponse) => {
            cache.put(event.request, freshResponse.clone());
            return freshResponse;
          });
        });
      })
    );
  }
});
```

But that's a one-liner addition later. v1 doesn't need it.

---

### 4 — Both Workflows Coexist

**YES.** Exactly as you described.

#### The Workflow

**Scenario 1: Local EPUB**
```
1. You import a local EPUB into Flow (or drag-drop into browser)
2. Flow stores it in IndexedDB
3. You read it locally
4. No server involved
5. PlotWise layer doesn't activate (no JWT, no server progress sync)
6. Works offline
```

**Scenario 2: Server EPUB from PlotWise Collection**
```
1. You open PlotWise and log in (JWT issued)
2. Browse available books: LotM, ORV, TBATE (all pre-ingested)
3. Click "Add ORV to library" → POST /api/library/orv
4. Open ORV → fetches EPUB as ArrayBuffer from server
5. You read chapter 47 → progress synced to server via /api/reader/progress
6. You ask "Who is Dokja?" → server queries with chapter gate at 47
7. Requires internet, but instant load and automatic progress sync
```

**Scenario 3: Mixed Session**
```
1. Flow open on your phone with Mix of:
   - Local EPUB imported yesterday (reads from IndexedDB)
   - Server EPUB you added today (reads from server)
2. Both appear in Flow's library UI
3. Both work independently
4. No conflict
```

#### What Doesn't Change

- Flow's library UI (no modifications needed)
- Flow's reading experience (epub.js rendering untouched)
- Flow's import/export (still works locally)
- Flow's highlight system (still local to IndexedDB)

#### What Adds to Flow

- A connection to PlotWise backend (optional, requires login)
- Server progress tracking for PlotWise books (syncs when you navigate chapters)
- Integration point: when reading a server book, on chapter change → `POST /api/reader/progress`

---

## Correction Needed in Document

Section 3.3 should be rewritten to clarify:

**Current (misleading):**
> "Flow's local storage features (IndexedDB import, export, offline cache) are no longer needed for the primary use case. The Flow fork becomes lighter — keep epub.js's rendering and pagination, strip out the local library management."

**Should be:**
> "Flow's local management stays fully intact. You can import EPUBs locally and read them entirely offline with Flow's standard features.
>
> PlotWise *adds* a parallel collection of server-hosted books. When logged in, you can add these pre-ingested books (LotM, ORV, etc.) to your library. The Flow reader doesn't need changes — it fetches the EPUB as an ArrayBuffer and passes it to epub.js exactly as it would any other file.
>
> Both workflows coexist: local EPUBs work as today, server EPUBs stream from the backend. A Flow session can have both simultaneously."

---

## Summary Table

| Aspect | Local EPUBs (Today's Flow) | Server EPUBs (PlotWise New) |
|--------|---------------------------|---------------------------|
| Storage | Your device (IndexedDB or disk) | PlotWise server |
| Auth | None needed | JWT required |
| Progress | Local IndexedDB | Synced to server |
| Spoiler gate | Not available | Active (server-enforced) |
| Queries ("Who is X?") | Not available | Active (server-gated) |
| Offline | Yes | No (after first fetch: yes if cached) |
| Library UI | Flow's existing UI | Same UI, no changes |
| Coexistence | ✅ Both work side-by-side |

---

## Additional Q&A: Login Flow Confirmation & Auth Model Tradeoff

### User Confirmation Question

> OK, so to confirm: You click on "PlotWise books" button → you have to sign in → then get access to the books which you can add to a separate library which will be saved irrespective of device?

**Answer: YES, exactly that.**

Here's the flow step-by-step:

```
┌─────────────────────────────────────────────────────────┐
│ You open Flow PWA on phone/laptop                       │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ Two options in Flow UI:        │
        │ 1. "Import local EPUB"         │
        │ 2. "Browse PlotWise books"     │
        └──────────────┬─────────────────┘
                       │
              [Click: Browse PlotWise books]
                       │
        ┌──────────────▼─────────────────┐
        │ Not logged in?                 │
        │ → Show login/signup form       │
        └──────────────┬─────────────────┘
                       │
     [Enter email & password, click Login]
                       │
        ┌──────────────▼─────────────────┐
        │ Server validates creds        │
        │ Issues JWT token              │
        │ Token stored in browser       │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ Show available books:         │
        │ • Lord of the Mysteries       │
        │ • Omniscient Reader's View    │
        │ • The Beginning After End     │
        └──────────────┬─────────────────┘
                       │
        [Click: "Add ORV to library"]
                       │
        ┌──────────────▼─────────────────┐
        │ POST /api/library/orv          │
        │ + Authorization: Bearer JWT    │
        │ → Creates user_libraries row   │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ ORV now in "My Library"        │
        │ (separate from local EPUBs)    │
        └──────────────┬─────────────────┘
                       │
        [Click: Open ORV]
                       │
        ┌──────────────▼─────────────────┐
        │ GET /api/books/orv/epub        │
        │ + Authorization: Bearer JWT    │
        │ → Fetches ArrayBuffer from     │
        │   server, epub.js renders      │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ You read chapter 47            │
        │ Progress synced to server      │
        │ (independent per-device)       │
        └──────────────┬─────────────────┘
                       │
   [Later: Open Flow on laptop]
                       │
        ┌──────────────▼─────────────────┐
        │ Same JWT still valid           │
        │ (or auto-login if stored)      │
        │ Open ORV on laptop             │
        └──────────────┬─────────────────┘
                       │
        ┌──────────────▼─────────────────┐
        │ Server reads reader_progress   │
        │ → Resumes at chapter 47        │
        │   (same chapter from phone!)   │
        └──────────────────────────────────┘
```

**Key points:**

1. ✅ **Separate library** — Your PlotWise books appear in a different section than local imports
2. ✅ **Device-independent** — Progress saved on server, same chapter on all devices
3. ✅ **Cross-device** — Log in on phone → library synced → log in on laptop → same library, same progress

---

### Tradeoff Question: Advantages of Login vs No-Login (Anonymous) Model?

> Any advantage of keeping a user login? Like, can't we just make it available without login auth, etc., but tracking across multiple devices won't be possible? That's fine.

**This is a great question. Let's compare:**

#### Option A: With Login (Current Design)

**Advantages:**
- ✅ Progress synced across devices (phone ↔ laptop ↔ tablet)
- ✅ Each user has their own separate library
- ✅ Server knows who you are → can enforce permissions (your books only)
- ✅ Enables future features: shared reading sessions, book recommendations per user, reading streaks
- ✅ Can add user-specific features: bookmarks, reading history, reading lists
- ✅ Can moderate: ban spam users, handle copyright complaints per-user

**Disadvantages:**
- ❌ Friction: one extra step (login) before reading
- ❌ Privacy: server knows your email and reading habits
- ❌ Credential management: passwords, reset flows, account recovery
- ❌ More server complexity: auth, session management, user tables

#### Option B: No Login (Anonymous)

**How it would work:**
```
1. No auth endpoint needed
2. User visits PlotWise
3. GET /api/books/available → returns all books, no auth
4. User clicks "Add ORV"
5. → Create anonymous session (localStorage key = random UUID)
6. user_library has anon_session_id instead of user_id
7. On chapter change: POST /api/reader/progress { session_id, chapter }
8. Server tracks that session_id's progress
```

**Advantages:**
- ✅ Zero friction: no login, just click and read
- ✅ Better privacy: server doesn't know who you are
- ✅ Simpler code: no password hashing, no JWT, no auth endpoints
- ✅ Lower GDPR burden (no personal data stored)

**Disadvantages:**
- ❌ **No cross-device sync** — If you open on phone, then laptop, they're different sessions
  - Phone: session_id = abc123 → progress at chapter 47
  - Laptop: session_id = xyz789 → progress at chapter 0 (new session)
  - No way to "resume where you left off" across devices
- ❌ **Library lost on browser clear** — If you clear localStorage, your "My Library" disappears
- ❌ **No way to recover** — Can't log in later and get your old library back
- ❌ **Can't implement user features** — no bookmarks per-user, reading history, recommendations

---

#### Which is Better? Depends on Your Goals

| Goal | Better Approach |
|------|---|
| Personal reading companion (single device) | Option B (anonymous) — simpler |
| Multi-device reading (phone + laptop + tablet) | Option A (login) — essential |
| Privacy-first (don't track users) | Option B (anonymous) — but you lose sync |
| Casual reading (don't want account) | Option B (anonymous) — low friction |
| Social features later (reading groups, etc.) | Option A (login) — easier to add |

---

#### Hybrid: "Light" Login (Email-only, No Password)

If you want most benefits of login with less friction, you could do **email-only auth**:

```
POST /api/auth/request-code
Body: { email: "you@example.com" }
→ Server sends 6-digit code to email

POST /api/auth/verify-code
Body: { email, code }
→ Server issues JWT (no password needed)
```

**Advantages:**
- ✅ Zero password management (no hashing, resets, etc.)
- ✅ Cross-device sync (still have login)
- ✅ Still get user table (simpler than no-login)
- ✅ Lower friction than password login (copy code from email)

**Disadvantages:**
- ❌ Users need to check email every time (annoying)
- ❌ Email spam if codes are sent repeatedly

---

#### Recommendation

For **PlotWise v1**, I'd suggest: **Option A (JWT login with password)**

**Reasoning:**
1. The core value of PlotWise is multi-device reading with spoiler gating
2. If someone's reading on phone at chapter 47, and opens on laptop, they expect to resume at 47
3. Without login, that's not possible
4. Login is a one-time friction cost; you only do it once per device
5. The future (bookmarks, reading history, sharing) becomes possible

**But if you want to defer complexity:** Start with **Option B (anonymous)**, ship it, gather feedback. Add Option A later:

- You can migrate from anonymous to login by just adding a `user_id` column and a migration script
- The schema change is tiny
- The ingestion pipeline and query engine don't care

---

#### Final Answer to Your Question

> "Can't we just make it available without login?"

**Technically: Yes, absolutely.**

**Practically:** You lose the #1 use case — resuming on a different device at the same chapter. But if you're building for single-device use (just mobile, or just laptop), anonymous is fine.

**My take:** Login with email is worth the ~30 seconds of friction because it unlocks cross-device reading and future features. But if you want to ship faster and defer this decision, go anonymous first.

---

_This Q&A clarifies the design choice: login enables device-agnostic library and progress, but costs friction and privacy. Anonymous is simpler but ties reading sessions to devices._


- new qn
do same as before update chat in chat-vsc.md
then is it better to simply remove the flow's no login feature ? is that recommeneded? or like make it so plotwise - is the ai companion included reader , & for your own books - use the default local epub reader - flow which has no ai feature

claude's reply

--
Good question — this is actually a core architecture decision. Let me lay it out clearly.

You're essentially choosing between two models:

---

**Model A — One app, two modes** *(current design)*
PlotWise (forked from Flow) handles both: local EPUBs (no login) and server books (login). Both coexist.

**Model B — Clean split** *(what you're now proposing)*
- **PlotWise** = AI companion reader, server-hosted curated books, always requires login
- **Flow** = untouched, for your own local EPUBs, no AI, no login

---

### Model B is the right call. Here's why:

**1. The mental models don't mix well**

"I want to read my own book" and "I want to read a curated book with AI help" are fundamentally different intents. Forcing them into one UI creates confusion — which library am I looking at? Why does one section need login and the other doesn't? That's cognitive overhead that serves nobody.

**2. PlotWise becomes genuinely simpler to build**

If PlotWise drops local EPUB management entirely:
- No IndexedDB logic
- No "two library sections" UI problem
- No "which mode am I in" state management
- Login is always required, always assumed — auth is never optional

The whole "they coexist with no conflict" language in your doc is a sign of accidental complexity. If they truly had nothing to do with each other, you wouldn't need to document that they don't conflict.

**3. Flow stays completely untouched**

No fork maintenance burden for local reading. Flow is Flow. You only maintain PlotWise for the AI + server books use case.

**4. The scope is cleaner for a GSoC project**

PlotWise's value proposition is sharp: *curated books + AI companion + cross-device sync*. That's a complete, defensible feature. Local EPUB reading is Flow's job.

---

### The one thing to consider

If someone ever asks *"can I use PlotWise AI on my own EPUB?"* — with Model B the answer is no. That's a feature gap. But for v1 (and honestly for a personal tool), that's fine. You can revisit it as a future feature: "upload your own EPUB to PlotWise" — which would just mean adding an admin-style upload flow for regular users.

---

### Recommendation

Go with **Model B**. Update the design doc to reflect:

- PlotWise has **no local EPUB management** — that's explicitly out of scope
- Login is **always required** — no anonymous mode
- Flow is **referenced as a separate tool** for personal EPUBs, not integrated
- The "PlotWise books button in Flow UI" idea goes away — PlotWise is its own standalone PWA

This also makes your architecture diagram cleaner: PlotWise ↔ server ↔ PostgreSQL. No IndexedDB, no hybrid auth states, no dual-library UX.


-- create doc 4 - final 
ok now create the final design doc with model B , file starting with1_ was the initial then 2 then 3 have the final changes 
merge these 3 into one, basically update teh changes & create the final system design doc in md format

doc 4 created