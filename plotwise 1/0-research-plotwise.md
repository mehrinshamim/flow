Here's a comprehensive research deep-dive across both topics, stitched together with the underlying architecture patterns that connect them.

---

## 🧠 Part 1: Long-Context Memory Systems for Narrative Agents

### The Core Problem

As input sequences grow to thousands or even millions of tokens, performance degrades on tasks requiring entity tracking, logical consistency, or recall of task-relevant facts across long interaction histories. The core difficulty lies not only in how much information the model can observe, but in how information is selectively retained, retrieved, and integrated during inference.

This is especially acute for _narrative_ agents — agents that must track plot state, character relationships, and causal chains across a book, game, or long conversation.

---

### The Memory Taxonomy

The field has converged on a **four-store cognitive model** mirroring human memory:

Cognitive Architectures for Language Agents propose a generalized blueprint where working, episodic, semantic, and procedural stores interact through a central executive (the LLM), directly echoing Baddeley's model. The Achilles' heel of hierarchical memory is orchestration.

Working memory constraints in individual agents necessitate external memory architectures that support both episodic recall of specific interactions and semantic abstraction of learned patterns.

|Store|What it holds|Narrative role|
|---|---|---|
|**Working**|Current context window|Active scene, current dialogue|
|**Episodic**|Timestamped events, "what happened"|Chapter-level plot tracking|
|**Semantic**|Abstracted facts, "how things work"|Character traits, world rules|
|**Procedural**|Skills / action patterns|Genre conventions, narrator style|

---

### Key Architectures

**SEEM — Structured Episodic Event Memory**

SEEM is a hierarchical framework that synergizes a graph memory layer for relational facts with a dynamic episodic memory layer for narrative progression. Grounded in cognitive frame theory, SEEM transforms interaction streams into structured Episodic Event Frames anchored by precise provenance pointers, with an associative fusion and Reverse Provenance Expansion mechanism to reconstruct coherent narrative contexts from fragmented evidence. Experimental results demonstrate that SEEM significantly outperforms baselines, enabling agents to maintain superior narrative coherence and logical consistency.

**ReadAgent — Gist Memory**

ReadAgent is a human-inspired reading agent that uses "gist memory" of very long contexts. Evaluated on QuALITY, NarrativeQA, and QMSum, ReadAgent outperforms baselines while extending the effective context window by 3.5–20×.

**LSTM-MAS — Chained Multi-Agent Memory**

LSTM-MAS organizes agents in a chained architecture where each node comprises a worker agent for segment-level comprehension, a filter agent for redundancy reduction, a judge agent for error detection, and a manager agent for globally regulating information propagation — analogous to LSTM's input gate, forget gate, error carousel, and output gate. These designs enable controlled information transfer and selective long-term dependency modeling across textual segments, avoiding error accumulation and hallucination propagation.

**H-MEM — Hierarchical Position-Indexed Memory**

H-MEM uses hierarchical memory and a position index to search layer by layer, effectively removing the influence of irrelevant memories on calculation — in contrast to traditional flat mechanisms that compute similarity across all stored memories at once.

---

### Long-Context vs. Long-Term Memory

An important distinction worth noting for system design:

Long-context and long-term-memory are complementary infrastructures: the former mitigates some of the compression pressure that motivates summarization, but leaves the full memory lifecycle intact. A larger context window does not remove the need for provenance tagging, principal-scoped retrieval, rollbackable state, or verified forgetting — those arise whenever content outlives a single session.

---

### Storage Paradigm Landscape (2025)

Storage paradigms range from cumulative memory (complete historical appending) to reflective/summarized memory (periodically compressed summaries), purely textual, parametric (embedding into model weights via fine-tuning), and structured memory (tables, triples, or graph-based). Multi-component systems like MIRIX include Core, Episodic, Semantic, Procedural, Resource, and Knowledge Vault stores.

The RAG + multi-session approach also remains competitive: RAG offers a balanced compromise, combining the accuracy of short-context LLMs with the extensive comprehension of wide-context LLMs, and does particularly well when dialogues are transformed into a database of assertions (observations) about each speaker's life and persona.

---

## 📖 Part 2: Spoiler-Aware Multimodal Reading Companions

### The Spoiler Detection Problem

The classical ML approach: LSTM, BERT, and RoBERTa language models were explored for spoiler detection at the sentence-level using the UCSD Goodreads Spoiler dataset, with LSTM results slightly exceeding the previous UCSD team's handcrafted-feature baseline in spoiler detection.

