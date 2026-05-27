# Understand-Anything Skills Reference

This guide documents all the Understand-Anything plugin skills with their prompts, instructions, and file locations. You can ask me to run any of these workflows instead of using the Copilot Chat commands directly.

---

## 📋 Quick Reference Table

| Skill | Command | Purpose | Requires Graph? |
|-------|---------|---------|---|
| **understand** | `/understand` | Analyze codebase → produces `knowledge-graph.json` | ❌ No |
| **understand-dashboard** | `/understand-dashboard` | Interactive visual knowledge graph | ✅ Yes |
| **understand-onboard** | `/understand-onboard` | Auto-generated architecture walkthrough | ✅ Yes |
| **understand-explain** | `/understand-explain <path>` | Deep-dive explanation of a component | ✅ Yes |
| **understand-chat** | `/understand-chat <query>` | Ask free-form questions about code | ✅ Yes |
| **understand-domain** | `/understand-domain [--full]` | Extract business domain flows | ❌ No* |
| **understand-diff** | `/understand-diff` | Analyze git diffs & impact | ✅ Yes |

> *Can work standalone with lightweight scan, or derive from existing graph

---

## 🗂️ File Locations

All skill files are located at:
```
~/.understand-anything/repo/understand-anything-plugin/skills/
```

Individual skill paths:
```
understand/
  ├── SKILL.md (main prompt)
  ├── compute-batches.mjs
  ├── extract-import-map.mjs
  ├── scan-project.mjs
  ├── merge-batch-graphs.py
  └── ...

understand-dashboard/
  └── SKILL.md

understand-onboard/
  └── SKILL.md

understand-explain/
  └── SKILL.md

understand-chat/
  └── SKILL.md

understand-domain/
  └── SKILL.md

understand-diff/
  └── SKILL.md
```

Graph files generated in your project:
```
<project-root>/.understand-anything/
├── knowledge-graph.json          (main analysis output)
├── domain-graph.json             (business flows)
├── diff-overlay.json             (changed components)
└── config.json                   (settings: language, auto-update)
```

---

## 1️⃣ `/understand` — Full Codebase Analysis

**Purpose:** Scan the entire codebase and produce a structured knowledge graph.

**Output:** `.understand-anything/knowledge-graph.json`

### Arguments

```
/understand [path] [--full|--auto-update|--no-auto-update|--review|--language <lang>]
```

| Argument | Effect |
|----------|--------|
| `--full` | Force full rebuild, ignore cached graphs |
| `--auto-update` | Enable automatic updates on commit |
| `--no-auto-update` | Disable auto-updates |
| `--review` | Run full LLM review instead of inline validation |
| `--language <lang>` | Generate descriptions in ISO 639-1 code (e.g., `zh`, `ja`, `ko`, `en`, `es`, `fr`, `de`) |
| `<path>` | Analyze a specific directory instead of current |

### Example Prompts for Me

- *"Run the `/understand` analysis on the Flow reader app"*
- *"Analyze `/home/mehrin/repo/_gsoc/flow` with full rebuild and language set to English"*
- *"Generate a knowledge graph for the entire Flow monorepo"*

### What It Produces

The knowledge graph contains:
```json
{
  "project": {
    "name": "flow",
    "description": "Open-source ePub reader",
    "languages": ["TypeScript", "JavaScript"],
    "frameworks": ["Next.js", "React", "Recoil"],
    "analyzedAt": "2026-05-27T...",
    "gitCommitHash": "abc123..."
  },
  "nodes": [
    {
      "id": "file:apps/reader/src/components/Reader.tsx",
      "type": "file",
      "name": "Reader.tsx",
      "filePath": "apps/reader/src/components/Reader.tsx",
      "summary": "Main ePub reader component...",
      "tags": ["component", "ui", "reader"],
      "complexity": 8,
      "languageNotes": "Uses React hooks and Recoil state management"
    }
    // ... more nodes
  ],
  "edges": [
    {
      "source": "file:apps/reader/src/components/Reader.tsx",
      "target": "file:packages/epubjs/src/book.js",
      "type": "imports",
      "direction": "outgoing",
      "weight": 1.0
    }
    // ... more edges
  ],
  "layers": [
    {
      "id": "layer:ui",
      "name": "UI Components",
      "description": "React components for the reader interface",
      "nodeIds": ["file:apps/reader/src/components/..."]
    }
  ],
  "tour": [
    {
      "order": 1,
      "title": "Project Setup",
      "description": "Understanding the monorepo structure",
      "nodeIds": ["file:pnpm-workspace.yaml", "file:turbo.json"]
    }
  ]
}
```

---

## 2️⃣ `/understand-dashboard` — Interactive Visual Graph

