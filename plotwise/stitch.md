# STITCH_AI_UI_PROMPT.md

## PlotWise Front-End Generation Prompt

Copy and paste the text block below into Stitch AI, v0.dev, Lovable, or Claude Artifacts to generate the UI components.

***

**Project Overview:**
Build a mobile-first, highly immersive EPUB reader web application called "PlotWise". It must look like a premium, distraction-free reading app (like Apple Books or Kindle) but with an AI companion layered seamlessly on top. Use Next.js, React, Tailwind CSS, and Framer Motion for smooth animations.

**Color Palette & Typography (Adaptive to Flow Architecture):**
* **Themes:** Strictly utilize custom CSS utility classes and variables that adapt to Flow's native dark, sepia, and light themes (`var(--background)`, `var(--text)`).
* **Brand/AI Accent Color:** Vibrant Amber/Gold (Tailwind: `amber-500`) to represent the AI features. Use this for the AI buttons, active states, and AI chat bubbles to contrast with the core reader.
* **Typography:** A elegant serif font for the book text (e.g., Merriweather or Playfair Display) and a clean sans-serif (e.g., Inter) for the UI controls and AI chat windows.

**Core Layout & Components to Build:**
Please build a single interactive prototype that cleanly demonstrates these three key UI states:

1. The Main Reading View (Default State):
* A full-screen text area filled with placeholder book text (formatted cleanly with standard narrative paragraphs and indents).
* A minimal, auto-hiding Top Navbar containing: A hamburger menu icon (left), book chapter title (center), and an 'Aa' font-settings icon (right).
* A floating action button (FAB) in the bottom right corner with a sparkle/magic icon, styled with the Amber accent color.

2. The Context Menu (Highlight State):
* Show a block of the narrative text that is currently highlighted by the user.
* Directly above the highlight, render a custom iOS-style popover tooltip.
* The tooltip should contain three buttons: "Copy", "Highlight", and a distinct Amber-colored "Ask PlotWise" button featuring a tiny sparkle icon (`✨`).

3. The AI Bottom Sheet (Chat State):
* When the FAB or "Ask PlotWise" button is clicked, a bottom sheet slides up smoothly, taking up the bottom 60% of the screen.
* The sheet must feature a drag handle at the top and a slight backdrop blur/glassmorphism effect over the book text behind it.
* Inside the sheet: A sleek chat interface. Show one user message bubble ("Who is Gehrman Sparrow?") and one AI response bubble ("Gehrman Sparrow is a crazy adventurer persona created by Klein Moretti...").
* At the bottom of the sheet, a clean text input bar with an amber submit arrow.

4. The Side Drawer (World State Tab):
* When the top-left hamburger menu is clicked, a left-side drawer slides in.
* Implement a segmented control/tab option at the top of this drawer: `[ Chapters | World State ]`.
* Show the "World State" tab active, displaying a structured list of character profile cards (e.g., "Klein Moretti", "Audrey Hall") with clean placeholder avatars, titles, and brief text summaries.

**Interactivity Rules:**
Make the UI prototype interactive. Clicking the FAB or the "Ask PlotWise" popover button must open the bottom sheet. Grabbing the drag handle or tapping the background text area must close it. Ensure perfect touch-responsiveness for a mobile viewport.


Gemini is AI and can make mistakes.