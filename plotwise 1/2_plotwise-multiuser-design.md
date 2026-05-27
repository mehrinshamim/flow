# PlotWise — Multi-User Platform Design

> Companion to: `PLOTWISE_SYSTEM_DESIGN.md`  
> Covers: how Flow stores data, current vs new vision, schema delta, deployment changes

---

## 1. How Flow Actually Stores Data (Important Context First)

Before anything else — Flow has **zero server-side user management**.

Flow stores all data locally in the browser using IndexedDB. It requires no registration and has no tracking. Books, reading positions, highlights, and library organization are all stored per-browser, per-device.

What this means practically:

|What Flow tracks|Where|
|---|---|
|EPUB file content|IndexedDB (browser)|
|Reading position (CFI)|IndexedDB (browser)|
|Library / bookshelf|IndexedDB (browser)|
|Highlights & annotations|IndexedDB (browser)|
|**User identity**|**Does not exist**|
|**Cross-device sync**|**Only via Dropbox (optional)**|

**Flow has no concept of users.** If you open Flow on your phone and your laptop, they are two entirely separate reading environments. There is no login.

This is important because the PlotWise AI layer must introduce the concepts Flow deliberately avoids: user identity, server-side progress sync, and shared state.

---

## 2. Current Vision vs New Vision

### Current Vision (from `PLOTWISE_SYSTEM_DESIGN.md`)

```
One person. One machine. Personal use.

- You upload your own EPUBs
- Ingestion runs locally on your machine
- SQLite lives on your machine
- FastAPI runs on localhost
- Flow reads from local IndexedDB
- No other users exist
- No auth needed
```

This is a **local-first personal tool**. It works perfectly for personal use. The schema in the main design doc reflects this — no `user_id` anywhere, single `reader_progress` row per book.

### New Vision (This Document)

```
Multiple users. Books processed once. Shared intelligence layer.

- Admin (you) uploads + processes ORV, LotM, TBATE once
- The character graph, identity merges, events, factions → stored globally
- Any user can add these books to their own library
- Each user has their own reading progress, independent of others
- Users read via their own local EPUB copy (stored in Flow's IndexedDB)
- The AI query layer is server-side and user-aware
```

This is a **shared platform with a global narrative intelligence layer and per-user reading state**.

### The Key Insight: What Is Global vs What Is Per-User

This is the most important design decision in the new model.

```
GLOBAL (processed once, shared by all users)     PER-USER (separate per person)
────────────────────────────────────────         ─────────────────────────────
identity_nodes                                   reader_progress
identity_merges                                  user_libraries
character_facts                                  query_history
relationships                                    bookmarks (future)
factions                                         preferences (future)
faction_memberships
events
chapter_summaries
books (metadata)
```

The narrative intelligence — characters, relationships, events, identity reveals — is a **global asset**. It is extracted once from the book and never duplicated. Every user benefits from the same processed graph.

Only the **reader's relationship with the book** is personal: where they are, what they've asked, what they've added to their library.

---

## 3. What Needs to Change

### 3.1 Database: SQLite → PostgreSQL

SQLite works for one person. For multiple concurrent users it becomes a bottleneck — SQLite serializes all writes. PostgreSQL handles concurrent connections properly.

> **If you are early-stage and want to keep SQLite:** Use WAL mode (`PRAGMA journal_mode=WAL`). SQLite WAL supports concurrent reads with serialized writes. Since most writes are just `UPDATE reader_progress`, it is viable for a small user base (under 50 users). Migrate to PostgreSQL when it becomes a real bottleneck.

### 3.2 Schema: Add User Tables + Modify Progress

Only three things actually change in the schema:

1. Add `users` table
2. Add `user_libraries` table
3. Add `user_id` to `reader_progress`

Everything else (all book intelligence tables) is **unchanged from the original design doc**.

### 3.3 EPUB Storage: Local → Two Options

In the current vision, EPUBs are files on your local disk.

In the multi-user vision, there are two models:

**Model A — Users Bring Their Own EPUB (Recommended)**

```
Admin processes the book → stores character graph in DB
Users upload their own copy of the EPUB → stored in their local browser (IndexedDB)
The server never holds EPUB files
```

- No copyright liability (server stores extracted facts, not book content)
- Users must source their own EPUB
- Cleanest legally

**Model B — Centralized EPUB Storage**

```
Admin uploads EPUB → stored in S3/R2 → served to all users on request
```

- Convenient but legally grey for licensed novels
- Most webnovels (LotM, ORV) are fan translations — unclear legal standing
- Not recommended unless you own the rights

**Recommendation: Model A.** The value PlotWise provides is the narrative intelligence layer, not EPUB hosting. The server stores only what it extracted. Users source EPUBs themselves.

### 3.4 Auth: None → JWT

Flow has no auth. PlotWise needs to know which user is asking which query at which chapter. JWT-based auth is sufficient.

