# MyGo Skills

Agent skills for building and maintaining apps with [github.com/egoist/mygo](https://github.com/egoist/mygo) — a Go framework for native desktop UI and webview frontends.

Published on [skills.sh](https://www.skills.sh); install with the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add transmit-bug/mygo-skills
```

Pick a single skill:

```bash
npx skills add transmit-bug/mygo-skills --skill mygo-components
```

[![skills.sh](https://skills.sh/b/transmit-bug/mygo-skills)](https://skills.sh/b/transmit-bug/mygo-skills)

## Skills

| Skill | Covers |
|---|---|
| [`mygo-usage`](skills/mygo-usage/SKILL.md) | How a MyGo app fits together — windows, view functions, state, threads — and how to build and verify a change, including headless `ui.NewTester` tests |
| [`mygo-components`](skills/mygo-components/SKILL.md) | Widget selection: reconciling every UI need against the official MyGo inventory before hand-rolling layout, plus layout, feedback and theming conventions |
| [`mygo-desktop`](skills/mygo-desktop/SKILL.md) | Services outside the window: dialogs, notifications, clipboard, shell, displays, theme, power, network, secrets, file watching, global shortcuts, menus, tray, lifecycle beyond `Run` |
| [`mygo-web`](skills/mygo-web/SKILL.md) | Web frontends: page loading and trust model, bound services, channels and typed events, the generated TypeScript client, and plugins |
| [`mygo-motion`](skills/mygo-motion/SKILL.md) | Motion: reconcile against built-in transitions, the timing table, values/poses/keyframes/timelines/gestures, less-motion and clock-driven animation testing |
| [`mygo-drawing`](skills/mygo-drawing/SKILL.md) | Custom painting: the draw-only rule and Painter, the four tiers, canvases, pan/zoom, shaders, shaped text, frame cost |

The skills reference each other by relative path (`../mygo-components/SKILL.md`); install them as a set — the sibling layout under `skills/` keeps those links working wherever the CLI copies them.

## Structure

```
skills/
├── mygo-usage/
├── mygo-components/
├── mygo-desktop/
├── mygo-web/
├── mygo-motion/
└── mygo-drawing/
```

Each skill is a directory with a `SKILL.md` (frontmatter `name` + `description`) — the standard [agent skills](https://agentskills.io) format. The `skills/` layout is auto-discovered by the CLI; no configuration needed.

## As a pack

On [skills.sh](https://www.skills.sh), add this repository (or the individual skills) to a [pack](https://www.skills.sh/docs/packs) to share all six with one install command:

```bash
npx skills add https://skills.sh/p/<pack-id>
```
