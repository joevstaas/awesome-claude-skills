# Portal Specification — Interactive HTML Report

Read this file after the Synthesizer role is approved. Build the portal described here as the final output of the scenario-analyst pipeline.

## Output

Write the portal to `output/report.html`, open nothing, and tell the user in one or two lines where the file is, that it works offline, and that **Export PDF** in the header prints all sections expanded (File > Print > Save as PDF).

This is **not** a document. It is a single-page application with a persistent sidebar, interactive visualizations, and drill-down panels. Think dashboard, not report.

The HTML file must be:

- Fully self-contained (no external dependencies, no CDN links, no images)
- All CSS inline in a `<style>` block in the `<head>`
- All JavaScript inline in a `<script>` block before `</body>`
- Readable offline with no internet connection required
- Driven by data: all report content embedded as JavaScript constants at the top of the `<script>` block (see [Data Embedding Pattern](#data-embedding-pattern)), not hardcoded HTML strings

---

## Layout — Single Page Application Shell

Three structural zones, always visible.

### Zone 1 — Top Header Bar

Fixed, full width, ~56px tall.

- Report title (short version, from the ANALYZE query)
- Subject organization name
- Time horizon badge (e.g. "2030")
- "Export PDF" button (calls `window.print()`)
- Color: dark navy (`#0f1f3d`), white text

### Zone 2 — Left Sidebar

Fixed, ~220px wide, full height, scrollable. Navigation links load content into Zone 3. Active link highlighted.

- **OVERVIEW**
  - Executive Summary
  - **No-Regret Moves** *(bold, highlighted — the anchor view)*
- **LANDSCAPE**
  - Drivers
  - 2x2 Scenario Matrix
  - Stakeholder Map *(collaborative mode only)*
- **SCENARIOS** *(one link per scenario, generated from data)*
- **ANALYSIS**
  - Game Theory
  - SWOT
  - Investments
- **REPORT**
  - Conclusions
  - Caveats
  - Sources & Terms

### Zone 3 — Main Content Panel

Fills remaining space, scrollable. All content loads here. **Default view on page load: No-Regret Moves.**

---

## Content Panels

### Executive Summary

- Large pull-quote: the single most important implication in 24px bold
- The full 3–4 sentence executive summary in readable prose
- Three KPI cards in a row: Time Horizon | Scenarios Modeled | Drivers Identified
- No-Regret Moves teaser: three cards, click to jump to the No-Regret Moves panel

### No-Regret Moves *(default landing panel)*

The most important panel. Design it to feel like a strategy command center, not a list.

**Top — three No-Regret Move cards side by side.** Each card contains:

- Move number and title in large text
- One-line strategic rationale
- Urgency badge: `0–6m` / `6–18m` / `18–36m` (color coded)
- Scenarios covered: small colored dots, one per scenario
- Click to expand: reveals the full rationale and specific actions in a smooth animated panel below the card

**Bottom — Conditional Bets table:** Scenario | Investment | Trigger Signal, compact, with the scenario column color-matched to the scenario colors.

### Drivers

Two sub-tabs: **"Trend Drivers (Class A)"** and **"Scenario Drivers (Class B)"**.

**Class A — horizontal bar chart** (SVG, generated from data):

- One bar per driver, sorted by Impact descending
- Bar color intensity reflects impact (light to dark blue)
- Direction indicator on the right: up arrow (green), down arrow (red), dash (grey)
- Hover: tooltip with Timeline and Strategic Note
- Below the chart: compact table with all Class A details

**Class B — uncertainty cards:**

- One card per driver, split horizontally: Outcome 1 (left) vs. Outcome 2 (right)
- Center: driver name, impact score, current probability lean
- The currently more probable outcome has a stronger background
- Below the cards: which axis each driver feeds into

### 2x2 Scenario Matrix

A large SVG (minimum 500×500px):

- Two axes with labels at each pole (from the approved axes)
- Four quadrants, each with a distinct muted background color, containing:
  - Scenario name in bold
  - Probability badge (High/Medium/Low)
  - 2–3 word evocative description
  - A subtle icon or symbol representing the world state
- Clicking a quadrant navigates to that scenario's panel
- The most likely quadrant has a glowing border or highlighted treatment

Below the SVG: a one-paragraph scenario comparison narrative.

### Stakeholder Map *(collaborative mode only)*

Built from `REPORT.entities` (the content of `output/entities.json`):

- Filter chips by entity type (organization, regulation, plan, measure, initiative, funding source, dataset, network) and by sector (public, private, voluntary, research)
- A simple SVG network view: entities as nodes colored by type, relations as lines; click a node to show its description, URL and relations in a side card
- Below: searchable table of all entities (Name | Type | Sector | Description | Link)

Keep the layout algorithm simple and inline (e.g. a small force or circular layout written in plain JavaScript).

### Scenario [N] — [Name]

One panel per scenario, same template. Header bar in the scenario's color with name, quadrant label, and probability badge.

**Left column (60%):**

- World Description in full (prose, readable font size)
- Defining Signals as a checklist with circle bullets

**Right column (40%):**

- **Radar chart** (SVG):
  - One axis per player group from the game theory
  - Competitive mode: threat level per group, title "Threat Landscape — [Scenario Name]"
  - Collaborative mode: show threat level and label each axis with the group's role (Contributor / User / Competing initiative / Gatekeeper / Funder), title "Stakeholder Landscape — [Scenario Name]"
  - Filled polygon in the scenario color with 40% opacity
  - Labels outside each axis with group name and level
- **Key drivers:** three pills with name and impact score; click to jump to that driver in the Drivers panel

**Bottom — expandable sections** (collapsed by default):

- `[+]` Game Theory detail for this scenario
- `[+]` SWOT Analysis for this scenario (renders as a 2x2 colored grid)
- `[+]` Recommended Actions for this scenario

Each section expands smoothly with a CSS transition.

### Game Theory (cross-scenario view)

Scenario selector tabs at the top (color coded). Switching tabs updates content without page reload.

Per scenario:

- Power Dynamics Summary paragraph
- Player group table: Player Group | Dominant Strategy | Key Moves | Threat Level (+ Role in collaborative mode)
  - Threat cells color-coded: Critical = red, High = orange, Medium = amber, Low = green
- Sub-scenarios as expandable accordion items

### SWOT

Scenario selector tabs at the top. Per scenario:

- 2x2 grid: Strengths (green tint), Weaknesses (red tint), Opportunities (blue tint), Threats (orange tint); bullet points, bold cell headers
- Strategic Priority Assessment below the grid as a compact table

### Investments

Mirrors the No-Regret Moves layout with full detail:

- **Section A:** capability gap tables per scenario (tabbed)
- **Section B:** ranked no-regret proposals with expanded prose for the top 3
- **Section C:** conditional bets table

### Conclusions

Five callout boxes, each with:

- A short bold headline (the conclusion in one line)
- Two to three sentences of supporting reasoning
- A tag showing which scenarios it applies to

Left border accent in the primary blue.

### Caveats

- Confidence rating as a three-segment bar (1 = Low, 2 = Medium, 3 = High)
- Assumptions as cards with a warning icon
- Watch signals as cards with a radar/signal icon

### Sources & Terms

- Sources with hyperlinks in two classes: **Official statistics & peer-reviewed research**, and **News, reports & other**
- Key terms with short explanations and Wikipedia links

### Credit

Small grey text at the bottom of the portal:

> Based on the Scenario Analyst skill shared by Jan Eivind Danielsen (linkedin.com/in/danielsenconsulting)

---

## Visual Design System

### Color Palette

| Role | Hex |
|---|---|
| Primary navy | `#0f1f3d` |
| Accent blue | `#2563eb` |
| Scenario 1 (purple) | `#7c3aed` |
| Scenario 2 (teal) | `#0891b2` |
| Scenario 3 (amber) | `#d97706` |
| Scenario 4 (emerald) | `#059669` |
| Success / positive | `#059669` |
| Warning / medium | `#d97706` |
| Danger / critical | `#dc2626` |
| Neutral | `#6b7280` |
| Background | `#f8fafc` |
| Card background | `#ffffff` |
| Border | `#e2e8f0` |

Assign one color per scenario and use it consistently throughout.

### Typography

- **Font:** `system-ui, -apple-system, sans-serif`
- **Headings:** 24px (panel title), 18px (section), 14px (label)
- **Body:** 14px, line-height 1.6
- **Data values:** `ui-monospace, monospace`

### Component Rules

- **Cards:** white background, 1px border (`#e2e8f0`), 8px radius, 12px padding, box-shadow `0 1px 3px rgba(0,0,0,0.08)`
- **Badges:** 4px radius, 4px × 10px padding, 12px font, weight 500
- **Expandable sections:** chevron rotates 90° on open; content animates with max-height transition (0.25s ease)
- **Hover states** on all interactive elements: 0.15s opacity or background transition
- **Active sidebar item:** 3px accent-blue left border, blue text, light blue background tint

### Print Styles (`@media print`)

- Hide sidebar and header bar
- Expand all collapsed sections
- Remove shadows and transitions
- Font size 11pt
- Page break before each major section
- Display all panels sequentially, including every scenario panel

---

## Data Embedding Pattern

At the top of the `<script>` block, embed all content as one constant. All rendering functions read from it; no content is hardcoded in HTML or rendering functions.

```javascript
const REPORT = {
  title: "...",
  subject: "...",
  horizon: "2030",
  date: "...",
  mode: "competitive" | "collaborative",
  executiveSummary: "...",
  keyImplication: "...",
  driversClassA: [
    { name, impact, timeline, direction, evidence, note },
  ],
  driversClassB: [
    { name, impact, timeline, outcome1, outcome2, currentLean, evidence, axis },
  ],
  axes: {
    x: { label, low, high },
    y: { label, low, high }
  },
  scenarios: [
    {
      id, name, quadrant, color, probability, keyDrivers,
      description, signals, implications,
      swot: { strengths, weaknesses, opportunities, threats },
      priorityAssessment: [ { priority, shift, why } ],
      gameTheory: {
        players: [ { group, strategy, moves, winsLoses, threat, role } ],
        powerSummary,
        subScenarios: [ { setup, winner, loser, subjectPosition, response } ]
      },
      capabilityGaps: [ { capability, currentState, gap, evidence } ],
      recommendedActions
    },
  ],
  noRegretMoves: [
    { rank, title, rationale, urgency, scenariosCovered, detail },
  ],
  conditionalBets: [
    { scenario, investment, trigger },
  ],
  conclusions: [
    { headline, reasoning, scenarios },
  ],
  caveats: { confidence, assumptions, watchSignals },
  entities: { entities: [ /* from output/entities.json */ ], relations: [ ] },
  sources: {
    official: [ { title, publisher, url } ],
    other:    [ { title, publisher, url } ]
  },
  terms: [ { term, explanation, wikipediaUrl } ]
};
```

Escape all text inserted into the DOM. Render every link with `target="_blank" rel="noopener"`.
