# PLOTWISE_PRD.md

## 1. Project Overview
**PlotWise** is an AI-powered, spoiler-aware reading companion layered over a web-based EPUB reader. The goal is to help readers track massive narratives (like *Lord of the Mysteries*, *Omniscient Reader's Viewpoint*, or *The Beginning After The End*) by resolving aliases, summarizing complex political situations, and recalling past events—strictly without spoiling future chapters.

**The Base Repository:** We are forking `pacexy/flow`, an open-source, mobile-first Next.js PWA powered by `epub.js`. 
**The Goal:** We will leave Flow's core reading mechanics (pagination, offline storage, themes) intact. We are strictly injecting new UI components and wiring them to a custom Next.js API route that connects to Groq.

---

## 2. Core Architecture: The Progressive Ledger
To solve the AI spoiler problem without relying on complex dynamic graph databases, PlotWise uses a **Progressive Ledger** architecture:

* **Ingestion (Python/TinkerHub Backend):** A heavy, offline batch process parses the EPUB, extracts entities/events chapter by chapter, and generates a static JSON state file for every chapter (e.g., `chapter_45_state.json`).
* **Reading State (Next.js/`epub.js`):** The frontend tracks the user's progress. As the user reads, the app resolves their current `epub.js` Canonical Fragment Identifier (CFI) into an integer: `currentChapter`.
* **Inference (Groq API):** When the user asks a question, the frontend sends the user's query AND the specific `chapter_{currentChapter}_state.json` ledger to the Groq API. The LLM uses only this snapshot to answer, guaranteeing zero future knowledge leakage.

---

## 3. UI/UX Implementation Specifications
We will maintain Flow's existing Tailwind CSS variables for seamless dark/light mode compatibility. The only new color introduced is an **Amber/Gold (`amber-500`)** accent color specifically reserved for PlotWise AI elements to differentiate them from standard UI elements.

### Feature A: The Context Menu Injection ("Who is this?")
* **Location:** Inside the existing text-highlight popover menu provided by Flow.
* **Design:** Next to the native "Copy" and "Highlight" buttons, inject a new button labeled "Ask PlotWise" with an amber sparkle icon (`✨`).
* **Behavior:** When clicked, it captures the highlighted text, saves it to a global state (e.g., `activeAIQuery`), closes the popover, and automatically triggers the AI Chat Bottom Sheet to slide up.

### Feature B: The AI Chat Assistant (Bottom Sheet)
* **Location:** A floating component anchored to the bottom of the viewport. Also accessible via an Amber Floating Action Button (FAB) in the reading view.
* **Design:** A drag-to-close Bottom Sheet (`framer-motion`) covering 60% of the screen height. It features a slight backdrop blur to keep the book text visible underneath. 
* **Components:**
  * A scrollable message history area displaying user queries and AI responses (markdown supported).
  * A fixed bottom text input bar with a submit arrow.
  * Loading state: A subtle, pulsing amber glow or typing indicator while waiting for the Groq API.
* **Behavior:** If opened via the Context Menu (Feature A), the input field is pre-populated with the highlighted text.

### Feature C: The World State Tracker
* **Location:** Inside the main side navigation drawer (where the Table of Contents currently lives).
* **Design:**袋 Implement a segmented control or tab system at the top of the drawer: `[ Contents | World State ]`.
* **Behavior:** When the "World State" tab is active, the app fetches the JSON ledger for the `currentChapter`. It renders the data as an accordion list or clean cards divided by:
  * **Characters:** (e.g., *Gehrman Sparrow: A crazy adventurer persona created by Klein.*)
  * **Factions:** (e.g., *Church of the Fool.*)
  * **Locations:** (e.g., *Bansy Harbor.*)

---

## 4. State Management & API Wiring
Flow uses React hooks and Zustand/Context for state. You must implement the following connections:

### A. Location Tracking
1. Hook into the `epub.js` rendition object (`rendition.on('relocated', ...)`).
2. Extract the current TOC chapter index based on the active CFI.
3. Store this in a global state store: `usePlotWiseStore((state) => state.currentChapter)`.

### B. The Next.js API Proxy Route (`app/api/chat/route.ts`)
Create a serverless function to handle the Groq API call.
* **Input:** `POST` request containing `{ messages: [], currentChapter: number, bookId: string }`.
* **Logic:**
  1. Fetch the corresponding `chapter_{currentChapter}_state.json` file from the `/public/ledgers` directory.
  2. Inject this JSON blob into the `system` prompt of the LLM.
  3. Send the payload to Groq (`Llama-3.3-70b-versatile`).
* **Output:** Stream the text response back to the frontend Chat Bottom Sheet.

---

## 5. Data Schema Contract (The Chapter Ledger)
The Python ingestion script will generate chapter JSON ledgers matching this exact schema. The frontend must expect this structure when rendering the "World State" tab.

```json
{
  "book_id": "lotm_volume_2",
  "chapter_index": 124,
  "chapter_title": "A Crazy Adventurer",
  "entities": [
    {
      "name": "Gehrman Sparrow",
      "aliases": ["Klein Moretti", "Sherlock Moriarty"],
      "faction": "The Tarot Club",
      "status": "Alive",
      "summary": "A ruthless adventurer persona adopted by Klein to hunt pirates and digest the Faceless potion."
    }
  ],
  "factions": [
    {
      "name": "Church of the Sea God",
      "description": "A local religion in the Rorsted Archipelago. Klein recently acquired the Sea God Scepter."
    }
  ],
  "plot_summary_so_far": "Klein has left Backlund after the Great Smog and is now traveling the Sonia Sea to acquire mermaid blood and digest his potion..."
}


