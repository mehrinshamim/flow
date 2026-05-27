# Flow — Repository Guide

> **Flow** is a free, open-source, browser-based ePub reader. This document explains the repository structure, architecture, technology choices, and how to contribute.

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
  - [Root Configuration](#root-configuration)
  - [Apps](#apps)
  - [Packages](#packages)
  - [Plotwise (GSoC)](#plotwise-gsoc)
- [Architecture Diagram](#architecture-diagram)
- [App Deep Dives](#app-deep-dives)
  - [Reader App (`apps/reader`)](#reader-app-appsreader)
  - [Website App (`apps/website`)](#website-app-appswebsite)
- [Package Deep Dives](#package-deep-dives)
  - [`@flow/epubjs`](#flowepubjs)
  - [`@flow/internal`](#flowinternal)
  - [`@flow/tailwind`](#flowtailwind)
- [Data Flow & State Management](#data-flow--state-management)
- [Development Setup](#development-setup)
  - [Prerequisites](#prerequisites)
  - [Quick Start](#quick-start)
  - [Environment Variables](#environment-variables)
  - [Useful Commands](#useful-commands)
- [Docker & Self-Hosting](#docker--self-hosting)
- [Code Style & Linting](#code-style--linting)
- [Internationalization (i18n)](#internationalization-i18n)
- [Testing](#testing)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

Flow lets users read `.epub` books directly in the browser — no installs, no accounts required. Key features include:

| Feature | Description |
|---|---|
| **Grid Layout** | Library view with cover thumbnails |
| **In-Book Search** | Full-text search across chapters |
| **Image Preview** | Zoom into embedded images |
| **Custom Typography** | Font family, size, weight, line-height, spread, zoom |
| **Highlights & Annotations** | Color-coded highlights (yellow / red / green / blue) with notes |
| **Theming** | Light/dark and custom color schemes via Material color utilities |
| **Share / Download** | Open books from a URL, export data as ZIP |
| **Cloud Sync** | Dropbox OAuth integration for cross-device reading progress |
| **PWA Support** | Installable progressive web app via `next-pwa` |
| **Internationalization** | English, Chinese, Japanese, German |

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Framework** | Next.js 12 (Pages Router) |
| **Language** | TypeScript 4.6 |
| **UI** | React 18, Tailwind CSS 3.2 |
| **Component Library** | `@literal-ui/core`, `@literal-ui/hooks`, `@literal-ui/next` |
| **State Management** | Recoil (global settings), Valtio (reader model/tabs) |
| **Client Database** | Dexie.js (IndexedDB wrapper) |
| **ePub Engine** | Vendored fork of Epub.js (`@flow/epubjs`) |
| **Monorepo** | pnpm workspaces + Turborepo |
| **Styling** | Tailwind CSS with shared preset (`@flow/tailwind`) |
| **Linting** | ESLint (TS + Next + Prettier integration) |
| **Formatting** | Prettier (single quotes, no semicolons, trailing commas) |
| **Error Monitoring** | Sentry (production builds) |
| **CI/CD** | Docker multi-stage build, Netlify deployment configs |
| **Git Hooks** | Husky + lint-staged |

---

## Repository Structure

```
flow/
├── apps/
│   ├── reader/          # Main ePub reader — Next.js app (port 7127)
│   └── website/         # Marketing / landing page — Next.js app (port 7117)
├── packages/
│   ├── epubjs/          # Vendored & patched Epub.js rendering engine
│   ├── internal/        # Shared TypeScript utilities across apps
│   └── tailwind/        # Shared Tailwind CSS preset (colors, typography, elevation)
├── plotwise/            # GSoC project docs — PlotWise narrative intelligence feature
├── Dockerfile           # Multi-stage Docker build for reader app
├── docker-compose.yml   # Docker Compose for self-hosting
├── turbo.json           # Turborepo pipeline configuration
├── pnpm-workspace.yaml  # pnpm workspace definition (apps/* + packages/*)
├── package.json         # Root scripts, devDependencies, lint-staged config
├── tsconfig.json        # Base TypeScript config (extended by apps)
├── .eslintrc.js         # Root ESLint config
├── prettier.config.js   # Prettier config
└── .husky/              # Git hooks (pre-commit formatting/linting)
```

### Root Configuration

| File | Purpose |
|---|---|
| `pnpm-workspace.yaml` | Declares `apps/*` and `packages/*` as workspace members |
| `turbo.json` | Defines `build`, `lint`, and `dev` pipelines; `build` depends on `^build` (dependency order) |
| `tsconfig.json` | Base config with path aliases; extended by `tsconfig.next.json`, `tsconfig.react.json`, `tsconfig.ts.json` |
| `.eslintrc.js` | Root ESLint — extends `eslint:recommended`, `@typescript-eslint/recommended`, `prettier`, `next`; enforces alphabetical import ordering with `@flow/**` as internal group |
| `prettier.config.js` | Single quotes, no semicolons, trailing commas, Tailwind CSS class sorting plugin |
| `.husky/` | Pre-commit hook runs `lint-staged` (Prettier + ESLint auto-fix) |

---

### Apps

#### `apps/reader` — The ePub Reader

The core product. A Next.js 12 app using the Pages Router.

```
apps/reader/
├── src/
│   ├── pages/               # Next.js pages
│   │   ├── index.tsx         # Main library + reader view (13KB — the primary UI)
│   │   ├── _app.tsx          # App wrapper (Recoil root, layout)
│   │   ├── _document.tsx     # Custom HTML document (fonts, meta)
│   │   ├── success.tsx       # OAuth success callback page
│   │   ├── styles.css        # Global CSS (Tailwind directives + custom styles)
│   │   └── api/              # API routes (e.g., Dropbox token refresh)
│   ├── components/
│   │   ├── Reader.tsx        # Core reader component — renders ePub via epub.js
│   │   ├── Layout.tsx        # App shell layout (sidebar + reader pane)
│   │   ├── TextSelectionMenu.tsx  # Context menu on text selection (highlight, annotate)
│   │   ├── Annotation.tsx    # Annotation rendering and management
│   │   ├── Theme.tsx         # Theme picker UI
│   │   ├── Tab.tsx           # Tab bar for multiple open books
│   │   ├── Form.tsx          # Form components (settings, inputs)
│   │   ├── Row.tsx           # List row component (library items)
│   │   ├── base/             # Low-level UI primitives (SplitView, DropZone, DnD)
│   │   ├── viewlets/         # Sidebar panels:
│   │   │   ├── TocView.tsx        # Table of Contents
│   │   │   ├── SearchView.tsx     # In-book search
│   │   │   ├── AnnotationView.tsx # Annotations list
│   │   │   ├── TypographyView.tsx # Typography settings
│   │   │   ├── ThemeView.tsx      # Theme settings
│   │   │   ├── ImageView.tsx      # Image preview
│   │   │   └── TimelineView.tsx   # Reading timeline
│   │   └── pages/
│   │       └── settings.tsx  # Settings page component
│   ├── models/
│   │   ├── reader.ts         # Central reader model (Valtio proxy) — book tabs, navigation, search, annotations
│   │   └── tree.ts           # Tree data structure helpers (for TOC/nav)
│   ├── hooks/                # Custom React hooks
│   │   ├── useTypography.ts      # Typography config hook
│   │   ├── useTextSelection.ts   # Text selection detection
│   │   ├── useMobile.ts          # Responsive breakpoint detection
│   │   ├── useTranslation.ts     # i18n hook
│   │   ├── useLibrary.ts         # Library data hook
│   │   ├── useEnv.ts             # Environment variables hook
│   │   ├── remote/               # Remote data hooks (Dropbox sync)
│   │   └── theme/                # Theme-related hooks (color scheme, background)
│   ├── db.ts              # Dexie.js database schema — books, files, covers tables
│   ├── state.ts            # Recoil atoms — navbar state, settings (typography, theme)
│   ├── sync.ts             # Dropbox sync logic — upload/download, data serialization, ZIP backup
│   ├── file.ts             # File handling — ePub parsing, book import, URL fetching
│   ├── annotation.ts       # Annotation types and color definitions
│   ├── styles.ts           # Default reader styles
│   ├── color.ts            # Color utilities
│   ├── platform.ts         # Platform detection (touch screen)
│   ├── mime.ts             # MIME type mappings (.epub, .zip)
│   └── utils.ts            # General utilities
├── locales/                # i18n translation files
│   ├── en-US.ts
│   ├── zh-CN.ts
│   ├── ja-JP.ts
│   └── de-DE.ts
├── public/                 # Static assets
├── .env.local.example      # Environment variable template
├── next.config.js          # Next.js config (PWA, Sentry, bundle analyzer, i18n, Docker output)
├── tailwind.config.js      # Tailwind config (extends @flow/tailwind preset)
└── package.json            # Dependencies and scripts
```

#### `apps/website` — Marketing Site

A simpler Next.js 12 app serving as the public landing page.

```
apps/website/
├── src/
│   ├── pages/
│   │   ├── index.tsx        # Landing page
│   │   ├── privacy.mdx      # Privacy policy (MDX)
│   │   ├── terms.mdx        # Terms of service (MDX)
│   │   ├── refund.mdx       # Refund policy (MDX)
│   │   └── styles.css       # Global styles
│   └── components/
│       ├── Layout.tsx       # Page layout
│       ├── MDX.tsx          # MDX rendering components
│       └── Seo.tsx          # SEO meta tags
├── locales/                 # i18n translations
├── public/                  # Screenshots, favicons, etc.
├── .env.local.example       # Environment variable template
├── next.config.js           # Next.js config (MDX, i18n, bundle analyzer)
└── package.json
```

---

### Packages

#### `packages/epubjs` — Vendored ePub Engine

A forked and maintained copy of [Epub.js](https://github.com/futurepress/epub.js/) (`v0.3.93`). This is the core rendering engine that parses and displays `.epub` files in the browser.

```
packages/epubjs/
├── src/
│   ├── book.js          # Book class — loads and manages ePub structure
│   ├── rendition.js     # Rendition — controls how content is displayed
│   ├── contents.js      # Content management — DOM manipulation within iframes
│   ├── epubcfi.js       # EPUB CFI (Canonical Fragment Identifier) parser
│   ├── navigation.js    # Table of Contents parsing
│   ├── annotations.js   # Annotation layer
│   ├── locations.js     # Location/pagination calculations
│   ├── section.js       # Individual chapter/section handling
│   ├── spine.js         # Book spine (ordered list of content documents)
│   ├── themes.js        # Theme injection into rendered content
│   ├── layout.js        # Layout calculations (paginated vs scrolled)
│   ├── managers/        # Rendering managers (continuous, default)
│   └── utils/           # URL, core, and other utilities
├── types/               # TypeScript type definitions
├── test/                # Karma/Mocha test suite
├── documentation/       # API documentation
└── package.json
```

#### `packages/internal` — Shared Utilities

Minimal shared TypeScript library consumed by both apps.

```typescript
// Currently exports:
export const str = 'Hello world'
export function range(n: number) {
  return [...new Array(n)].map((_, i) => i)
}
```

#### `packages/tailwind` — Shared Tailwind Preset

Provides a consistent design system across apps:

```
packages/tailwind/src/
├── index.js         # Tailwind preset entry point
├── colors.js        # Color palette definitions
├── typography.js    # Typography scale and font configurations
├── elevation.js     # Shadow/elevation utilities
└── state.js         # State-based (hover, focus, active) style utilities
```

---

### Plotwise (GSoC)

The `plotwise/` directory contains design documents for **PlotWise** — a GSoC project that adds a progress-aware narrative intelligence engine to Flow. It's designed to give readers spoiler-free AI-assisted context (character tracking, plot summaries) locked to their current reading progress.

| File | Content |
|---|---|
| `pitch.md` | Executive summary and product vision |
| `plotwise-prd.md` | Product requirements document |
| `usecase.md` | User stories and use cases |
| `stitch.md` | Technical stitching / integration strategy |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Monorepo (pnpm + Turborepo)          │
│                                                         │
│  ┌───────────────────────┐  ┌────────────────────────┐  │
│  │   apps/reader         │  │   apps/website         │  │
│  │   (Next.js :7127)     │  │   (Next.js :7117)      │  │
│  │                       │  │                        │  │
│  │  ┌─────────────────┐  │  │  Landing page          │  │
│  │  │  React UI       │  │  │  Privacy / Terms (MDX) │  │
│  │  │  (components,   │  │  │  SEO + i18n            │  │
│  │  │   viewlets,     │  │  └────────────────────────┘  │
│  │  │   hooks)        │  │                              │
│  │  └───────┬─────────┘  │                              │
│  │          │             │                              │
│  │  ┌───────▼─────────┐  │                              │
│  │  │ State Layer      │  │                              │
│  │  │ Recoil (settings)│  │                              │
│  │  │ Valtio (reader)  │  │                              │
│  │  └───────┬─────────┘  │                              │
│  │          │             │                              │
│  │  ┌───────▼─────────┐  │                              │
│  │  │ Data Layer       │  │                              │
│  │  │ Dexie (IndexedDB)│  │                              │
│  │  │ Dropbox Sync     │  │                              │
│  │  └───────┬─────────┘  │                              │
│  │          │             │                              │
│  │  ┌───────▼─────────┐  │                              │
│  │  │ @flow/epubjs     │◄─┼──── Vendored Epub.js fork   │
│  │  │ (ePub engine)    │  │                              │
│  │  └─────────────────┘  │                              │
│  └───────────────────────┘                              │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                  Shared Packages                  │   │
│  │  @flow/internal   │  @flow/tailwind               │   │
│  │  (TS utilities)   │  (design system preset)       │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## App Deep Dives

### Reader App (`apps/reader`)

#### Core Components

- **`Reader.tsx`** — The heart of the app. Embeds the `@flow/epubjs` rendition in an iframe, handles keyboard navigation (arrow keys, space), drag-and-drop file import, image previews, annotation overlays, and text selection menus.
- **`Layout.tsx`** — The app shell with a sidebar (viewlets) and the main reader pane arranged in a split view.
- **`TextSelectionMenu.tsx`** — Floating context menu that appears on text selection, allowing users to highlight, annotate, copy, or search selected text.
- **`Tab.tsx`** — Multi-tab interface supporting multiple simultaneously open books.

#### Viewlets (Sidebar Panels)

| Viewlet | Purpose |
|---|---|
| `TocView` | Table of Contents navigation |
| `SearchView` | Full-text search within the current book |
| `AnnotationView` | Browse and manage highlights/annotations |
| `TypographyView` | Adjust font size, family, weight, line height, spread, zoom |
| `ThemeView` | Switch between light/dark themes and custom color schemes |
| `ImageView` | Full-screen image preview using `react-photo-view` |
| `TimelineView` | Reading progress timeline |

#### Data Model

The reader uses **Dexie.js** (IndexedDB) with three tables:

| Table | Schema | Purpose |
|---|---|---|
| `books` | `id, name, size, metadata, createdAt, updatedAt, cfi, percentage, definitions, annotations, configuration` | Book metadata, reading position (CFI), progress, highlights |
| `files` | `id, file` | Raw ePub file blobs |
| `covers` | `id, cover` | Cover image data URLs |

The database has been through **5 schema versions** with migration logic to handle upgrades (adding metadata extraction, annotations, per-book typography config).

#### State Management

- **Recoil** — Used for global app state:
  - `navbarState` — sidebar visibility toggle
  - `settingsState` — persisted to `localStorage` (typography, theme, text selection menu toggle)
- **Valtio** — Used for the reader model (`models/reader.ts`):
  - Manages open book tabs, navigation, search, annotations
  - Reactive proxy-based state that auto-renders on changes

---

### Website App (`apps/website`)

A straightforward marketing site with:
- Landing page showcasing Flow's features
- Legal pages rendered from MDX (privacy policy, terms of service, refund policy)
- SEO optimization via `next-seo`
- Internationalization via `next-translate`

---

## Data Flow & State Management

```
User drops .epub file
        │
        ▼
  handleFiles() ──► fileToEpub() ──► ePub(arrayBuffer)
        │                                   │
        ▼                                   ▼
  addBook() ──────────────────────► epub.loaded.metadata
        │
        ├──► db.books.add(bookRecord)    // IndexedDB
        ├──► db.files.add({id, file})    // Store raw file
        └──► db.covers.add({id, cover}) // Extract & store cover
                    │
                    ▼
            reader.addTab(book)  // Valtio state
                    │
                    ▼
            Rendition.display()  // epub.js renders in iframe
```

**Sync flow (Dropbox):**
1. User authenticates via Dropbox OAuth → refresh token stored in cookie
2. `uploadData()` serializes `BookRecord[]` to JSON → uploads to Dropbox as `data.json`
3. `dropboxBooksFetcher()` downloads and deserializes the data
4. ZIP export/import via `pack()` / `unpack()` using JSZip

---

## Development Setup

### Prerequisites

| Tool | Version |
|---|---|
| [Node.js](https://nodejs.org) | >= 18.0.0 |
| [pnpm](https://pnpm.io/installation) | 10.6.4 (managed via `corepack`) |
| [Git](https://git-scm.com/downloads) | Any recent version |

### Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/pacexy/flow
cd flow

# 2. Install dependencies
pnpm install

# 3. Set up environment variables
cp apps/reader/.env.local.example apps/reader/.env.local
cp apps/website/.env.local.example apps/website/.env.local

# 4. Start all apps in parallel (with hot reload)
pnpm dev
```

The reader will be available at **http://localhost:7127** and the website at **http://localhost:7117**.

### Environment Variables

#### Reader (`apps/reader/.env.local`)

| Variable | Required | Description |
|---|---|---|
| `NEXT_PUBLIC_DROPBOX_CLIENT_ID` | No | Dropbox app client ID (for cloud sync) |
| `DROPBOX_CLIENT_SECRET` | No | Dropbox app secret (for token refresh) |
| `NEXT_PUBLIC_WEBSITE_URL` | Yes | URL of the website app (default: `http://localhost:7117`) |

#### Website (`apps/website/.env.local`)

| Variable | Required | Description |
|---|---|---|
| `NEXT_PUBLIC_APP_URL` | Yes | URL of the reader app (default: `http://localhost:7127`) |

### Useful Commands

| Command | Description |
|---|---|
| `pnpm dev` | Start all apps in parallel with hot reload |
| `pnpm build` | Production build for all workspaces |
| `pnpm lint` | Run ESLint across all workspaces |
| `pnpm --filter @flow/reader dev` | Run only the reader app |
| `pnpm --filter @flow/website dev` | Run only the website |
| `pnpm --filter @flow/epubjs test` | Run epubjs test suite (requires Chrome headless) |
| `pnpm release` | Publish packages to npm (public access) |

---

## Docker & Self-Hosting

### Using Docker Compose

```bash
# Set up environment variables first
cp apps/reader/.env.local.example apps/reader/.env.local
# Edit apps/reader/.env.local with your values

# Start the reader
docker compose up -d
```

The reader will be available at **http://localhost:3000**.

### Manual Docker Build

```bash
docker build -t flow .
docker run -p 3000:3000 --env-file apps/reader/.env.local flow
```

### Docker Build Details

The Dockerfile uses a **3-stage build**:

1. **Builder** — Uses `turbo prune` to isolate the `@flow/reader` workspace and its dependencies
2. **Installer** — Installs deps with `pnpm`, builds the Next.js app with `output: 'standalone'`
3. **Runner** — Minimal Alpine image running `node apps/reader/server.js` as a non-root user

---

## Code Style & Linting

### Prettier

| Rule | Value |
|---|---|
| Quotes | Single (`'`) |
| Semicolons | None |
| Trailing Commas | All |
| Indentation | 2 spaces |
| Tailwind Plugin | Auto-sorts Tailwind CSS classes |

### ESLint

- Extends: `eslint:recommended`, `@typescript-eslint/recommended`, `prettier`, `next`
- Import ordering enforced: builtin → external → `@flow/**` (internal) → parent → sibling → index
- Unused vars flagged as errors (with `_` prefix exception for args, rest siblings ignored)
- Relaxed rules: `no-explicit-any`, `ban-ts-comment`, `no-empty-function` are off

### Git Hooks

Pre-commit hook (via Husky + lint-staged):
- Runs Prettier on `*.{js,json,css,ts,tsx,md,mdx,yml,yaml}`
- Runs ESLint with `--fix` on `*.{js,ts,tsx}`

---

## Internationalization (i18n)

The reader supports 4 locales via Next.js built-in i18n routing:

| Locale | Language |
|---|---|
| `en-US` | English (default) |
| `zh-CN` | Simplified Chinese |
| `ja-JP` | Japanese |
| `de-DE` | German |

Translation files are TypeScript modules in `apps/reader/locales/`. The website uses `next-translate` with its own locale files.

---

## Testing

- **`@flow/epubjs`**: Karma + Mocha test suite
  ```bash
  pnpm --filter @flow/epubjs test
  ```
  Runs in Chrome Headless (no sandbox mode).

- **Reader & Website**: Currently rely on manual verification through `pnpm dev`. Smoke test steps should be documented in PR descriptions.

---

## Contributing

1. **Fork & clone** the repository
2. **Create a branch**: `git checkout -b feat/my-feature`
3. **Make changes** following the code style guidelines
4. **Test**: Run `pnpm build`, `pnpm lint`, and relevant tests
5. **Commit**: Use [Conventional Commits](https://www.conventionalcommits.org/) — `feat:`, `fix:`, `chore(reader):`
6. **Open a PR**: Link related issues, summarize UX changes, attach screenshots/recordings

---

## License

Flow is licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**. The vendored `@flow/epubjs` package is licensed under **BSD-2-Clause**.
