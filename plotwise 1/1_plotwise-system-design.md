# PlotWise — System Design & Developer Onboarding

> Last updated: May 2026  
> Audience: Any developer joining the project cold  
> Read time: ~25 min

---

## Table of Contents

1. [What Is PlotWise?](#1-what-is-plotwise)
2. [The Problem, Precisely](#2-the-problem-precisely)
3. [Core Design Philosophy](#3-core-design-philosophy)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Phase 1 — Ingestion Pipeline](#5-phase-1--ingestion-pipeline)
6. [Phase 2 — Runtime Query Engine](#6-phase-2--runtime-query-engine)
7. [Data Model](#7-data-model)
8. [The Spoiler Gate](#8-the-spoiler-gate)
9. [Identity Resolution System](#9-identity-resolution-system)
10. [API Design](#10-api-design)
11. [Frontend Integration (Flow Fork)](#11-frontend-integration-flow-fork)
12. [Tech Stack & Rationale](#12-tech-stack--rationale)
13. [Alternatives Considered](#13-alternatives-considered)
14. [Known Hard Problems](#14-known-hard-problems)
15. [Development Setup](#15-development-setup)
16. [Project File Structure](#16-project-file-structure)

---

## 1. What Is PlotWise?

PlotWise is a **progress-aware narrative intelligence engine** layered over a web-based EPUB reader. It lets readers ask questions about the story they are currently reading — characters, factions, events, relationships — and guarantees that every answer is bounded to what the reader has already encountered. No spoilers. Ever.

It is not a summary tool. It is not a chatbot that happens to know about books. It is a **stateful reading companion** that builds an internal model of the story as the reader progresses through it.

### Target content

PlotWise is designed for the hardest possible reading cases:

|Novel|Chapters|Why it's hard|
|---|---|---|
|Lord of the Mysteries (LotM)|1,400+|12+ aliases per protagonist, Beyonder pathway sequences are spoilers, secret organizations with shifting members|
|Omniscient Reader's Viewpoint (ORV)|550+|Constellation identities, scenario conditions, character roles revealed progressively|
|The Beginning After The End (TBATE)|500+|Faction geography, asura politics, protagonist power evolution|
|Reverend Insanity|2,300+|Soul swaps, possession, identity cycling across lifetimes|

These are not edge cases. They are the primary design target. If PlotWise works for LotM, it works for anything.

---

## 2. The Problem, Precisely

### 2.1 The Wiki Landmine

A reader at chapter 200 of LotM wants to remember who "Dunn Smith" is. They Google it. The Fandom wiki page for Dunn Smith reveals his fate at chapter 800 in the second paragraph. The reader's emotional experience of that arc is destroyed.

This is not a hypothetical. It happens to every reader of serialized fiction, every time.

### 2.2 The Generic LLM Failure

If you paste a chapter into ChatGPT or Claude and ask "who is Gehrman Sparrow?", two things go wrong:

1. **No historical context.** The model doesn't know what happened in the previous 199 chapters unless you paste all of them — which is expensive and hits context limits.
2. **Hallucinated internet spoilers.** LotM and ORV exist on the internet. The model's pretrained knowledge includes fan wikis, Reddit discussions, and translated chapters that reveal end-game spoilers. It will blend those into its answer without warning.

### 2.3 The Cognitive Load Problem

Serialized webnovels introduce hundreds of named characters. Readers take breaks. When they return weeks later, they've forgotten:

- Who a character is and why they matter
- What a faction's goals are
- Why two characters are enemies
- What happened in a specific arc three hundred chapters ago

There is no tool that solves this safely. PlotWise is that tool.

---

## 3. Core Design Philosophy

These are the principles that drive every design decision in this document. When in doubt, come back here.

### 3.1 The Spoiler Gate is Inviolable

Every single piece of information the system returns to a user must pass through a chapter-index filter. There are no exceptions. A piece of information is safe to show if and only if it was introduced at or before the reader's current chapter.

This is not a UX preference. It is the foundational trust contract between the product and its users. Violating it once destroys the product.

### 3.2 Ingestion is Offline, Queries are Real-Time

Heavy AI processing happens once, during ingestion, before the reader ever opens the book. At query time, the system does fast structured lookups and a single Groq API call. The reader should never wait more than 1–2 seconds for an answer.

This separation is why the system is viable. Doing live multi-chapter analysis on every query would be expensive, slow, and fragile.

### 3.3 The Character Graph is Temporally Versioned

Characters are not static rows in a database. They are nodes in a graph whose topology changes over time. Relationships form, break, and transform. Identities merge. Factions splinter. The data model must represent this as timestamped mutations, not overwritten state.

### 3.4 Generic by Design

PlotWise must work on any EPUB. It cannot hardcode LotM-specific logic. Every design decision must be implementable for a novel the system has never seen. LotM is a stress test, not a special case.

---

## 4. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PLOTWISE SYSTEM                             │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    INGESTION PIPELINE (one-time)             │  │
│  │                                                              │  │
│  │  EPUB File                                                   │  │
│  │    └─► ebooklib parser ──► chapter_chunks[]                  │  │
│  │                                  │                           │  │
│  │              ┌───────────────────┘                           │  │
│  │              │                                               │  │
│  │         PASS 1 (per-chapter, 8B model)                       │  │
│  │         Extract: names, aliases, events,                     │  │
│  │         relationships, factions, reveal-flags                │  │
│  │              │                                               │  │
│  │         PASS 2 (flagged chapters only, 70B/Claude)           │  │
│  │         Resolve: identity merges, canonical nodes            │  │
│  │              │                                               │  │
│  │         SQLite DB  +  FAISS index (local)                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          ↕ persisted to disk                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    FASTAPI BACKEND (runtime)                  │  │
│  │                                                              │  │
│  │  Request: { query, reader_chapter, book_id }                 │  │
│  │              │                                               │  │
│  │         SPOILER GATE                                         │  │
│  │         Filter all DB access to chapter ≤ reader_chapter     │  │
│  │              │                                               │  │
│  │         IDENTITY RESOLVER                                    │  │
│  │         Merge profiles if merge event ≤ reader_chapter       │  │
│  │              │                                               │  │
│  │         Build lean context payload (800–1200 tokens)         │  │
│  │              │                                               │  │
│  │         Groq API (llama-3.3-70b-versatile)                   │  │
│  │              │                                               │  │
│  │         Response: spoiler-safe answer                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          ↕ REST API                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    NEXT.JS PWA (frontend)                    │  │
│  │                                                              │  │
│  │  Flow fork (pacexy/flow) — EPUB reader mechanics unchanged   │  │
│  │  PlotWise additions:                                         │  │
│  │    - Chapter progress hook                                   │  │
│  │    - Highlight → "Who is this?" popover                      │  │
│  │    - Character sidebar / dossier panel                       │  │
│  │    - Arc recap panel                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Phase 1 — Ingestion Pipeline

Ingestion runs once per book, offline, before the reader opens it. It produces the SQLite database and FAISS index that all runtime queries depend on.

### 5.1 EPUB Parsing

```python
import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

def parse_epub(filepath: str) -> list[dict]:
    book = epub.read_epub(filepath)
    chapters = []
    for i, item in enumerate(book.get_items_of_type(ebooklib.ITEM_DOCUMENT)):
        soup = BeautifulSoup(item.get_content(), 'html.parser')
        text = soup.get_text(separator='\n', strip=True)
        chapters.append({
            "chapter_index": i,
            "title": item.get_name(),
            "text": text,
            "token_count": len(text.split())  # rough estimate
        })
    return chapters
```

Each chapter becomes a standalone processing unit. The chapter index is the single most important piece of metadata in the entire system — it is the key that the spoiler gate uses for every single lookup.

### 5.2 Pass 1 — Per-Chapter Sequential Extraction

**Model:** `llama-3.1-8b-instant` via Groq (cheap, fast, sufficient for extraction)  
**Purpose:** Build the initial structured state from each chapter  
**Pattern:** Sequential — Chapter N is processed using Chapter N-1's entity list as context (delta extraction)

**Extraction prompt structure:**

```
System:
You are a narrative entity extractor. You extract structured information 
from novel chapters. Return only valid JSON, no preamble.

User:
KNOWN ENTITIES SO FAR (from previous chapters):
{prev_entity_summary}

CURRENT CHAPTER ({chapter_index}):
{chapter_text}

Extract any NEW or UPDATED information:
1. New characters introduced (name, description, role)
2. New aliases or titles for existing characters
3. New relationships formed or changed
4. New factions or organizations mentioned
5. Key events (summarized in 1-2 sentences)
6. FLAG: Any phrases suggesting identity reveals
   (e.g., "is actually", "was revealed to be", "true identity",
    "all along", "unmasked", "same person as")

Return as JSON matching this schema: {schema}
```

**Why sequential / delta extraction?**  
Processing each chapter with context of what came before allows the model to correctly attribute a new alias to an existing character rather than creating a duplicate node. Without this, you end up with 40 separate nodes for Klein Moretti across 1,400 chapters.

**Pass 1 output per chapter:**

- New identity nodes
- New character facts (with `introduced_chapter`)
- New relationships (with `established_chapter`)
- New faction memberships
- Potential reveal flags (chapter index + flagged text snippet)

### 5.3 Pass 2 — Identity Merge Resolution

**Model:** `llama-3.3-70b-versatile` via Groq or Claude Haiku  
**Input:** Only the flagged chapters from Pass 1 (typically 5–10% of total chapters)  
**Purpose:** Resolve which identity nodes should be merged, and at which chapter

**Why a separate pass?**  
Identity reveals in serialized fiction are often foreshadowed across 200+ chapters before the explicit reveal. A chapter-by-chapter model cannot see the full pattern. Pass 2 takes all flagged chapters together and asks the model to reason about the complete merge event.

**Pass 2 prompt structure:**

```
System:
You are resolving character identity merges in a novel. 
You will be given chapters where identity reveals may occur.
For each confirmed merge, extract structured data. Return JSON only.

User:
KNOWN IDENTITY NODES:
{all_nodes_with_ids}

FLAGGED CHAPTERS (potential identity reveals):
{flagged_chapter_texts}

For each confirmed identity reveal, return:
{
  "node_a_id": <id of first identity>,
  "node_b_id": <id of second identity>,
  "revealed_at_chapter": <chapter index of the reveal>,
  "merge_type": "alias|disguise|reincarnation|possession|split_personality",
  "canonical_node_id": <which node becomes the primary identity post-reveal>,
  "confidence": "high|medium|low"
}
```

**Merge types matter for display logic:**

- `alias` — same person using a fake name (Klein / Gehrman Sparrow)
- `disguise` — physical disguise with separate backstory
- `reincarnation` — past-life identity, memories surface gradually
- `possession` — another entity inhabiting the same body (Reverend Insanity)
- `split_personality` — both identities are "real" simultaneously

### 5.4 Embedding & FAISS Index

After both passes, chapter summaries are embedded and stored in a local FAISS index for semantic search at query time.

```python
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

model = SentenceTransformer('all-MiniLM-L6-v2')  # small, fast, local

def build_faiss_index(chapter_summaries: list[dict]):
    texts = [c['summary'] for c in chapter_summaries]
    embeddings = model.encode(texts, show_progress_bar=True)
    
    index = faiss.IndexFlatL2(embeddings.shape[1])
    index.add(np.array(embeddings, dtype='float32'))
    
    faiss.write_index(index, f"data/{book_id}_index.faiss")
    # also save chapter_id → text mapping
```

**Why local FAISS, not Pinecone/Weaviate?**  
This is a personal-use system. No cloud vector DB dependency. FAISS is a native library, zero latency, zero cost, works offline. The index for 1,400 chapters is roughly 15–20MB on disk.

### 5.5 Ingestion Cost Estimate

|Novel|Chapters|Pass 1 (8B Groq)|Pass 2 (70B, ~5% flagged)|Total (approx.)|
|---|---|---|---|---|
|LotM|1,400|~$0.60|~$0.15|**~$0.75**|
|ORV|550|~$0.24|~$0.06|**~$0.30**|
|TBATE|500|~$0.22|~$0.05|**~$0.27**|
|Reverend Insanity|2,300|~$1.00|~$0.25|**~$1.25**|

_Estimates based on Groq's pricing as of mid-2026. Run once, never again._  
_Alternatively: use Claude Haiku for Pass 1 at comparable cost with higher extraction accuracy._

---

## 6. Phase 2 — Runtime Query Engine

This is what the reader interacts with in real time.

### 6.1 Query Flow

```
User query: "Who is Gehrman Sparrow?"
Reader chapter: 120
Book ID: lotm

Step 1 — Name resolution
  Search identity_nodes WHERE display_name LIKE '%Gehrman%'
  → found: node_id = 42, first_chapter = 45

Step 2 — Spoiler gate check
  node.first_chapter (45) <= reader_chapter (120) ✓ safe to return

Step 3 — Identity merge check
  SELECT * FROM identity_merges 
  WHERE (node_a = 42 OR node_b = 42) 
  AND revealed_chapter <= 120
  → found: merge with node_id = 1 (Klein Moretti), revealed at chapter 118

Step 4 — Cluster collection
  Get all nodes in this merge cluster visible at chapter 120
  → [node 1: Klein Moretti, node 42: Gehrman Sparrow]
  → merge type: alias, canonical: Klein Moretti

Step 5 — Fact retrieval
  SELECT * FROM character_facts 
  WHERE node_id IN (1, 42) 
  AND introduced_chapter <= 120
  ORDER BY introduced_chapter

Step 6 — Semantic context
  FAISS search for chapters semantically related to "Gehrman Sparrow"
  → top 3 relevant chapter summaries, all <= chapter 120

Step 7 — Context assembly
  Build payload: ~900 tokens
  {
    "character": "Klein Moretti (also known as Gehrman Sparrow from ch.45)",
    "facts": [...],
    "relevant_chapters": [...]
  }

Step 8 — Groq call
  System: "You are PlotWise. Answer only using provided context.
           Never reference events beyond what is in this context."
  User: "Who is Gehrman Sparrow?" + assembled payload

Step 9 — Return to user
  Spoiler-safe answer in ~600ms
```

### 6.2 Context Payload Size Management

A critical implementation detail: the context payload must stay within a useful size. For a character like Klein Moretti at chapter 1,200, the raw fact list would be thousands of tokens.

**Strategy: Tiered fact retrieval**

```python
def build_context_payload(nodes, reader_chapter, query, max_tokens=1000):
    
    # Tier 1: Always include — core identity facts
    core = get_core_facts(nodes, reader_chapter)        # ~200 tokens
    
    # Tier 2: Include if relevant — recent appearances
    recent = get_recent_facts(nodes, reader_chapter, n=5)  # ~300 tokens
    
    # Tier 3: Include if space — semantically relevant to query
    semantic = faiss_search(query, reader_chapter, k=3)    # ~400 tokens
    
    # Tier 4: Trim if over budget
    return trim_to_budget([core, recent, semantic], max_tokens)
```

---

## 7. Data Model

### 7.1 Full SQLite Schema

```sql
-- ─────────────────────────────────────────────
-- BOOKS
-- ─────────────────────────────────────────────
CREATE TABLE books (
    id              TEXT PRIMARY KEY,   -- slug: "lord-of-the-mysteries"
    title           TEXT NOT NULL,
    author          TEXT,
    total_chapters  INTEGER,
    epub_path       TEXT,
    ingested_at     TEXT,               -- ISO timestamp
    ingestion_status TEXT DEFAULT 'pending'
                                        -- pending | pass1_done | complete
);

-- ─────────────────────────────────────────────
-- READER STATE
-- ─────────────────────────────────────────────
CREATE TABLE reader_progress (
    book_id         TEXT REFERENCES books(id),
    current_chapter INTEGER DEFAULT 0,
    last_updated    TEXT,
    PRIMARY KEY (book_id)
);

-- ─────────────────────────────────────────────
-- IDENTITY NODES
-- ─────────────────────────────────────────────
-- Each distinct identity the reader encounters is a node.
-- Two nodes can be merged later via identity_merges.
CREATE TABLE identity_nodes (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id         TEXT REFERENCES books(id),
    display_name    TEXT NOT NULL,
    first_chapter   INTEGER NOT NULL,
    node_type       TEXT DEFAULT 'person',
                                        -- person | organization | entity | place
    gender          TEXT,
    profile_summary TEXT,               -- extracted description at intro
    importance      TEXT DEFAULT 'minor'
                                        -- major | supporting | minor
);

-- ─────────────────────────────────────────────
-- IDENTITY MERGES
-- ─────────────────────────────────────────────
-- Records when two nodes are revealed to be the same entity.
-- This table is the core of the spoiler-safe identity system.
CREATE TABLE identity_merges (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id             TEXT REFERENCES books(id),
    node_a              INTEGER REFERENCES identity_nodes(id),
    node_b              INTEGER REFERENCES identity_nodes(id),
    revealed_chapter    INTEGER NOT NULL,   -- THE spoiler gate key
    merge_type          TEXT NOT NULL,
                        -- alias | disguise | reincarnation | possession
    canonical_node      INTEGER REFERENCES identity_nodes(id),
                        -- which node is "primary" after the reveal
    confidence          TEXT DEFAULT 'high',
    extractor_note      TEXT                -- why the model flagged this
);

-- ─────────────────────────────────────────────
-- CHARACTER FACTS
-- ─────────────────────────────────────────────
-- All information about any node is stored as facts,
-- each timestamped with the chapter it was introduced.
CREATE TABLE character_facts (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    node_id             INTEGER REFERENCES identity_nodes(id),
    book_id             TEXT REFERENCES books(id),
    fact_type           TEXT NOT NULL,
                        -- alias | ability | title | affiliation |
                        --  personality | appearance | goal | backstory
    content             TEXT NOT NULL,
    introduced_chapter  INTEGER NOT NULL,
    is_superseded       INTEGER DEFAULT 0,
                        -- 1 if a later fact overrides this one
    superseded_at_chapter INTEGER
);

-- ─────────────────────────────────────────────
-- RELATIONSHIPS
-- ─────────────────────────────────────────────
CREATE TABLE relationships (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id             TEXT REFERENCES books(id),
    node_a              INTEGER REFERENCES identity_nodes(id),
    node_b              INTEGER REFERENCES identity_nodes(id),
    relationship_type   TEXT NOT NULL,
                        -- ally | enemy | mentor | student | family |
                        --  romantic | subordinate | superior | unknown
    description         TEXT,
    established_chapter INTEGER NOT NULL,
    ended_chapter       INTEGER,        -- NULL if still active
    is_secret           INTEGER DEFAULT 0
                        -- 1 if the relationship itself is a spoiler
);

-- ─────────────────────────────────────────────
-- FACTIONS
-- ─────────────────────────────────────────────
CREATE TABLE factions (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id             TEXT REFERENCES books(id),
    name                TEXT NOT NULL,
    description         TEXT,
    faction_type        TEXT,           -- organization | kingdom | cult | guild
    first_chapter       INTEGER NOT NULL
);

CREATE TABLE faction_memberships (
    faction_id          INTEGER REFERENCES factions(id),
    node_id             INTEGER REFERENCES identity_nodes(id),
    role                TEXT,
    joined_chapter      INTEGER NOT NULL,
    left_chapter        INTEGER,        -- NULL if still member
    PRIMARY KEY (faction_id, node_id, joined_chapter)
);

-- ─────────────────────────────────────────────
-- EVENTS
-- ─────────────────────────────────────────────
CREATE TABLE events (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id             TEXT REFERENCES books(id),
    chapter_index       INTEGER NOT NULL,
    summary             TEXT NOT NULL,  -- 1-3 sentence summary
    event_type          TEXT,           -- battle | reveal | death | meeting | arc_start
    significance        TEXT DEFAULT 'minor'
                                        -- major | moderate | minor
);

CREATE TABLE event_participants (
    event_id            INTEGER REFERENCES events(id),
    node_id             INTEGER REFERENCES identity_nodes(id),
    role                TEXT,           -- protagonist | antagonist | witness
    PRIMARY KEY (event_id, node_id)
);

-- ─────────────────────────────────────────────
-- CHAPTER SUMMARIES
-- ─────────────────────────────────────────────
CREATE TABLE chapter_summaries (
    book_id             TEXT REFERENCES books(id),
    chapter_index       INTEGER,
    summary             TEXT NOT NULL,
    faiss_vector_id     INTEGER,        -- maps to FAISS index position
    PRIMARY KEY (book_id, chapter_index)
);
```

### 7.2 Key Design Decisions in the Schema

**Why `character_facts` instead of columns on `identity_nodes`?**

A character's abilities, aliases, and affiliations are not static properties — they change. A column-per-property model would need constant updates with no history. The facts table lets you ask "what did the reader know about this character at chapter 50?" by simply filtering on `introduced_chapter`. The temporal dimension is native.

**Why `is_superseded` on facts?**

Some facts are overwritten by later information. Example: at chapter 100, a character's goal is "find the artifact." At chapter 300, their goal changes to "destroy the organization." Both facts are real — but at chapter 150, only the first should be shown. At chapter 350, the second is more current. `is_superseded` + `superseded_at_chapter` handles this without deleting history.

**Why store `ended_chapter` on relationships?**

Relationships end. Allies become enemies. Mentors die. If you only store the current state, you lose the history. A reader at chapter 400 asking "why do X and Y hate each other?" needs the history of that relationship arc to get a meaningful answer.

---

## 8. The Spoiler Gate

The Spoiler Gate is not a feature. It is a constraint layer that wraps every single database access in the system. No query result escapes it.

### 8.1 Implementation

```python
class SpoilerGate:
    def __init__(self, db: sqlite3.Connection, reader_chapter: int, book_id: str):
        self.db = db
        self.chapter = reader_chapter
        self.book_id = book_id

    def get_node(self, node_id: int) -> dict | None:
        return self.db.execute("""
            SELECT * FROM identity_nodes
            WHERE id = ? AND book_id = ? AND first_chapter <= ?
        """, [node_id, self.book_id, self.chapter]).fetchone()

    def get_facts(self, node_ids: list[int]) -> list[dict]:
        placeholders = ','.join('?' * len(node_ids))
        return self.db.execute(f"""
            SELECT * FROM character_facts
            WHERE node_id IN ({placeholders})
            AND book_id = ?
            AND introduced_chapter <= ?
            ORDER BY introduced_chapter
        """, [*node_ids, self.book_id, self.chapter]).fetchall()

    def get_visible_merges(self, node_id: int) -> list[dict]:
        return self.db.execute("""
            SELECT * FROM identity_merges
            WHERE (node_a = ? OR node_b = ?)
            AND book_id = ?
            AND revealed_chapter <= ?
        """, [node_id, node_id, self.book_id, self.chapter]).fetchall()

    def get_relationships(self, node_ids: list[int]) -> list[dict]:
        placeholders = ','.join('?' * len(node_ids))
        return self.db.execute(f"""
            SELECT * FROM relationships
            WHERE (node_a IN ({placeholders}) OR node_b IN ({placeholders}))
            AND book_id = ?
            AND established_chapter <= ?
            AND (ended_chapter IS NULL OR ended_chapter <= ?)
        """, [*node_ids, *node_ids, self.book_id, self.chapter, self.chapter]).fetchall()
```

**Every FastAPI endpoint receives a `SpoilerGate` instance.** No raw database calls are permitted in endpoint handlers — only gate-wrapped calls.

### 8.2 The FAISS Gate

Semantic search also needs gating. When searching the FAISS index, retrieve top-K results and then filter by chapter:

```python
def gated_semantic_search(query: str, reader_chapter: int, k: int = 10) -> list[dict]:
    query_vec = embed(query)
    # over-fetch because some will be filtered
    distances, indices = faiss_index.search(query_vec, k * 3)
    
    results = []
    for idx in indices[0]:
        summary = chapter_summaries[idx]
        if summary['chapter_index'] <= reader_chapter:
            results.append(summary)
        if len(results) >= k:
            break
    return results
```

---

## 9. Identity Resolution System

This is the most complex part of the system. Read it carefully.

### 9.1 The Problem Restated

During ingestion, the extraction model creates separate identity nodes for what appear to be separate characters. Some of these will later be revealed to be the same entity. The system must:

1. Keep them separate in the database (they are separate in the reader's experience until the reveal)
2. Merge them transparently at query time once the reveal chapter is passed
3. Never expose the merge before the reveal chapter

### 9.2 The Merge Cluster Algorithm

```python
def get_merge_cluster(seed_node_id: int, gate: SpoilerGate) -> list[int]:
    """
    Returns all node IDs that are revealed to be the same entity
    as seed_node_id, as of the reader's current chapter.
    Uses BFS over the identity_merges graph.
    """
    visited = set()
    queue = [seed_node_id]
    
    while queue:
        current = queue.pop(0)
        if current in visited:
            continue
        visited.add(current)
        
        merges = gate.get_visible_merges(current)
        for merge in merges:
            other = merge['node_b'] if merge['node_a'] == current else merge['node_a']
            if other not in visited:
                queue.append(other)
    
    return list(visited)


def resolve_identity(query_name: str, gate: SpoilerGate) -> dict:
    # 1. Find matching node
    node = find_node_by_name(query_name, gate)
    if not node:
        return {"error": "Character not found within your reading progress"}
    
    # 2. Get merge cluster
    cluster_ids = get_merge_cluster(node['id'], gate)
    
    # 3. Determine canonical node
    canonical = get_canonical_node(cluster_ids, gate)
    
    # 4. Collect all facts from all nodes in cluster
    facts = gate.get_facts(cluster_ids)
    relationships = gate.get_relationships(cluster_ids)
    
    # 5. Build unified profile
    return {
        "canonical_name": canonical['display_name'],
        "also_known_as": [
            n['display_name'] for n in get_nodes(cluster_ids, gate)
            if n['id'] != canonical['id']
        ],
        "first_appearance": min(n['first_chapter'] for n in get_nodes(cluster_ids, gate)),
        "facts": facts,
        "relationships": relationships,
        "merge_count": len(cluster_ids) - 1
    }
```

### 9.3 Merge Types and Display Behavior

|Merge Type|Display Before Reveal|Display After Reveal|
|---|---|---|
|`alias`|Two separate profiles|One profile: "Klein Moretti (also known as Gehrman Sparrow from ch.45)"|
|`disguise`|Two separate profiles|One profile with note: "Disguised as [X] from ch.Y to ch.Z"|
|`reincarnation`|Two separate profiles|One profile: "Arthur Leywin — past life: [name]. Memories recovered across ch.X–Y"|
|`possession`|Two separate profiles|Separate profiles with relationship: "[A] was possessed by [B] from ch.X to ch.Y"|

**Possession is intentionally NOT a full merge.** The two entities remain distinct people; the possession is a relationship event, not a unification. Reverend Insanity requires this distinction.

---

## 10. API Design

### 10.1 Endpoints

```
POST   /api/books/ingest
       Body: { epub_path: string }
       Response: { book_id, status, chapter_count }
       — Triggers ingestion pipeline (async, returns job ID)

GET    /api/books/{book_id}/status
       Response: { status, pass1_progress, pass2_progress }

POST   /api/reader/progress
       Body: { book_id, chapter_index }
       Response: { ok }
       — Called by frontend on chapter change

POST   /api/query/resolve
       Body: { book_id, query, reader_chapter }
       Response: { answer, character_profile?, source_chapters }
       — Main Q&A endpoint. "Who is X?", "Why are X and Y enemies?"

GET    /api/dossier/{book_id}
       Query: ?chapter=120
       Response: { characters[], factions[], recent_events[] }
       — Full dashboard data for the dossier sidebar

POST   /api/query/summarize
       Body: { book_id, arc_description, reader_chapter }
       Response: { summary }
       — "Summarize the political tension leading up to where I am"

GET    /api/character/{book_id}/{node_id}
       Query: ?chapter=120
       Response: { full character profile, gated to chapter }
```

### 10.2 Request / Response Contract for `/api/query/resolve`

```json
// Request
{
  "book_id": "lord-of-the-mysteries",
  "query": "Who is Gehrman Sparrow and why does everyone fear him?",
  "reader_chapter": 120
}

// Response
{
  "answer": "Gehrman Sparrow is a name used by Klein Moretti (revealed to be the same person at chapter 118). He is a Beyonder of the Fool pathway operating in Backlund, known for his reputation as an exceptionally ruthless and unpredictable adventurer. By chapter 120, he has earned this reputation through...",
  "character_profile": {
    "canonical_name": "Klein Moretti",
    "also_known_as": ["Gehrman Sparrow"],
    "pathway": "Fool",
    "first_appearance": 1,
    "last_seen": 119
  },
  "source_chapters": [45, 87, 112, 118, 119],
  "spoiler_safe": true
}
```

---

## 11. Frontend Integration (Flow Fork)

### 11.1 What We Don't Touch

Flow's core mechanics stay untouched:

- EPUB rendering via epub.js
- Pagination system
- Offline storage (IndexedDB)
- Themes and font controls
- Mobile touch UX

### 11.2 What We Add

**A. Chapter progress hook**

Flow fires events when chapters change. We intercept this:

```javascript
// In reader component (Flow fork)
reader.on('relocated', (location) => {
  const chapterIndex = location.start.index;
  fetch('/api/reader/progress', {
    method: 'POST',
    body: JSON.stringify({ book_id: currentBookId, chapter_index: chapterIndex })
  });
});
```

**B. Highlight → "Who is this?" popover**

```javascript
document.addEventListener('selectionchange', () => {
  const selection = window.getSelection();
  if (selection.toString().trim().length > 0) {
    showWhoIsThisButton(selection);
  }
});

function onWhoIsThisClick(selectedText) {
  openQueryPanel({
    query: `Who is ${selectedText}?`,
    prefill: selectedText
  });
}
```

**C. PlotWise sidebar panel**

A right-side drawer component (React + Tailwind, following Flow's existing design tokens) with three tabs:

- **Ask** — free-form query input
- **Dossier** — character/faction cards fetched from `/api/dossier`
- **Recap** — arc summary input

The sidebar is a toggle — a single floating button opens it. It never obscures the reading surface on mobile.

### 11.3 State Management

Reader state (current chapter, book ID) lives in a React context:

```javascript
const ReaderContext = createContext({
  bookId: null,
  currentChapter: 0,
  setChapter: () => {}
});
```

The PlotWise components consume this context to know what chapter to gate all queries against.

---

## 12. Tech Stack & Rationale

|Component|Choice|Rationale|Alternative Considered|
|---|---|---|---|
|EPUB parsing|`ebooklib` (Python)|Battle-tested, clean chapter extraction|`pandoc` — heavier, less structured output|
|Pass 1 model|Groq `llama-3.1-8b-instant`|Fast and cheap for extraction tasks|Claude Haiku — slightly more accurate, marginally higher cost|
|Pass 2 model|Groq `llama-3.3-70b-versatile`|Needed for cross-chapter identity reasoning|Claude Sonnet — better, but 3× cost for one-time ingestion|
|Structured store|SQLite|Zero-infra, sufficient for personal use, works offline|PostgreSQL — overkill for single-user, adds infra complexity|
|Vector search|FAISS (local)|No cloud dependency, 15MB index, sub-10ms search|Chroma, Pinecone — add cloud/infra cost for negligible benefit|
|Embeddings|`all-MiniLM-L6-v2`|Local, 80MB model, good semantic quality|OpenAI embeddings — cloud dependency, per-call cost|
|Runtime inference|Groq|Genuinely sub-second inference for 70B model|Anthropic Claude — excellent quality but 3–5× slower than Groq|
|Backend|FastAPI + Python|Mehrin's existing stack, excellent async support|Node.js Express — weaker AI ecosystem, unnecessary context switch|
|Frontend|Next.js (Flow fork)|Already a mobile-first PWA, EPUB rendering solved|Build from scratch — months of work for zero user-facing gain|
|Styling|Tailwind CSS|Flow already uses it, consistent with existing system|CSS Modules — inconsistent with Flow's existing patterns|

---

## 13. Alternatives Considered

### 13.1 Static JSON files per chapter (rejected)

The original idea was to pre-generate one JSON file per chapter (e.g., `lotm_chapter_45_state.json`) and append the whole file to the Groq prompt at query time.

**Why rejected:**

- By chapter 800 of LotM, the state JSON is thousands of tokens. The Klein Moretti entry alone spans 12 aliases, 4 pathway advancements, and 50+ relationships.
- 1,400 static files is rigid — if the reader is at chapter 47, there's no file for 47 unless you pre-generate all 1,400.
- No semantic search — you can't answer "who uses lightning-based abilities?" without a vector index.
- SQLite + dynamic gated queries gives the same result with a fraction of the context budget.

### 13.2 Live full-book analysis on every query (rejected)

Send the entire book text to the model on each query, ask it to answer with the spoiler gate as a prompt instruction.

**Why rejected:**

- LotM at 1,400 chapters is roughly 4–5 million tokens. No current model handles this in a single call.
- Even if context windows supported it, the cost per query would be prohibitive.
- Prompt-level spoiler instructions ("only answer from before chapter 120") are not reliable — models leak future knowledge under adversarial or ambiguous queries.

### 13.3 Fan wiki scraping + spoiler detection (rejected)

Scrape existing wikis (Fandom, etc.) and filter results by chapter.

**Why rejected:**

- Wikis are structurally organized around final-state knowledge. Spoiler filtering on top of them is fragile.
- No coverage for novels without popular wikis (most webnovels).
- Violates the "works for any novel" design principle.
- The core value proposition is that PlotWise builds its knowledge _from the book itself_, not from the internet.

### 13.4 Fine-tuning a model (rejected for now)

Fine-tune a small model on narrative entity extraction for specific series.

**Why rejected (for now):**

- High one-time cost and complexity.
- Requires labeled training data.
- Prompt engineering on a capable base model gets 90% of the way there.
- Worth revisiting if ingestion accuracy becomes the bottleneck.

---

## 14. Known Hard Problems

These are open problems in the current design. Document them honestly.

### 14.1 Extraction Accuracy on Ambiguous Aliases

Some aliases are never explicitly stated in the text — they're inferred by readers over time. The model may miss these or create false positive merges. **Mitigation:** Pass 2's `confidence` field. Low-confidence merges are stored but flagged for manual review.

### 14.2 Possession and Soul-Swap in Reverend Insanity

Fang Yuan swaps bodies multiple times across 2,300 chapters. The `possession` merge type handles the most common case, but simultaneous multi-body scenarios (Fang Yuan controlling puppets while inhabiting a host) may not map cleanly to the binary merge model. **Mitigation:** This is explicitly out of scope for v1. Reverend Insanity is a stress test, not a v1 requirement.

### 14.3 Non-English Webnovels / MTL Quality

Many webnovels are machine-translated. MTL text has inconsistent character name romanizations (same character spelled 3 different ways across chapters). The extraction model will create duplicate nodes. **Mitigation:** Name normalization pass in the parser; fuzzy matching in node resolution.

### 14.4 Context Window at Late-Book Chapters

For a character like Klein Moretti at chapter 1,300, even the tiered fact retrieval may struggle to fit all relevant context in 1,200 tokens. **Mitigation:** Tiered importance scoring on facts — `major` importance facts always included, `minor` facts dropped when over budget.

---

## 15. Development Setup

### 15.1 Prerequisites

- Python 3.11+
- Node.js 20+
- Groq API key (get at console.groq.com)
- Anthropic API key (optional, for higher-accuracy ingestion)

### 15.2 Repository Structure

```
plotwise/
├── api/                        ← FastAPI backend
│   ├── main.py
│   ├── ingestion/
│   │   ├── epub_parser.py
│   │   ├── pass1_extractor.py
│   │   ├── pass2_merger.py
│   │   └── embedder.py
│   ├── query/
│   │   ├── spoiler_gate.py
│   │   ├── identity_resolver.py
│   │   ├── context_builder.py
│   │   └── groq_client.py
│   ├── db/
│   │   ├── schema.sql
│   │   ├── migrations/
│   │   └── models.py
│   ├── routers/
│   │   ├── books.py
│   │   ├── query.py
│   │   ├── dossier.py
│   │   └── reader.py
│   └── requirements.txt
│
├── reader/                     ← Flow fork (Next.js PWA)
│   ├── src/
│   │   ├── components/
│   │   │   ├── PlotWiseSidebar/
│   │   │   ├── WhoIsThisPopover/
│   │   │   └── DossierPanel/
│   │   ├── context/
│   │   │   └── ReaderContext.tsx
│   │   └── hooks/
│   │       └── usePlotWise.ts
│   └── ... (rest of Flow's structure)
│
├── data/                       ← Runtime data (gitignored)
│   ├── books/
│   │   └── {book_id}/
│   │       ├── plotwise.db
│   │       └── {book_id}.faiss
│   └── epubs/
│
└── PLOTWISE_SYSTEM_DESIGN.md   ← this file
```

### 15.3 Running Locally

```bash
# Backend
cd api
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# Frontend
cd reader
npm install
npm run dev    # runs at localhost:3000

# Ingest a book (run once)
python -m ingestion.run --epub path/to/book.epub --book-id "lord-of-the-mysteries"
```

### 15.4 Environment Variables

```
# api/.env
GROQ_API_KEY=gsk_...
ANTHROPIC_API_KEY=sk-ant-...   # optional, for higher-accuracy pass 2
DATA_DIR=../data
DB_PATH=../data/books/{book_id}/plotwise.db
```

---

## 16. Project File Structure

Start with this build order. Each step is a testable checkpoint.

|Step|Deliverable|Test|
|---|---|---|
|1|`epub_parser.py` — chapter extraction|Parse LotM, verify chapter count and text quality|
|2|`schema.sql` + DB init|All tables created, FK constraints working|
|3|`pass1_extractor.py` — entity extraction|Run on 10 chapters, review JSON output quality|
|4|`pass2_merger.py` — identity resolution|Verify Klein/Gehrman merge detected correctly|
|5|`embedder.py` + FAISS index|Semantic search returns relevant chapters|
|6|`spoiler_gate.py` — gated DB queries|Unit test: no facts returned above reader chapter|
|7|`identity_resolver.py` — merge cluster|Unit test: cluster correct before and after reveal chapter|
|8|`/api/query/resolve` endpoint|E2E: "Who is Gehrman Sparrow?" at ch.50 vs ch.120|
|9|Flow fork + chapter progress hook|Chapter change fires progress update to API|
|10|PlotWise sidebar UI|Ask question → answer appears in sidebar|
|11|Dossier panel|Character cards render correctly at current chapter|

---

_This document is the single source of truth for PlotWise's architecture. Update it when design decisions change. Never let the code and this document diverge._