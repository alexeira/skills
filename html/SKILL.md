---
name: html
description: >
  Generate self-contained HTML artifacts instead of Markdown for any non-code output: specs,
  plans, code reviews, design prototypes, reports, custom editing interfaces, explainers, decks.
  Trigger when the user asks to create a document, report, plan, spec, brainstorm output,
  prototype, visual explainer, or any rich artifact that benefits from layout, color,
  interactivity, or shareability. Do NOT trigger for source-code files, configs, README files,
  or when the user explicitly requests Markdown.
license: Apache-2.0
metadata:
  author: alexeira
  version: 0.1.0
  repository: https://github.com/alexeira/skills
  category: output
  tags: [html, artifacts, documentation, visualization, prototyping, reports]
---

# HTML Artifact Skill

## Philosophy

Markdown is a restricting format for AI-generated documents. HTML is not a web-development task — it is a richer communication medium. HTML lets you express:

- **Tabular data** with sortable, annotated tables
- **Visual hierarchy** with color, typography, and spacing
- **Diagrams and illustrations** with inline SVG
- **Interactions** with sliders, tabs, toggles, and copy buttons
- **Spatial layouts** for side-by-side comparisons, timelines, flow diagrams
- **Code** with syntax highlighting, diffs, and inline annotations

A 300-line HTML artifact is readable. A 300-line Markdown file is not.

---

## Non-Negotiable Rules

1. **Always single-file, self-contained.** Inline all CSS from `html/assets/base.css` into a `<style>` tag. No external stylesheets, no CDN imports, no `<link>` tags pointing to files outside the artifact.

2. **Never invent CSS.** Inline `html/assets/base.css` verbatim at the top of every `<style>` block. Add only component-specific rules after it. This saves tokens and ensures visual consistency.

3. **Use SVG for all diagrams.** Flowcharts, timelines, architecture diagrams, data-flow graphs — all SVG, inline. Never ASCII art.

4. **Every custom editing interface must export.** Any UI with draggable cards, form editors, toggles, or configuration panels must include a "Copy as JSON", "Copy as Markdown", or "Copy as Prompt" button so the output can flow back into Claude Code.

5. **Open in browser after generation.** Run `open <file>.html` (macOS) or tell the user to open the file locally. Never just say "here is the HTML".

6. **Name files descriptively.** Use kebab-case: `feature-plan.html`, `pr-review-auth.html`, `onboarding-designs.html`. Never `output.html` or `result.html`.

---

## When to Trigger

**Trigger for:**
- Planning, specs, implementation plans, architecture documents
- Code review artifacts, PR writeups, annotated diffs
- Design explorations, mockup comparisons, prototype interactions
- Reports, incident summaries, research explainers, status updates
- Custom editors for tickets, configs, prompts, datasets, annotations
- Any output longer than ~80 lines that a human needs to read

**Do NOT trigger for:**
- Source code files (`.ts`, `.py`, `.go`, etc.)
- Configuration files (`.json`, `.yaml`, `.env`)
- README or CHANGELOG files
- Simple one-paragraph answers
- When the user explicitly says "give me markdown" or "in markdown"

---

## CSS Template

Read `html/assets/base.css` from this skill and inline it verbatim into the `<style>` block of every artifact. The base CSS includes design tokens, reset, layout, typography, tables, code blocks, cards, badges, callouts, tabs, and a copy button — enough for every use case below.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title><!-- descriptive title --></title>
<style>
/* === base.css — paste full content here === */
/* [contents of html/assets/base.css] */

/* === component-specific styles below === */
</style>
</head>
<body>
<div class="wrap">
  <!-- content -->