**Purpose:** Launch a Vite dev server to visualize the knowledge graph in a browser dashboard.

**Requires:** `.understand-anything/knowledge-graph.json` (run `/understand` first)

### Arguments

```
/understand-dashboard [project-path]
```

| Argument | Effect |
|----------|--------|
| `[project-path]` | Point dashboard at a specific project (default: current dir) |

### What It Does

1. Verifies the knowledge graph exists
2. Resolves the plugin dashboard code location
3. Installs dependencies (`pnpm install`)
4. Starts Vite dev server on port 5173 (or next available)
5. Generates a **tokenized URL** with access token (required to view data)
6. Opens the dashboard in your default browser

### Dashboard Features

- **Pan & zoom** across the knowledge graph
- **Search** nodes by name, type, or tag
- **Click nodes** to see detailed metadata (summary, complexity, dependencies)
- **Follow edges** to trace imports, calls, and dependencies
- **Filter by layer** to focus on architectural sections
- **Diff overlay** (if `/understand-diff` was run) highlights changed/affected nodes

### Example Prompts for Me

- *"Launch the Understand-Anything dashboard for the Flow project"*
- *"Start the interactive knowledge graph viewer"*
- *"I want to visualize the reader app architecture"*

### URL Example

```
http://127.0.0.1:5173?token=sk-view-abc123def456...
```

⚠️ **Important:** The token is required! Without it, you'll see an "Access Token Required" gate.

---

## 3️⃣ `/understand-onboard` — Guided Architecture Walkthrough

**Purpose:** Generate a comprehensive onboarding guide for new team members.

**Requires:** `.understand-anything/knowledge-graph.json`

**Output:** Markdown guide (suggested save: `docs/ONBOARDING.md`)

### What It Does

1. Reads project metadata (name, languages, frameworks)
2. Extracts architectural layers from the graph
3. Identifies file-level structural nodes (skips function/class details)
4. Finds complexity hotspots
5. Generates guide with sections:
   - **Project Overview** — tech stack, description
   - **Architecture Layers** — layers and key files per layer
   - **Key Concepts** — patterns, design decisions
   - **Guided Tour** — step-by-step walkthrough
   - **File Map** — what each key file does
   - **Complexity Hotspots** — areas to approach carefully

### Example Prompts for Me

- *"Generate an onboarding guide for new Flow developers"*
- *"Create a walkthrough of the reader app's architecture"*
- *"I want a high-level tour of the codebase without deep function details"*

### Sample Output Structure

```markdown
# Onboarding Guide for Flow

## Project Overview
Flow is an open-source ePub reader built with Next.js, React, and TypeScript.

**Tech Stack:**
- Languages: TypeScript, JavaScript
- Frameworks: Next.js 12, React 18, Recoil (state)
- Packages: Tailwind CSS, React Icons, SWR

## Architecture Layers

### Layer 1: UI Components (`src/components/`)
- Reader.tsx — Main reader viewport
- HighlightsMenu.tsx — Highlight management UI
- ...

### Layer 2: State Management (`src/hooks/`, Recoil atoms)
- useReaderState — Central reader state
- ...

### Layer 3: Data Persistence (`db.ts`, Dexie)
- Local IndexedDB for annotations, bookmarks
- Cloud sync via Dropbox integration

## Guided Tour
1. Start with monorepo structure
2. Explore the reader app entry point
3. Walk through a highlight flow (create → save → sync)
4. Review the epub.js engine integration
```

---

## 4️⃣ `/understand-explain` — Deep-Dive Component Analysis

**Purpose:** Get a thorough, in-depth explanation of a specific file, function, or module.

**Requires:** `.understand-anything/knowledge-graph.json`

### Arguments

```
/understand-explain [file-path|function-notation]
```

| Format | Example |
|--------|---------|
| File path | `src/components/Reader.tsx` |
| Function notation | `src/components/Reader.tsx:useHighlights` |
| Full path | `/home/mehrin/repo/_gsoc/flow/apps/reader/src/db.ts` |

### What It Does

1. Searches the graph for the target component
2. Finds all connected edges (imports, calls, dependents)
3. Reads the source file for deep analysis
4. Explains in context:
   - Role in architecture (which layer, why it exists)
   - Internal structure (functions, classes it contains)
   - External connections (imports, what calls it)
   - Data flow (inputs → processing → outputs)
   - Patterns & complexity worth understanding

### Example Prompts for Me

- *"Explain how the highlight system works in `/apps/reader/src/annotation.ts`"*
- *"Give me a deep dive into the reader's state management hook"*
- *"I want to understand the epub.js Book.js module and how it integrates with the reader"*
- *"What does `db.ts` do and what depends on it?"*

### Sample Output

