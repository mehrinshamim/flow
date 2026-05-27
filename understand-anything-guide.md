# Understand-Anything — VS Code + GitHub Copilot Setup Guide

> **Understand-Anything** turns any codebase into an interactive knowledge graph you can explore, search, and ask questions about.
>
> 🔗 [GitHub Repository](https://github.com/Lum1104/Understand-Anything) · [Live Demo](https://understand-anything.com/demo/) · [Discord](https://discord.gg/pydat66RY)

---

## 📦 Installation

The plugin was installed via the unified installer script targeting the `vscode` platform:

```bash
curl -fsSL https://raw.githubusercontent.com/Lum1104/Understand-Anything/main/install.sh | bash -s vscode
```

This cloned the repository to `~/.understand-anything/repo` and symlinked individual skills into `~/.copilot/skills/`, where GitHub Copilot Chat auto-discovers them.

### Installed skill symlinks

| Symlink | Target |
|---|---|
| `~/.copilot/skills/understand` | `~/.understand-anything/repo/understand-anything-plugin/skills/understand` |
| `~/.copilot/skills/understand-chat` | `…/skills/understand-chat` |
| `~/.copilot/skills/understand-dashboard` | `…/skills/understand-dashboard` |
| `~/.copilot/skills/understand-diff` | `…/skills/understand-diff` |
| `~/.copilot/skills/understand-domain` | `…/skills/understand-domain` |
| `~/.copilot/skills/understand-explain` | `…/skills/understand-explain` |
| `~/.copilot/skills/understand-knowledge` | `…/skills/understand-knowledge` |
| `~/.copilot/skills/understand-onboard` | `…/skills/understand-onboard` |

### Updating

To pull the latest version without reinstalling:

```bash
curl -fsSL https://raw.githubusercontent.com/Lum1104/Understand-Anything/main/install.sh | bash -s -- --update
```

### Uninstalling

```bash
curl -fsSL https://raw.githubusercontent.com/Lum1104/Understand-Anything/main/install.sh | bash -s -- --uninstall vscode
```

---

## 🚀 Getting Started in VS Code

### Prerequisites

- **VS Code** with the **GitHub Copilot Chat** extension installed and active.
- An active **GitHub Copilot subscription** (Individual, Business, or Enterprise).

### Step 1 — Reload VS Code

After installation, reload the VS Code window so Copilot picks up the new skills:

1. Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on macOS)
2. Type **"Developer: Reload Window"** and hit Enter

### Step 2 — Open your project

Open the project you want to analyze. For the Flow monorepo:

```
File → Open Folder → /home/mehrin/repo/_gsoc/flow
```

### Step 3 — Open Copilot Chat

- Press `Ctrl+Shift+I` (or `Cmd+Shift+I` on macOS), **or**
- Click the **Copilot Chat** icon in the sidebar / title bar

### Step 4 — Use a slash command

Type any of the available commands in the Copilot Chat input box. See the full list below.

---

## 🛠️ Available Commands

### `/understand` — Full Codebase Analysis

Runs a multi-agent pipeline to analyze every file, function, class, and dependency. Produces a structured graph JSON.

```
/understand
```

### `/understand-dashboard` — Interactive Knowledge Graph

Generates an interactive HTML dashboard you can open in your browser to visually explore the codebase as a knowledge graph with pan, zoom, and search.

```
/understand-dashboard
```

> [!TIP]
> This is the best starting point — it gives you a visual map of the entire project.

### `/understand-onboard` — Guided Onboarding Tour

Creates an auto-generated walkthrough of the architecture, ordered by dependency layers. Ideal for getting up to speed on a new codebase.

```
/understand-onboard
```

### `/understand-domain` — Business Domain Mapping

Switches to a domain-oriented view, mapping source code to real business processes, flows, and steps.

```
/understand-domain
```

### `/understand-diff` — Diff Impact Analysis

Analyzes your current uncommitted changes (or a specific diff) and shows which parts of the system are affected. Great for pre-commit review.

```
/understand-diff
```

### `/understand-explain` — Explain Code

Ask for a plain-English explanation of specific files, functions, or modules.

```
/understand-explain src/components/Reader.tsx
```

### `/understand-chat` — Ask Questions

Ask free-form questions about the codebase and get context-aware answers grounded in the knowledge graph.

```
/understand-chat How does the highlight system work?
```

### `/understand-knowledge` — Knowledge Base Graphs

Point it at documentation or a wiki to generate a force-directed knowledge graph with community clustering.

```
/understand-knowledge
```

---

## 💡 Tips & Best Practices

1. **Start with `/understand-dashboard`** to get the big picture, then drill down with `/understand-explain` for specific areas.

2. **Use `/understand-onboard`** when joining a new project or helping a teammate ramp up.

3. **Run `/understand-diff` before committing** to catch unexpected ripple effects across the codebase.

4. **The dashboard is an HTML file** — after generation, open it in your browser for the full interactive experience (pan, zoom, search, click nodes).

5. **Large codebases**: The analysis can take a few minutes on very large projects. The plugin batches files and processes them in parallel.

6. **`.understandignore`**: Create a `.understandignore` file in your project root (same syntax as `.gitignore`) to exclude directories like `node_modules/`, `dist/`, or `vendor/` from analysis.

---

## 🗂️ File Locations

| Item | Path |
|---|---|
| Plugin repo clone | `~/.understand-anything/repo` |
| Skills directory | `~/.copilot/skills/` |
| Universal plugin link | `~/.understand-anything-plugin` |
| Generated output | Typically in `.understand-anything/` within your project |

---

## 🔧 Troubleshooting

### Skills not showing up in Copilot Chat

- Make sure you **reloaded the VS Code window** after installation.
- Verify the symlinks exist: `ls -la ~/.copilot/skills/`
- Check that GitHub Copilot Chat extension is installed and signed in.

### Permission errors

- The installer needs write access to `~/.copilot/skills/`. If that directory doesn't exist, it will be created automatically.

### Alternative: Auto-discovery via repo

Instead of global symlinks, you can clone the Understand-Anything repo and open it directly in VS Code. Copilot will auto-discover the plugin via its `.copilot-plugin/plugin.json` file.

```bash
cd ~/.understand-anything/repo
code .
```