</div>
</body>
</html>
```

---

## Use Cases

### 1. Specs, Planning & Exploration

Generate a visual web of artifacts rather than a single flat document. Start with explorations (multiple directions side by side), then expand into a full plan with mockups, data flow diagrams, and annotated code snippets.

**Patterns:**
- Side-by-side comparison grid for multiple approaches (use `.grid-2` or `.grid-3`)
- Timeline with milestones for implementation plans
- Inline SVG for architecture/data-flow diagrams
- Embedded code snippets with highlighted risk areas

**Example prompts:**
- "I'm not sure which direction to take for the onboarding screen. Generate 4 distinctly different approaches — vary layout, tone, and density — in a single HTML file in a grid. Label each with its tradeoff."
- "Create a thorough implementation plan as an HTML file: milestones timeline, data-flow diagram, mockups for the key screens, and the 3 most critical code snippets to review."
- "Read the codebase and brainstorm 3 architecturally distinct ways to solve the caching problem. Lay them out side by side with pros, cons, and a recommendation."

---

### 2. Code Review & Understanding

Render diffs, call graphs, module relationships, and annotations spatially. A reviewer should be able to scan the change in 2 minutes.

**Patterns:**
- Annotated diff: render `+`/`-` lines with `.diff-add`/`.diff-del`, margin notes using `.annotation`
- Severity badges: `.badge.red` (blocking), `.badge.yellow` (suggestion), `.badge.green` (praise)
- Module diagram: SVG boxes and arrows showing dependencies changed by the PR
- Tabbed layout: "Summary" | "File by File" | "Risk Areas"

**Example prompts:**
- "Help me review this PR. Create an HTML artifact with an annotated diff, color-coded by severity, focused on the streaming/backpressure logic I don't understand."
- "I don't understand how the rate limiter works. Read the code and produce a single HTML explainer: a token-bucket flow diagram, 3–4 annotated key snippets, and a gotchas section."
- "Write a PR description as an HTML file my reviewers will actually read: motivation, before/after comparison, file-by-file tour with the why, and where to focus."

---

### 3. Design & Prototypes

HTML is the best sketch pad for UI design — even when the target is React, Swift, or native. Prototype interactions, animations, and layout options. Add sliders and knobs to tune parameters interactively.

**Patterns:**
- Multi-column grid for layout options
- CSS transitions/animations for interaction prototypes
- Range `<input type="range">` sliders to tune animation parameters, spacing, colors
- "Copy parameters" button that exports tuned values as a prompt or JSON

**Example prompts:**
- "Prototype a checkout button: when clicked it plays an animation then turns purple. Create an HTML file with sliders for animation duration, easing, and color. Add a 'Copy parameters' button."
- "Generate 6 distinctly different designs for the empty state screen. Vary tone, illustration style, and CTA. Lay them in a 2×3 grid with labels."
- "Create a design token explorer: show every color, spacing, and font from our Tailwind config in a visual palette I can reference."

---

### 4. Reports, Research & Learning

Synthesize information from multiple sources (codebase, git history, MCPs, files) into a readable artifact. Use SVG for diagrams and tables for data. Design for someone reading it once.

**Patterns:**
- Long-form document layout with a sticky table of contents
- SVG flowcharts and architecture diagrams inline
- Tables with sortable columns for data-heavy sections
- Callout blocks (`.callout.info`, `.callout.warn`) for key findings
- Executive summary card at the top

**Example prompts:**
- "Read the git history for the auth module and produce an HTML report: what changed, why, and the 3 biggest architectural decisions. Target: engineering leadership."
- "Explain how our queue system works to someone new to the codebase. HTML page: a data-flow diagram, the 4 key code paths annotated, and a common-mistakes section."
- "Pull together a weekly status report as a polished HTML page: shipped features, open blockers, metrics from the last sprint, and what's next."

---

### 5. Custom Editing Interfaces

Build throwaway, purpose-built editors for structured data that is painful to describe in a text box. Always end with a copy/export button.

**Patterns:**
- Drag-and-drop card columns (Kanban) with a "Copy as Markdown" export
- Form-based config editors with dependency warnings and "Copy diff" export
- Side-by-side prompt editor with live variable substitution and token counter
- Approve/reject row annotator with "Export selection as JSON"

**The export rule:** every editing interface must have a clearly visible copy button that turns the UI state into text (JSON, Markdown, or a prompt) the user can paste back into Claude Code.

**Example prompts:**
- "I need to reprioritize 30 Linear tickets. Make an HTML file with each as a draggable card across Now / Next / Later / Cut columns. Pre-sort by priority. Add a 'Copy as Markdown' button."
- "Build a form editor for our feature-flag config. Group flags by area, show dependencies, warn on conflicts. Add a 'Copy diff' button."
- "Make a side-by-side prompt editor: editable prompt on the left with variable slots highlighted, 3 sample inputs on the right that render the filled template live. Add a token counter and copy button."

---

## Design System Pattern

To match a project's existing visual style, ask Claude to generate a design-system reference file first:

```
Read the codebase (tailwind.config, CSS variables, component files) and produce a single HTML file
that documents our design system: color palette swatches, typography scale, spacing scale, and the
10 most-used component patterns. Save as design-system.html.
```

Then reference that file in subsequent artifact prompts:
```
Using the style from design-system.html, create the implementation plan as an HTML file.
```

---

## Quick Reference

| Use case | Key elements |
|---|---|
| Specs & planning | Timeline SVG, `.grid`, code snippets, mockup frames |
| Code review | `.diff-add/.diff-del`, `.annotation`, `.badge`, module SVG |
| Design exploration | `.grid-3`, sliders, copy-params button |
| Reports | TOC nav, `.callout`, tables, flowchart SVG |
| Custom editor | Drag-and-drop, form inputs, `.copy-btn` export |
| Explainer | Annotated code, flow diagram SVG, `.callout.info` |

---

## Checklist Before Saving

- [ ] `base.css` inlined in full into `<style>` tag
- [ ] No external CSS/JS dependencies
- [ ] All diagrams are SVG (not ASCII, not `<img>`)
- [ ] Custom editing UIs have a visible export/copy button
- [ ] File named descriptively in kebab-case
- [ ] Run `open <file>.html` after writing
