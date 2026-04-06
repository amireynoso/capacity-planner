# Capacity Planner — Developer Handover

This document provides full context for continuing development on the Capacity Planner. It is intended for use with Claude Code, Cursor, or any AI-assisted development environment.

---

## Project overview

A single-file, self-contained sprint capacity planning tool built with React (via CDN), Tailwind CSS, and Babel Standalone. No build step, no backend, no package manager. The entire app lives in `index.html`.

It was built iteratively for the Insurance Setup team at Jane to answer the question: _given who's on the team, what's coming up, and what's already on their plates — can we actually deliver this?_

There is a second version (a Claude artifact) that connects to Claude's storage API and Atlassian MCP for live Jira sync. The standalone `index.html` described here is the local/GitHub Pages version with `localStorage` and no Jira integration.

---

## Tech stack

| Concern      | Choice           | Notes                                                                 |
| ------------ | ---------------- | --------------------------------------------------------------------- |
| UI framework | React 18 (UMD)   | Loaded from unpkg CDN                                                 |
| Styling      | Tailwind CSS     | Loaded from CDN — no compiler, so only pre-built utility classes work |
| JSX          | Babel Standalone | Transforms JSX in the browser at runtime                              |
| Persistence  | `localStorage`   | Key: `cap-planner-local`                                              |
| Build        | None             | Open `index.html` directly                                            |

> **Important Tailwind constraint:** Because there is no Tailwind compiler, arbitrary values (`w-[300px]`) and named group variants (`group-hover/name`) do **not** work. Use inline `style={{}}` for dynamic widths and React state for hover effects instead.

---

## File structure

```
capacity-planner/
├── index.html      # The entire application
└── README.md       # User-facing documentation
```

All components, helpers, constants, and state live inside the `<script type="text/babel">` block in `index.html`.

---

## Architecture

### Component tree

```
App
├── GanttView
│   └── EpicRow          (separate component — uses React hover state, not Tailwind group-hover)
├── CapacityView
├── AssignView
│   └── PeriodChip       (separate component — uses React hover state)
├── TeamView
│   └── LogRow
├── EventsView
├── SprintsView
└── DataView
```

### State — all lives in `App`

| State               | Type               | Description                                                                |
| ------------------- | ------------------ | -------------------------------------------------------------------------- | ------------------------------------------- |
| `team`              | `Person[]`         | People on the team, each with id, name, velocity                           |
| `projects`          | `Project[]`        | Projects/initiatives, each with id, name, start, end, ragOverride, epics[] |
| `events`            | `Event[]`          | Team-wide capacity events (hackathon, retreat, etc.)                       |
| `assignments`       | `Assignment[]`     | Person-to-project allocations with periods (see below)                     |
| `capOv`             | `{ "pid            | sid": number }`                                                            | Per-person per-sprint capacity override (%) |
| `capReasons`        | `{ "pid            | sid": string }`                                                            | Reason label for each override              |
| `sprints`           | `Sprint[]`         | Sprint calendar                                                            |
| `genStart` / `genN` | string / number    | Controls for auto-generate sprints                                         |
| `expanded`          | `{ projId: bool }` | Which Gantt project rows are expanded                                      |

### Key data shapes

```js
// Person
{ id: string, name: string, velocity: number }

// Project
{ id: string, name: string, start: string|null, end: string|null, ragOverride: "green"|"amber"|"red"|null, epics: Epic[] }

// Epic
{ id: string, name: string, start: string|null, end: string|null, points: number|null }

// Event
{ id: string, name: string, start: string, end: string, impact: number } // impact 0–100 = % capacity reduction

// Assignment (a "period")
{ id: string, personId: string, projectId: string, pct: number, fromSprintId: string }

// Sprint
{ id: string, label: string, start: string, end: string } // dates are YYYY-MM-DD
```

### Assignment periods model

Assignments are not flat — each `Assignment` record is a **period** with a `fromSprintId`. A person can have multiple periods on the same project:

```
Spencer → Prior Auth:
  - from S1 at 50%
  - from S5 at 100%
```

The `activeAlloc(personId, projectId, sp)` function finds the most recent period whose `fromSprintId` index is ≤ the target sprint's index. This allows allocations to change over time without losing history.

### Capacity calculation chain

```
effectiveCapacity(person, sprint) =
  capOv[person|sprint] ?? 100
  × event reductions for overlapping events

availablePoints(person, sprint) =
  person.velocity × effectiveCapacity / 100

projectPoints(project, sprint) =
  Σ availablePoints(person, sprint) × activeAlloc(person, project, sprint) / 100
  for each person assigned to project
```

### RAG logic

RAG is **capacity-based, not scope-based** — it reflects whether assigned people are available, not whether the project is on track to deliver. This is a known limitation.

```
avgEffectiveAlloc = mean(effCap(person, sprint) × alloc% / 100) for assigned people
green  = avgEffectiveAlloc >= 70
amber  = avgEffectiveAlloc >= 35
red    = avgEffectiveAlloc < 35 OR no one assigned
```

RAG can be manually overridden per project via the Gantt edit panel (`ragOverride` field).

---

## Persistence

Data auto-saves to `localStorage` under the key `cap-planner-local` on every state change via a `useEffect`. Load happens once on mount.

```js
// Save
localStorage.setItem("cap-planner-local", JSON.stringify({ team, projects, ... }));

// Load
const d = JSON.parse(localStorage.getItem("cap-planner-local"));
```

The Data tab exposes export (JSON display + copy), import (file upload or paste), and a "Clear all data" reset.

---

## Known quirks and gotchas

### Tailwind named group variants don't work

`group-hover/ep` and similar named group variants require the Tailwind JIT compiler and **silently fail** with the CDN version. This caused several bugs during development. The fix is to use React state:

