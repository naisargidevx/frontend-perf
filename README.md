# frontend-perf

A Claude Code skill that audits and fixes frontend performance issues — Core Web Vitals, image optimization, bundle size, React rendering, and Fynd theme performance.

## Installation

The skill must be placed at `.claude/skills/frontend-perf/` inside your project root.

### Option 1: Git Submodule (Recommended)

Keeps the skill version-controlled and updatable.

```bash
mkdir -p .claude/skills
git submodule add https://github.com/naisargidevx/frontend-perf.git .claude/skills/frontend-perf
git commit -m "Add frontend-perf Claude skill"
```

Teammates clone the skill along with the project:

```bash
git submodule update --init --recursive
```

Update to latest version later:

```bash
git submodule update --remote .claude/skills/frontend-perf
git commit -m "Update frontend-perf skill"
```

---

### Option 2: Clone Directly

No submodule tracking — just a one-time install.

```bash
mkdir -p .claude/skills
git clone https://github.com/naisargidevx/frontend-perf.git .claude/skills/frontend-perf
```

---

### Option 3: Global Install (All Projects)

Install once, available in every project on your machine.

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/naisargidevx/frontend-perf.git ~/.claude/skills/frontend-perf
```

---

## Verify Installation

After installing, your project structure should look like:

```
your-project/
└── .claude/
    └── skills/
        └── frontend-perf/
            ├── SKILL.md
            └── references/
```

Claude Code auto-detects `SKILL.md` — no extra config needed.

## Usage

Once installed, trigger the skill by mentioning performance in Claude Code:

- "My site is slow, can you audit it?"
- "Check my LCP score"
- "Optimize images in HeroBanner.jsx"
- "Audit my project"

## What It Covers

- Core Web Vitals: LCP, CLS, INP, FCP, TTFB, TTI
- Image optimization and lazy loading
- Bundle size and tree shaking
- React rendering performance
- Fynd theme: `transformImage`, SSR safety, section code splitting
