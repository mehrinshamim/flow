# PLOTWISE_PRODUCT_PITCH.md

# PlotWise: World's First Progress-Aware Narrative Intelligence Engine

## 1. Executive Summary
PlotWise transforms long-form reading from a passive activity into an interactive, stateful narrative experience. By replacing static, spoiler-ridden fan wikis with a progress-locked local intelligence layer, PlotWise eliminates cognitive fatigue for readers of massive, serialized fiction (webnovels, light novels, and epic fantasy series).

---

## 2. The Problem: The Cognitive Load Crisis
Modern fiction consumption is shifting heavily toward massive, serialized formats spanning hundreds or thousands of chapters (e.g., *Lord of the Mysteries*, *Omniscient Reader's Viewpoint*). 
* **Cognitive Fatigue:** Readers frequently pause a series for a few weeks, only to realize they have forgotten intricate political subplots, character aliases, or faction structures.
* **The Wiki Landmine:** Attempting to look up a character on a public wiki instantly spoils major plot points, deaths, or hidden identities from later chapters, destroying the reader's immersion and emotional continuity.
* **Reader Churn:** This friction causes high user churn. Readers abandon excellent stories simply because the mental overhead of catching up is too high.

---

## 3. The Solution: The Progress-Aware Vault
PlotWise acts as a local cache for narrative context. It physically locks the AI’s memory boundary to the user’s current page or chapter index, guaranteeing a 100% zero-spoiler conversational environment.

### The Three Commercial Pillars
* **Pillar 1: The Contextual Memory Vault (The "Who is This?" Engine):** Readers can highlight any name or ask a question in real-time. PlotWise resolves aliases, hidden identities, and past actions strictly bounded by what has been read so far.
* **Pillar 2: The Evolving Dossier (Stateful Progression Maps):** A clean dashboard of characters, factions, and locations that updates automatically as chapters are marked complete. It functions as a living, personalized wiki.
* **Pillar 3: Adaptive Arc Summarization:** Users returning from a reading hiatus can request target summaries: *"Remind me what happened in the political arc leading up to where I am right now,"* generating an instant, spoiler-free recap.

---

## 4. The Technical Moat (Why Generic LLMs Fail)
If a reader copies and pastes a chapter into a generic LLM UI, the system fails for two reasons:
1. **Lack of Historical Context:** The model doesn't know what happened in the previous 50 or 100 chapters without hitting extreme context window costs.
2. **Hallucinated Internet Spoilers:** If the model has pre-trained internet knowledge of the book, it will accidentally leak end-of-book spoilers into early-chapter answers.

PlotWise owns the end-to-end extraction pipeline: **EPUB Parser $\rightarrow$ Chapter-Aware Token Tracking $\rightarrow$ Coordinate-Filtered Context Space.** This vertical integration makes the experience impossible to replicate in a standard chat prompt.

---

## 5. Monetization Strategy
* **Freemium SaaS Model:** A free tier allows 1 active tracked book. Premium tiers ($5–$9/month) unlock unlimited library slots, deep forensic arc analysis, and cross-device reading sync.
* **B2B Publishing Integration:** A long-term licensing opportunity to embed the "Spoiler-Gate API" directly into massive native webnovel reading platforms to increase reader retention on long-running monetization arcs.