# Update Summary — PlotWise Design Clarifications

**Date:** 27 May 2026  
**Status:** ✅ Completed

---

## What Was Updated

### 1. Design Document: `3_current-plotwise-server-epub-design.md`

#### Section 3.2 — Clarified Auth Flow
**Before:** Generic endpoint with unclear auth

**After:** Complete user flow showing:
- User clicks "PlotWise books" button
- Login screen (email + password)
- JWT issued and stored
- Access to book collection
- Add books to separate library
- Progress synced across devices

**Key addition:** Detailed explanation of JWT (Bearer token) and why it's needed for cross-device sync

#### Section 3.3 — Fixed "Stripping Flow" Misconception
**Before:** 
> "Flow's local storage features... are no longer needed for the primary use case. The Flow fork becomes lighter — strip out the local library management."

**After:** 
Clear statement that **Flow's local management is NOT stripped**:
- Local import/export still works
- Local EPUB reading unchanged
- IndexedDB still stores local books
- Offline still works for local EPUBs

**Added:** Distinction between:
- **Flow's local library** (untouched)
- **PlotWise server library** (new, separate, synced)
- Both coexist in same session with no conflict

**Added:** Clear explanation of ArrayBuffer:
- Binary data in memory (not on disk)
- Fetched from server → stored in RAM while reading
- Why it's better than download (instant, no disk usage)
- Future option: cache in Service Worker for offline

#### Section 4 — Renamed and Reorganized
**Before:** "Admin Workflow" mixed with user flow

**After:** 
- **Section 4:** "User Workflow: Accessing PlotWise Books" — confirmed flow
- **Section 5:** "Admin Workflow" — separate from user flow
- **Section 6:** "Mobile Reading Flow (End to End)" — complete scenario

#### Section 8 — Updated Summary Table
Added corrections:
- ✅ Flow's local library management is completely preserved
- ✅ PlotWise books in separate library (doesn't replace Flow's)
- ✅ Both workflows work simultaneously with no conflict
- ✅ Auth is JWT, enables cross-device sync

---

### 2. Chat Log: `chat-vsc.md`

#### Added New Q&A Section (after original clarifications)

**Question 1: Login Flow Confirmation**

> "OK, so to confirm: You click on 'PlotWise books' button → you have to sign in → then get access to the books which you can add to a separate library which will be saved irrespective of device?"

**Answer:** YES, exactly. Includes:
- Step-by-step diagram of the flow
- 10-step breakdown from initial click to cross-device resume
- Key points emphasizing separate library and device-independent progress

**Question 2: Auth Model Tradeoff**

> "Any advantage of keeping a user login? Can't we just make it available without login auth, etc., but tracking across multiple devices won't be possible? That's fine."

**Answer:** Comprehensive comparison including:

| Dimension | Option A (Login) | Option B (Anonymous) |
|-----------|---|---|
| **Advantages** | Cross-device sync, user isolation, future features, moderation | Zero friction, better privacy, simpler code, low GDPR burden |
| **Disadvantages** | Friction (login step), privacy concern, credential mgmt | No cross-device, library lost on browser clear, no user features |

**Hybrid Option:** Email-only auth (no password, lower friction)

**Recommendation:** 
- v1: Use Option A (JWT login with password) — core value is multi-device sync
- Alternatively: Start with Option B (anonymous), migrate to Option A later (schema change is tiny)

**Reasoning:** 
1. Multi-device resume at same chapter is the #1 use case
2. Login is one-time friction per device
3. Enables future features (bookmarks, reading history, sharing)

---

## Files Changed

```
/home/mehrin/repo/_gsoc/flow/plotwise 1/3_current-plotwise-server-epub-design.md
  ✅ Updated sections 3.2, 3.3, 4, 5, 6, 8
  ✅ Clarified login flow (step-by-step)
  ✅ Fixed ArrayBuffer explanation
  ✅ Removed "stripping Flow" misconception
  ✅ Emphasized coexistence of local + server EPUBs
  ✅ Updated change summary table

/home/mehrin/repo/_gsoc/flow/plotwise 1/chat-vsc.md
  ✅ Added "Additional Q&A" section at end
  ✅ Login flow confirmation with 10-step diagram
  ✅ Auth model tradeoff analysis
  ✅ Saved for reference
```

---

## Key Takeaways

### ✅ What Was Confirmed

1. **Login Flow:**
   - Click PlotWise books button → Login (email + password) → see available books → add to library → library synced across devices

2. **Separate Libraries:**
   - Flow's local library (unchanged, works offline with local EPUBs)
   - PlotWise library (new, server-synced, for pre-ingested books)
   - Both exist in same session, no conflict

3. **ArrayBuffer:**
   - Binary EPUB data fetched from server → stored in memory (not disk)
   - Instant load, no disk usage, passed directly to epub.js
   - Can be cached for offline in future (Service Worker)

4. **Auth Choice:**
   - **Recommended (v1):** JWT login with password (enables cross-device sync)
   - **Alternative:** Anonymous (simpler, but single-device only)
   - **Migration path:** Can add Option A later to Option B with small schema change

### ✅ Documentation Now Reflects

- Design intent clearly (no more "stripping Flow")
- User perspective (click button → login → add books → read)
- Technical details (JWT, ArrayBuffer, device-agnostic progress)
- Auth tradeoffs (with pros/cons and recommendation)

---

_All changes saved and ready for reference. Design document is now aligned with actual implementation intent._