```jsx
// ❌ Don't do this — silently broken
<div className="group/ep">
  <button className="opacity-0 group-hover/ep:opacity-100">...</button>
</div>;

// ✅ Do this instead
const [hovered, setHovered] = useState(false);
<div
  onMouseEnter={() => setHovered(true)}
  onMouseLeave={() => setHovered(false)}
>
  <button style={{ opacity: hovered ? 1 : 0 }}>...</button>
</div>;
```

This pattern is used in `EpicRow` and `PeriodChip`. Apply it to any new hover-reveal UI.

### Fixed-width left column in Gantt

The Gantt uses a fixed `LW = 300` constant for the left label column and `CW = 90` for each sprint column. These are passed as props to `EpicRow`. The left column must use `width: LW, minWidth: LW, flexShrink: 0` (not just `minWidth`) to prevent it from growing with long project names and misaligning the sprint cells.

### Assignment `newProject` default bug (fixed)

When opening the "Add project" form for a person, `newProject` state must be set to the first **unassigned** project for that specific person — not a global default. The `firstAvailable(personId)` helper computes this from `assignments` directly (not from `byPerson` memo) to avoid ordering/closure issues. If you refactor this area, preserve this pattern.

### Sprint IDs

Auto-generated sprints use sequential IDs (`sp1`, `sp2`, ...). Manually added sprints use `uid()` (timestamp-based). The `activeAlloc` function uses `sprints.findIndex()` to compare sprint positions, so IDs don't need to be sequential — only the `sprints` array order matters.

### Date handling

All dates are `YYYY-MM-DD` strings. All `new Date()` calls append `T12:00:00Z` to avoid timezone edge cases where a date string like `2026-03-03` gets parsed as UTC midnight and rolls back to the previous day in negative-UTC timezones.

```js
// Always do this
new Date("2026-03-03T12:00:00Z");
// Not this
new Date("2026-03-03");
```

---

## Planned / potential improvements

These were discussed during development but not yet built:

### High priority

- **Scope-aware RAG** — each project gets a "points remaining" field. RAG compares remaining scope against projected capacity burn rate to predict whether the end date is achievable. Currently RAG only reflects capacity availability, not delivery risk.
- **Jira import via file** — since the standalone version has no MCP, a workaround would be a CSV/JSON import from a Jira export. The data shape is documented in the Claude version's `syncJira` function.

### Medium priority

- **Selective event impact** — currently events reduce everyone's capacity equally. A useful extension would be tagging which team members an event affects (e.g. only on-call rotation affects 2 people, not the whole team).
- **Incident response tracking** — log incidents as capacity deductions with a shared label (e.g. "Incident: #1234") so multiple people's reductions are linked. Discussed but punted in favour of using the existing quick deduct + reason log for now.
- **Allocation check in Gantt** — the allocation check (over/under/unassigned) currently lives only in the Assignments tab. It would be useful to surface it in the Gantt header row per sprint.

### Lower priority

- **GitHub Pages + sharing** — add Netlify or Vercel deploy for a site password (Option A), or move storage to Supabase/Firebase for proper per-user data (Option C). See README for the full options breakdown.
- **Sprint velocity history** — track actual points delivered per sprint and compare against planned capacity. Would require a "completed points" input per sprint.
- **Dark mode** — the colour palette is light-only. All colours are Tailwind utility classes so a dark mode pass would be straightforward with a `dark:` variant (requires compiler or manual override).

---

## Development notes

### Adding a new tab

1. Add the tab name to `TABS` in `App`.
2. Add a `{tab==="newtab" && <NewTabView ... />}` line in the render.
3. Create `function NewTabView({ ... }) { ... }` at the bottom of the script block.
4. Pass any needed state and handlers as props from `App`.

### Adding a new field to a data shape

1. Update the relevant `add*` function in `App` to include the new field with a default.
2. Update the relevant `upd*` function if needed (most use spread so new fields pass through automatically).
3. Update the UI component to display/edit the field.
4. The `onImport` function uses conditional assignment (`if (d.field) set...`) so old exports without the field will just use the default — no migration needed for additive changes.

### Debugging persistence

```js
// In browser console — inspect stored state
JSON.parse(localStorage.getItem("cap-planner-local"));

// Clear stored state
localStorage.removeItem("cap-planner-local");
```

### Running without internet (offline)

The app currently loads React, Tailwind, and Babel from CDN. To run fully offline, download these files and update the `<script src>` and `<link href>` tags to point to local copies:

- `react.production.min.js` from unpkg
- `react-dom.production.min.js` from unpkg
- `babel.min.js` from unpkg
- Tailwind — either vendor the CSS output or use the CLI to generate a static stylesheet

---

## Claude version (for reference)

A parallel version exists as a Claude artifact with these additions:

- **Jira sync** via Atlassian MCP — pulls projects, epics, and sprints live from the Insurance Setup Jira board
- **Shared cloud storage** via `window.storage` (Claude's built-in storage API, shared: true) — anyone with the artifact link sees the same live data
- **PIN-based edit lock** — view-only mode for shared viewers, PIN-protected editing for the owner
- **Read-only mode** — all inputs and action buttons are disabled when locked

The data shape is identical between versions, so a JSON export from the Claude version can be imported into the standalone version and vice versa.

The Jira sync prompt and MCP config live in the `syncJira` async function in `App`. The Atlassian MCP URL is team-specific and should be updated if reused.

---

## Contact / context

Built for the Insurance Setup team at Jane. Primary user is the engineering manager. The tool is used for:

- Sprint-by-sprint capacity planning
- Quarterly planning conversations with stakeholders
- Communicating delivery timelines with real capacity data
- Tracking PTO, on-call, and events and their impact on available sprint points
