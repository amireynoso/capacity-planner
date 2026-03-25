# Capacity Planner

A lightweight, self-contained sprint capacity planning tool. No build step, no dependencies to install, no backend required. Just open `index.html` in a browser.

---

## Features

- **Gantt view** — project and epic timeline across sprints, grouped by month, quarter, and half-year. RAG status per project per sprint based on team capacity.
- **Capacity overview** — per-person and team-total sprint points, accounting for events and capacity overrides.
- **Assignments** — assign people to projects with time-ranged allocation periods (e.g. "Spencer is 80% on Project X from sprint 3").
- **Team** — set velocity per person, override capacity for specific sprints (PTO, sick, on-call), with a quick deduct calculator and a deductions log.
- **Events** — hackathons, retreats, holidays — any event that reduces the whole team's capacity for the sprints it overlaps.
- **Sprints** — edit sprint labels and dates directly, or auto-generate a sequence of Mon–Fri two-week sprints.
- **Data** — export your plan as JSON, import from a file or paste, and clear all data if needed.

---

## Getting started

### Run locally

```bash
# Clone the repo
git clone https://github.com/amireynoso/capacity-planner.git
cd capacity-planner

# Open in your browser — no server needed
open index.html
# or on Windows:
start index.html
# or on Linux:
xdg-open index.html
```

That's it. No `npm install`, no build step.

### Run from GitHub Pages

https://amireynoso.github.io/capacity-planner

---

## Data storage

Data is saved automatically to **browser localStorage** as you make changes. It persists across sessions on the same browser and device.

> ⚠️ localStorage is per-browser, per-device. Clearing browser data will wipe your plan. **Export a backup regularly.**

### Sharing data

Since localStorage is local, sharing the URL alone won't share your data. To share your plan:

1. Go to the **Data tab** → **Export** → copy the JSON.
2. Send the JSON to your collaborator.
3. They open the planner, go to **Data tab** → paste into the import box → **Import from text**.

---

## Backup

Export your plan from the **Data tab** at any time. The JSON file can be:

- Saved as a local backup
- Committed to the repo as a snapshot (e.g. `snapshots/2026-Q1.json`)
- Shared with teammates for import

---

## Concepts

### Velocity

Each team member has a fixed average velocity — the number of story points they typically deliver per sprint. This is set in the **Team tab**.

### Capacity

Available capacity per person per sprint, expressed as a percentage (100% = fully available). Reduced by:

- **Manual overrides** — for PTO, on-call, part-time weeks, etc.
- **Events** — hackathons, retreats, etc. reduce everyone's capacity proportionally for the sprints they overlap.

**Effective points** = velocity × effective capacity (after overrides and events).

### Assignments

Each assignment connects a person to a project at an allocation percentage. Allocations are time-ranged using **periods** — you can change someone's allocation from sprint 5 onward without affecting earlier sprints.

### RAG status

The Gantt shows a status indicator per project per sprint:

- 🟢 **On track** — assigned people have ≥70% average effective allocation
- 🟡 **At risk** — 35–70% average effective allocation
- 🔴 **Off track** — <35%, or no one assigned

> Note: RAG reflects **capacity availability**, not scope completion. It tells you whether people are available to work on the project, not whether it will be delivered on time.

You can override RAG manually per project in the Gantt edit panel.

---

## File structure

```
capacity-planner/
├── index.html   # The entire app — HTML, CSS (Tailwind CDN), and React (CDN)
└── README.md
```

All dependencies are loaded from CDN at runtime:

- [React 18](https://react.dev) via unpkg
- [Tailwind CSS](https://tailwindcss.com) via CDN
- [Babel Standalone](https://babeljs.io/docs/babel-standalone) for JSX transformation

> For production use or offline-first requirements, consider vendoring these dependencies locally.

---

## Customisation

Everything lives in `index.html`. Key things you might want to change:

| What                    | Where in the file                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Team name in header     | Search for `Capacity Planner` in the `App` component                                                                     |
| Default team members    | `INIT_TEAM` constant near the top                                                                                        |
| Default sprint start    | `INIT_SPRINTS` constant                                                                                                  |
| Sprint length           | `genSprints` function — change `addDays(s, 11)` (12 calendar days = Mon–Fri fortnight) and `addDays(e, 3)` (weekend gap) |
| RAG thresholds          | `rag` callback in `App` — adjust the 70/35 percentages                                                                   |
| Capacity deduct reasons | `REASONS` constant near the top                                                                                          |

---

## Tips

- **Start with sprints** — set up your sprint calendar in the Sprints tab before adding anything else.
- **Add your team** — set each person's velocity in the Team tab.
- **Add projects** — use the Gantt tab. Manual projects work the same as Jira-synced ones.
- **Assign people** — go to Assignments, add each person to their project(s) with an allocation % and a starting sprint.
- **Log time off** — use the quick deduct calculator in Team to log PTO, sick days, and on-call.
- **Export before making big changes** — it's your safety net.