```
## Component: Reader.tsx
**Location:** apps/reader/src/components/Reader.tsx
**Type:** React Component
**Complexity:** High (8/10)
**Layer:** UI Components

### What It Does
The Reader component is the main viewport for displaying EPUB books. It manages:
- Book rendering via epub.js
- Annotation overlays
- Navigation controls
- Search highlighting

### Internal Structure
**Functions:**
- `Reader()` — Main component
- `renderBook()` — Initialize book in viewport
- `handleLocationChange()` — Navigate chapters

**State:**
- Uses Recoil for global reader state
- Local state for UI controls (search, annotations)

### Connections
**Imports:**
- `@flow/epubjs` — Book rendering engine
- `useReaderState()` — Recoil atom
- `useHighlights()` — Custom hook for annotation

**Imported By:**
- `pages/read.tsx` — Route handler
- `components/AppLayout.tsx` — Main layout

### Data Flow
1. **Input:** Book file (EPUB) + chapter index
2. **Processing:** epub.js renders to DOM, overlays annotations
3. **Output:** Interactive reader UI with navigation, search, highlights
```

---

## 5️⃣ `/understand-chat` — Ask Free-Form Questions

**Purpose:** Ask natural-language questions about the codebase and get context-aware answers.

**Requires:** `.understand-anything/knowledge-graph.json`

### Arguments

```
/understand-chat <query>
```

The query can be anything you'd normally ask a developer.

### What It Does

1. Searches the graph for relevant nodes (by name, summary, tags)
2. Traces connected edges (dependencies, callers)
3. Identifies architectural layers involved
4. Answers your question using the subgraph
5. References specific files, functions, and relationships

### Example Prompts for Me

- *"How does the highlight system work?"*
- *"What packages does the reader app depend on?"*
- *"Where is the cloud sync code?"*
- *"Show me all files that interact with the epub.js library"*
- *"What's the difference between the reader and website apps?"*
- *"Which component handles book search?"*
- *"How does annotation data flow from the UI to the database?"*

### Sample Output

```
## How does the highlight system work?

**Components involved:**
1. **UI Layer:** HighlightsMenu.tsx, AnnotationButton.tsx
2. **State:** useHighlights() hook (Recoil)
3. **Data:** annotation.ts (models), db.ts (persistence)
4. **Sync:** sync.ts (cloud storage)

**Flow:**
1. User selects text → HighlightsMenu shows color/note options
2. Action dispatched to Recoil state via useHighlights()
3. Highlight stored locally in IndexedDB (db.ts)
4. If cloud sync enabled → Dropbox integration (sync.ts)

**Key Files:**
- apps/reader/src/annotation.ts — Highlight/annotation models
- apps/reader/src/components/HighlightsMenu.tsx — UI
- apps/reader/src/db.ts — Local storage via Dexie
- apps/reader/src/sync.ts — Cloud sync logic
```

---

## 6️⃣ `/understand-domain` — Business Domain Flows

**Purpose:** Extract business domain knowledge and generate an interactive domain flow graph.

**Output:** `.understand-anything/domain-graph.json`

### Arguments

```
/understand-domain [--full]
```

| Argument | Effect |
|----------|--------|
| `--full` | Force fresh scan even if graph exists |
| (none) | Derive from existing knowledge-graph.json (cheap) |

### What It Does

#### Path 1: Lightweight Scan (no existing graph)
1. Scans file tree (respects `.gitignore`)
2. Detects entry points (HTTP routes, CLI commands, event handlers)
3. Extracts exports/imports
4. Collects code snippets

#### Path 2: Derive from Graph (existing knowledge-graph.json)
1. Reads the knowledge graph
2. Formats as structured context

#### Both Paths Then:
1. Dispatches domain-analyzer agent
2. Validates output
3. Saves to `domain-graph.json`
4. Auto-launches dashboard with domain view

### Business Domain Concepts

- **Domain** — Business area (e.g., "Annotation", "Cloud Sync")
- **Flow** — Process sequence (e.g., "Highlight Creation Flow")
- **Step** — Individual action (e.g., "User selects text", "Save to database")

### Example Prompts for Me

- *"Extract the business domain flows from the Flow reader app"*
- *"Show me all the business processes in the codebase"*
- *"Generate a domain graph to visualize user workflows"*
- *"Force a fresh domain analysis with --full flag"*

### Sample Domain Graph Structure

```json
{
  "domains": [
    {
      "id": "domain:annotation",
      "name": "Annotation & Highlights",
      "description": "User-created highlights, notes, and annotations on books",
      "flows": ["domain:annotation:creation", "domain:annotation:sync"]
    }
  ],
  "flows": [
    {
      "id": "flow:highlight-creation",
      "name": "Highlight Creation",
      "steps": [
        "step:select-text",
        "step:open-menu",
        "step:choose-color",
        "step:save-highlight",
        "step:sync-cloud"
      ]
    }
  ]
}
```

