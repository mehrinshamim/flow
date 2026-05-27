# PERSONAL_USE_CASE_STRATEGY.md

# PlotWise: Custom Implementation & Compute Strategy

## 1. Target Masterpieces (The "Final Boss" Use Case)
This implementation of PlotWise is optimized to track dense, massive narrative structures with heavy alias usage, shifting power systems, and expansive world-building. Initial testing focuses on three primary webnovel series:
* ***Lord of the Mysteries* (1,400+ Chapters):** Tracking complex Beyonder pathway formulas, secret organizations (The Tarot Club), and major character alias shifts (e.g., ensuring the system knows *Gehrman Sparrow* or *Sherlock Moriarty* is *Klein Moretti* without exposing future sequence advancements).
* ***Omniscient Reader's Viewpoint* (550+ Chapters):** Managing intricate constellation modifiers (e.g., *Demon King of Salvation*), scenario conditions, and sprawling character networks.
* ***The Beginning After The End* (500+ Chapters):** Tracking geographical factions, core asura conflicts, and protagonist power evolution.

---

## 2. Core Architecture: The Hybrid Compute Model
To maximize real-time query speeds while avoiding rate limits and high API costs, the system splits computation between local pipeline ingestion and high-speed cloud inference.

```
┌──────────────────────────────────────────────────────────────────┐
│                     HYBRID COMPUTE PIPELINE                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [ EPUB File ] ──► ( TinkerHub Offline Ingestion )               │
│                            │                                     │
│                            ▼                                     │
│                    [ Chapter Ledgers ]                           │
│                    (Static JSON Folder)                          │
│                            │                                     │
│                            ▼                                     │
│  [ User Query ] ──► ( Next.js Frontend Reader )                  │
│                            │                                     │
│                            ▼                                     │
│                     ( Groq Cloud API ) ──► [ Millisecond Q&A ]   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

```

### Step 1: The Ingestion Engine (Powered by TinkerHub)
* **Goal:** Heavy background processing to read the EPUB files and extract state sequentially.
* **Tech Stack:** Heavy open-source models (like Llama 3 70B or Mixtral) hosted via TinkerHub compute resources. 
* **Behavior:** Runs as an offline batch script. It parses Chapter 1 $\rightarrow$ generates state. It then parses Chapter 2 using Chapter 1's output as baseline context $\rightarrow$ saves updated state.
* **Output:** A flat folder of static, chapter-isolated JSON files (e.g., `orv_chapter_45_state.json`) containing clean, pre-summarized character states and plot ledgers.

### Step 2: The Runtime Assistant (Powered by Groq)
* **Goal:** Instantaneous, millisecond-level conversational retrieval during an active reading session.
* **Tech Stack:** Groq API running high-speed inference models (`Llama-3.3-70b-versatile` or `Llama-3.1-8b-instant`).
* **Behavior:** When the user enters a prompt at Chapter 45, the app grabs the pre-built `orv_chapter_45_state.json` ledger file and appends it to the system prompt context. Groq evaluates this thin, highly structured payload instantly, avoiding heavy database compute or runtime graph traversals.

---

## 3. User Experience & Form Factor Philosophy
* **Skip the IDE Plugin:** Webnovels are personal entertainment consumed casually on phones, tablets, or in bed. Forcing reading execution inside an IDE or development environment breaks user immersion and guarantees project abandonment.
* **PWA Deployment:** Built entirely as a web-accessible Progressive Web App (PWA) by forking the browser-optimized `pacexy/flow` reader. This allows cross-platform installation on mobile home screens with a fully responsive touch UI.

---

## 4. Developer Workflow: AI-Assisted Frontend
As a backend-focused engineer, frontend development tasks are heavily offloaded to state-of-the-art AI coding tools (Cursor, v0.dev, Claude) to accelerate deployment.

* **Backend Focus:** Maintain complete control over data ingestion, text parsing algorithms, context window structuring, and the JSON state architecture.
* **Frontend Offloading Strategy:** Provide generative AI tools with structured UI mock prompts based on Flow's existing Tailwind CSS system to build out the draggable chat bottom sheets, highlight popovers, and side dashboard drawers automatically.