Simple flow:

```
User registers (email + password) → server issues JWT
JWT sent in Authorization header on every API request
Backend extracts user_id from JWT → uses it for reader_progress lookups
```

No OAuth required for v1. Add Google/GitHub login later if needed.

### 3.5 Reading Progress: Flow's IndexedDB → Synced to Server

Flow tracks reading position locally as a CFI (Canonical Fragment Identifier — epub.js's internal position format). For PlotWise's spoiler gate, we need chapter index on the server.

The sync bridge:

```javascript
// In Flow fork — intercept chapter change
reader.on('relocated', async (location) => {
  const chapterIndex = location.start.index;

  // 1. Flow still updates IndexedDB as normal (don't break Flow's local state)
  // 2. ALSO sync to PlotWise server
  await fetch('/api/reader/progress', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${getJWT()}` },
    body: JSON.stringify({ book_id: currentBookId, chapter_index: chapterIndex })
  });
});
```

Flow continues to own local reading state. PlotWise server tracks chapter index for spoiler gating. Both coexist.

---

## 4. New Schema (Delta Only)

These are the **additions and modifications** to the original schema. Everything not listed here is unchanged.

```sql
-- ─────────────────────────────────────────────
-- NEW: USERS
-- ─────────────────────────────────────────────
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT UNIQUE NOT NULL,
    username      TEXT UNIQUE,
    password_hash TEXT NOT NULL,          -- bcrypt
    role          TEXT DEFAULT 'user',    -- 'user' | 'admin'
    created_at    TIMESTAMP DEFAULT NOW(),
    last_active   TIMESTAMP
);

-- ─────────────────────────────────────────────
-- NEW: USER LIBRARIES
-- Which books a user has added to their library.
-- Books exist globally; this is the user <-> book relationship.
-- ─────────────────────────────────────────────
CREATE TABLE user_libraries (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    book_id     TEXT REFERENCES books(id),
    added_at    TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, book_id)
);

-- ─────────────────────────────────────────────
-- MODIFIED: READER PROGRESS
-- Original had no user_id (single row per book).
-- Now: per-user per-book.
-- ─────────────────────────────────────────────
CREATE TABLE reader_progress (
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    book_id         TEXT REFERENCES books(id),
    current_chapter INTEGER DEFAULT 0,
    last_updated    TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, book_id)
);