---

## 7️⃣ `/understand-diff` — Analyze Code Changes

**Purpose:** Analyze git diffs and show what changed, what's affected, and risk assessment.

**Requires:** `.understand-anything/knowledge-graph.json`

**Output:** `.understand-anything/diff-overlay.json` (for dashboard visualization)

### What It Does

1. Gets changed files (`git diff` or PR)
2. Finds matching nodes in the graph
3. Traces 1-hop connections (upstream callers, downstream dependencies)
4. Identifies affected architectural layers
5. Provides risk assessment based on:
   - Component complexity
   - Cross-layer edge count
   - Blast radius (# affected components)
6. Writes diff overlay for dashboard

### Example Prompts for Me

- *"Analyze what code changed and what might be affected"*
- *"Show me the impact of my recent changes"*
- *"I want to see a pre-commit diff analysis"*
- *"What components might break from my changes?"*

### Sample Output

```
## Diff Impact Analysis

**Changed Components:**
- apps/reader/src/annotation.ts (complex: 7/10)
  → Highlight model additions

- apps/reader/src/components/HighlightsMenu.tsx (complex: 5/10)
  → New UI for color picker

**Affected Components (1-hop):**
- apps/reader/src/db.ts (imports annotation.ts)
- apps/reader/src/pages/read.tsx (imports HighlightsMenu)
- apps/reader/src/hooks/useHighlights.ts (depends on annotation.ts)

**Affected Layers:**
- UI Components (HighlightsMenu changes)
- State Management (useHighlights hook affected)
- Data Persistence (annotation model propagates to db.ts)

**Risk Assessment:** MEDIUM
- Medium complexity changes
- 3 downstream dependents affected
- Cross-layer impact (UI → State → Data)
- **Recommended Review:** useHighlights hook & db.ts integration tests
```

The dashboard will then visualize the changed nodes (highlighted) and affected nodes (dimmed but connected).

---

## 📊 Graph Structure Reference

All skills that use the knowledge graph work with this structure:

### Nodes
```json
{
  "id": "file:apps/reader/src/db.ts",
  "type": "file|function|class|module|concept|config|document|service|...",
  "name": "db.ts",
  "filePath": "apps/reader/src/db.ts",
  "summary": "IndexedDB wrapper for local annotation storage...",
  "tags": ["database", "persistence", "local-storage"],
  "complexity": 6,
  "languageNotes": "Uses Dexie ORM with TypeScript"
}
```

### Edges
```json
{
  "source": "file:apps/reader/src/components/Reader.tsx",
  "target": "file:packages/epubjs/src/book.js",
  "type": "imports|contains|calls|depends_on|configures|documents|...",
  "direction": "outgoing|bidirectional|incoming",
  "weight": 1.0
}
```

### Layers
```json
{
  "id": "layer:ui-components",
  "name": "UI Components",
  "description": "React components for the reader interface",
  "nodeIds": ["file:apps/reader/src/components/Reader.tsx", "..."]
}
```

---

## 🚀 Quick Tips for Asking Me

When requesting analysis, be specific:

❌ *"Analyze the code"*
✅ *"Run `/understand` on the Flow monorepo and generate a knowledge graph"*

❌ *"Explain something"*
✅ *"Use `/understand-explain` on `apps/reader/src/db.ts` to show me how annotations are persisted"*

❌ *"Show me what changed"*
✅ *"Run `/understand-diff` to analyze the impact of changes to the highlight system"*

---

## 📝 Example Full Workflow

```bash
# Step 1: Generate the knowledge graph (one-time setup)
> "Run /understand on the Flow project with language set to English"

# Step 2: Explore the architecture
> "Launch the /understand-dashboard to visualize the graph"

# Step 3: Get context on new areas
> "Use /understand-explain to deep-dive into apps/reader/src/sync.ts"

# Step 4: Ask specific questions
> "Use /understand-chat to answer: How does cloud sync work for annotations?"

# Step 5: Generate onboarding docs
> "Run /understand-onboard to create an architecture walkthrough"

# Step 6: Analyze changes
> "Run /understand-diff to see what's affected by my recent changes"

# Step 7: Extract business flows
> "Run /understand-domain to visualize the highlight creation flow"
```

---

## 🔗 Related Files

- Main plugin repo: `~/.understand-anything/repo/`
- Skills directory: `~/.understand-anything/repo/understand-anything-plugin/skills/`
- Agent prompts: `~/.understand-anything/repo/understand-anything-plugin/agents/`
- Generated outputs: `<project-root>/.understand-anything/`

---

**Happy analyzing! 🚀**