Modern LLM-based enhancement: Research on "Enhancing the Performance of Spoiler Review Detection by a LLM with Hints" has been published, treating spoiler detection as a hint-augmented classification problem.

---

### Narrative Position Tracking — The Core Primitive

The most important recent paper for building a spoiler-aware companion is the **100-Endings metric**:

The 100-Endings metric walks through a story sentence by sentence: at each position, a model predicts how the story will end 100 times given only the text so far, and measures tension as how often predictions fail to match. Beyond the mismatch rate, the sentence-level curve yields complementary statistics such as inflection rate — a geometric measure of how frequently the curve reverses direction — tracking twists and revelations.

Measuring tension requires a position-level evaluation that tracks predictability across the entire arc — capturing where information is withheld, whether resolution is delayed, or whether the narrative resists prediction at each point in its progression.

This is exactly the primitive needed for a spoiler-aware companion: **a per-position model of what the reader can and cannot yet know**.

---

### Narrative Structure for Understanding

Narratology frames narrative as a system of interrelations among fundamental components — Style, Character, Event, and Setting — not as a mere sequence of words. Computational literary studies have drawn on these theoretical grounds for NLP research, modeling characters as agents and evaluating settings as spatial frames that shape interpretation.

---

## 🔧 Part 3: Architecture Blueprint — Spoiler-Aware Multimodal Reading Companion

Combining both research threads, here's how you'd build this:

```
┌─────────────────────────────────────────────────────┐
│               READING COMPANION SYSTEM               │
│                                                      │
│  ┌──────────────┐    ┌────────────────────────────┐ │
│  │ READER STATE │    │    BOOK KNOWLEDGE BASE      │ │
│  │              │    │                            │ │
│  │ current_page │    │  Full text (pre-indexed)   │ │
│  │ chapter_idx  │    │  Character graph (semantic) │ │
│  │ session_hist │    │  Plot timeline (episodic)  │ │
│  └──────┬───────┘    │  Themes/motifs (semantic)  │ │
│         │            └────────────┬───────────────┘ │
│         ▼                         ▼                  │
│  ┌──────────────────────────────────────────┐       │
│  │         SPOILER GATE (KEY LAYER)          │       │
│  │                                          │       │
│  │  - Reader position pointer               │       │
│  │  - Filter all retrieval to ≤ current_page│       │
│  │  - "Tension map" for safe hint depth     │       │
│  └──────────────────┬───────────────────────┘       │
│                     ▼                                │
│  ┌──────────────────────────────────────────┐       │
│  │       MULTIMODAL RESPONSE ENGINE          │       │
│  │  Text Q&A / Character maps / Timeline viz │       │
│  │  Audio (read-along) / Image (scene art)  │       │
│  └──────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────┘
```

### Key Design Decisions

**1. The Spoiler Gate is non-negotiable.** Every retrieval from the book knowledge base must be filtered against `reader.current_position`. You maintain two indices of the book: one for "what has happened" (≤ current page) and one for the full graph (never exposed).

**2. Memory tiers map directly to reading needs:**

- **Working memory** → current chapter context
- **Episodic memory** → reader's own reading history ("last week you were confused about X")
- **Semantic memory** → persistent character/world facts the reader has _encountered_
- **Procedural memory** → reader's question patterns, depth preference, spoiler sensitivity

**3. Multimodality means more spoiler surface area.** A character relationship graph generated at chapter 3 must look different than one generated at chapter 20. Images of a character's arc, timelines, and maps must all be generated only from reader-safe knowledge.

**4. ComoRAG / SEEM for narrative state.** ComoRAG — a Cognitive-Inspired Memory-Organized RAG for Stateful Long Narrative Reasoning — is directly relevant here, as it preserves narrative coherence across RAG retrievals, which is critical when answers span multiple chapters.

---

### Open Research Gaps (Good for original work)

- **Spoiler-safe graph generation** — no published work on dynamically masking knowledge graphs at a per-reader position. High novelty.
- **Reader affect modeling** — detecting confusion, surprise, or engagement from query patterns to adapt companion depth.
- **Tension-aware hint system** — using the 100-Endings position curve to decide _how much_ to hint toward an upcoming revelation without spoiling it.
- **Multimodal provenance** — when generating scene imagery or character maps, ensuring the visual output doesn't visually encode future-state information (e.g., a character shown with a weapon they haven't acquired yet).

This is a legitimately underexplored intersection — narrative agents have gotten memory research attention, and spoiler detection has gotten NLP attention, but the **reading companion that combines both with multimodal output** is essentially an open design problem.