-- ─────────────────────────────────────────────
-- NEW (OPTIONAL): QUERY HISTORY
-- Useful for session resume, "what did I ask last time"
-- ─────────────────────────────────────────────
CREATE TABLE query_history (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID REFERENCES users(id) ON DELETE CASCADE,
    book_id          TEXT REFERENCES books(id),
    query            TEXT NOT NULL,
    response         TEXT,
    chapter_at_query INTEGER,
    created_at       TIMESTAMP DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- MODIFIED: BOOKS
-- Track who processed a book and whether it is public
-- ─────────────────────────────────────────────
ALTER TABLE books ADD COLUMN uploaded_by UUID REFERENCES users(id);
ALTER TABLE books ADD COLUMN is_public   BOOLEAN DEFAULT true;
-- is_public = false: only the uploader can see it (for private uploads later)
```

---

## 5. Updated API Endpoints

### Auth endpoints (new)

```
POST   /api/auth/register
       Body: { email, username, password }
       Response: { user_id, token }

POST   /api/auth/login
       Body: { email, password }
       Response: { token, user }

GET    /api/auth/me
       Header: Authorization: Bearer <token>
       Response: { user_id, username, email, role }
```

### Endpoints that change

```
-- reader_progress is now scoped to the authenticated user
POST   /api/reader/progress
       Header: Authorization: Bearer <token>
       Body: { book_id, chapter_index }
       Behavior: extracts user_id from JWT, upserts reader_progress

-- dossier now uses the authenticated user's actual chapter
GET    /api/dossier/{book_id}
       Header: Authorization: Bearer <token>
       Behavior: fetches current_chapter from reader_progress for this user
                 no need to pass ?chapter= in query string

-- resolve: chapter is read from DB, not from request body
POST   /api/query/resolve
       Header: Authorization: Bearer <token>
       Body: { book_id, query }
       Behavior: chapter gate comes from reader_progress, not client
```

> **Why remove reader_chapter from the query body?**
> 
> In the personal-use design, the frontend sends `reader_chapter` with each query. That is fine when there is only one user. In a multi-user system, letting the client dictate the chapter is a spoiler vulnerability — any user could send `reader_chapter: 9999` and get full spoilers. The server must own this value, reading it from `reader_progress`. The client never controls what chapter it gets answers for.

### Library endpoints (new)

```
GET    /api/library
       Header: Authorization: Bearer <token>
       Response: { books: [{ book_id, title, current_chapter, added_at }] }

POST   /api/library/{book_id}
       Header: Authorization: Bearer <token>
       Behavior: adds book to user's library

DELETE /api/library/{book_id}
       Header: Authorization: Bearer <token>
       Behavior: removes from library

GET    /api/books/available
       No auth required
       Response: all globally available books (is_public = true)
```

---

## 6. SpoilerGate Update

The SpoilerGate class needs one change: it reads chapter from the DB instead of accepting it as a caller-provided parameter.

```python
# Original (personal use) — caller provides chapter
class SpoilerGate:
    def __init__(self, db, reader_chapter: int, book_id: str):
        self.chapter = reader_chapter

# Multi-user version — server reads chapter from DB
class SpoilerGate:
    def __init__(self, db, user_id: str, book_id: str):
        row = db.execute("""
            SELECT current_chapter FROM reader_progress
            WHERE user_id = ? AND book_id = ?
        """, [user_id, book_id]).fetchone()

        self.chapter = row['current_chapter'] if row else 0
        self.user_id = user_id
        self.book_id = book_id
```

Every endpoint handler extracts `user_id` from the JWT, passes it to `SpoilerGate`. The gate resolves the chapter itself. No user-controlled chapter values anywhere in the query path.

---

## 7. Deployment Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                        DEPLOYED STACK                         │
│                                                               │
│  ┌──────────────────┐     ┌─────────────────────────────┐    │
│  │  Next.js PWA     │     │   FastAPI (Python)           │    │
│  │  Vercel /        │────►│   Railway / Render / VPS     │    │
│  │  Netlify         │     │                              │    │
│  └──────────────────┘     └──────────────┬──────────────┘    │
│                                          │                    │
│                            ┌─────────────┴─────────────┐     │
│                            │  PostgreSQL                │     │
│                            │  Railway / Supabase / Neon │     │
│                            └───────────────────────────┘     │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  User's Browser                                       │   │
│  │  Flow (IndexedDB) — EPUB file, local reading state    │   │
│  │  PlotWise UI — queries server API with JWT            │   │
│  └───────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────┘
```

**Recommended early-stage services:**

|Component|Service|Estimated Cost|
|---|---|---|
|Frontend|Vercel (free tier)|$0|
|Backend|Railway starter|~$5/mo|
|PostgreSQL|Railway or Neon (free tier)|$0|
|FAISS indexes|Stored on backend server disk|included|
|EPUBs|Not stored server-side (Model A)|$0|

FAISS indexes live on the FastAPI server's disk. LotM's index is roughly 20MB. For dozens of books this is negligible. If you scale to hundreds of books, move indexes to a persistent volume.

---

## 8. What Stays Exactly the Same

Do not touch these. They are designed correctly already.

- The entire ingestion pipeline (Pass 1, Pass 2)
- identity_nodes, identity_merges, character_facts, relationships, factions, events, chapter_summaries
- The SpoilerGate query logic (only the chapter source changes)
- The identity resolution algorithm and merge cluster BFS
- FAISS semantic search
- The Groq runtime query and context payload builder
- Flow's core reading mechanics

The multi-user shift is **purely additive** from a data model perspective. Three new tables, two modified. The narrative intelligence layer is untouched.

---

## 9. Migration Path: Personal → Multi-User

If you build the personal version first (recommended — ship something working fast), migrating later is straightforward:

```
Step 1  Add users table, insert one row for yourself
Step 2  Add user_id to reader_progress with DEFAULT = your user UUID
Step 3  Add user_libraries, insert rows for your existing books
Step 4  Add JWT auth to FastAPI (python-jose + passlib)
Step 5  Update SpoilerGate to read chapter from DB
Step 6  Add /auth/* endpoints
Step 7  Add token handling to the Next.js frontend
Step 8  Migrate SQLite → PostgreSQL (pgloader handles this in one command)
```

Each step is independently deployable. You can add auth without touching the narrative intelligence layer at all. The character graph does not care who is asking.

---

## 10. Summary of Differences

|Dimension|Current Vision|New Vision (This Doc)|
|---|---|---|
|Users|1 (you)|Multiple, with auth|
|Auth|None|JWT (email + password)|
|EPUB storage|Local disk|Not stored server-side|
|Books|Per-user upload|Global, admin-processed|
|Character graph|Tied to local machine|Shared globally|
|Reader progress|Single row per book|Per-user per-book|
|Chapter gate source|Frontend-provided|Server-owned from reader_progress|
|Database|SQLite, localhost|PostgreSQL, deployed|
|Deployment|localhost only|Vercel + Railway|
|New tables|—|users, user_libraries, query_history|
|Modified tables|—|reader_progress (user_id), books (uploaded_by, is_public)|

---

_Read alongside `PLOTWISE_SYSTEM_DESIGN.md`. That document is the source of truth for the ingestion pipeline and narrative intelligence layer. This document covers only the delta for multi-user support._