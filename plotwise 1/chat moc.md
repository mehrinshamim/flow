long-context memory systems for narrative agents and to build spoiler-aware multimodal reading companions

Can you research on this

[[0-research-plotwise]]



---

ok give me your full nalysi & summary how prd system deign , data flow etc should be -> create md file mention reason why you chose this design - any alternatives if needed - , explain as you are onboarding a new developer to this project

ans
The full document is ready. Here's what's covered and the key decisions worth calling out explicitly:

---

**The three non-obvious design calls:**

**1. SQLite over static JSON files** — your original plan of one JSON per chapter collapses at late-book chapters for LotM because Klein's profile alone would be thousands of tokens by chapter 800. SQLite with dynamic gated queries builds the payload _on demand_ at the right size.

**2. Two-pass ingestion** — 8B for per-chapter extraction (cheap, sequential, sufficient), 70B only for flagged reveal chapters (maybe 80 out of 1,400 for LotM). This keeps the one-time ingestion cost under $1 for LotM while getting the identity resolution right where it actually matters.

**3. Identity nodes as a graph, not rows** — the `identity_merges` table with `revealed_chapter` is the entire spoiler-safe merge system. The BFS cluster algorithm at query time means you never need to modify the underlying data — you just change what the reader can _see_ of the graph.

**The build order in Section 16 is the actual roadmap** — each step is independently testable. Start with the parser + schema, get to a working `/resolve` endpoint, then bolt on the frontend. The spoiler gate unit tests (step 6) are the most important thing to get right before anything else.

[[1_plotwise-system-design]]

---

will this sql schema work if like users can be multiple, idk check how flow tracks usersw , is there sign in users or is it locally stored , my vision is , books are uploaded globally , like orv & lotm has been processed and we can use ai fo rthese uploaded books & user can add them to library & start reading - if this is diffreent from the currnet vision , explain current vision - make this a separate design & giev md file with chnages needed


ans
[[2_plotwise-multiuser-design]]
Here's the short version of everything important in that doc:

**Flow's storage model:** Fully local, IndexedDB, no user accounts whatsoever. It's essentially a browser app that happens to read EPUBs. There's no server to hook into for user state — you build that entirely fresh in PlotWise.

**The critical architectural shift** in the multi-user model is one decision that changes the security of the entire system: the spoiler gate chapter must be **server-owned**, not client-provided. In the personal version, the frontend sends `reader_chapter: 47` with each query. In multi-user, any user could send `reader_chapter: 9999` and spoil themselves (or fish for full graph data). The server reads chapter from `reader_progress` in the DB — the client never touches it.

**What's actually additive:** Only three new tables (`users`, `user_libraries`, `query_history`), two column additions to existing tables, and one change to how SpoilerGate resolves the chapter. The entire ingestion pipeline, character graph, identity merge system, and FAISS index are completely unchanged. The narrative intelligence layer genuinely doesn't know or care how many users exist.

**Build order recommendation:** Build the personal version first from the main design doc. It'll be working in weeks. Then migrate using the 8-step path in Section 9 — each step is independently deployable and none of them touch the AI layer.

----
wait is it not possible then to do this , like i want to read orv in my phone, but i cant do the prcessing in my phone, thats why , mayb emake it so these books are available in the server itself , & i can download it so i can use it in my device for reading - & for theqns , & all fns after ingestion - that is send by the client side to server i will have to deploy this project fo rthis to work like the processing can happen loaclly or as like smthng only admin can do - if ever deployed, but users can get acess to those books for which the ai is available

ans -
[[3_current-plotwise-server-epub-design]]

Yes, totally doable and actually simpler than Model A. This is the right call for your use case. Let me write the updated design.The key insight that makes this clean: **epub.js natively accepts an ArrayBuffer**. You don't need to rethink the reader at all — just swap `ePub("/local/file")` for `ePub(await fetch("/api/books/orv/epub").then(r => r.arrayBuffer()))`. That one line change is the entire difference between a local-only reader and a cross-device one.

Three things worth flagging:

**Flow becomes much lighter.** You're keeping epub.js for rendering and stripping most of Flow's local storage management (IndexedDB import/export, local library). PlotWise's server replaces all of that. The fork gets simpler, not more complex.

**The ingestion stays fully local.** You run Pass 1 + Pass 2 on your machine, the DB and FAISS indexes get built locally, then you either push the DB to Railway/Neon or just re-run ingestion once on the deployed server. Either way, processing never happens on a phone.

**$7–10/month gets you everything** — Railway for FastAPI + persistent disk (EPUB files live here), Neon for PostgreSQL, Vercel for the frontend. That's a fully working cross-device reading companion. Cloudflare R2 is the upgrade path when you have enough books that server disk feels messy.


---
ig will need money no matter what?
any free alts available