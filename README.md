# frontend-perf

A Claude Code skill that audits and fixes frontend performance issues — Core Web Vitals, image optimization, bundle size, React rendering, and Fynd theme performance.

## Installation

The skill must be placed at `.claude/skills/frontend-perf/` inside your project root.


### Option 1: Git Submodule (Recommended)

Keeps the skill version-controlled and updatable.

```bash
npx skills add  https://github.com/naisargidevx/frontend-perf.git .claude/skills/frontend-perf
```

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
