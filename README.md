# Karat Interview Prep — Citibank via Deloitte

Technical interview preparation material for senior Java/Spring Boot roles. Covers Codility (coding test) + Karat (AI-driven interview).

## How to use (no tools required)

1. Start with `checklists/` — see what topics exist per level.
2. Read topic files in `topics/level-1/`, `level-2/`, `level-3/` — each follows a consistent template: why it matters → core concept → code → 4-beat interview answer → pitfalls → self-check question.
3. Work through levels in order. Level 2 is the primary focus for senior roles.

## Structure

```
├── prep/cheatsheet.md          # Original question list from Director
├── checklists/                 # Topic checklists per level
│   ├── level-1-standard.md     #   30 topics — fluency warm-up
│   ├── level-2-senior.md       #   44 topics — PRIMARY FOCUS
│   └── level-3-expert.md       #   29 topics — expert stretch
├── topics/                     # Study material (one file per topic)
│   ├── level-1/
│   ├── level-2/
│   └── level-3/
├── codility/                   # Coding test patterns and attempts
│   ├── patterns.md
│   └── attempts/
└── _status.yaml.template       # Progress tracking template (see below)
```

## Optional: progress tracking with Claude Code

If you use [Claude Code](https://claude.ai/code), this repo includes rules (`CLAUDE.md`) that enable automatic progress tracking:

1. Copy the template: `cp _status.yaml.template _status.yaml`
2. Open the repo in Claude Code — it reads `CLAUDE.md` and tracks your study progress in `_status.yaml`.
3. Commands: "show revision queue", "quiz me on level 2", "mark complete".
4. `_status.yaml` is gitignored — your progress stays local.

If you don't use Claude Code, ignore `CLAUDE.md` and `_status.yaml.template` entirely. The topic files work standalone.

## Levels

| Level | Target | Topics | Focus |
|:-----:|--------|:------:|-------|
| 1 | 2-3 yrs | 30 | Java basics, Spring fundamentals — warm-up |
| **2** | **3-5 yrs** | **44** | **Collections internals, concurrency, Spring Boot, microservices — PRIMARY** |
| 3 | 5+ yrs | 29 | JVM internals, GC, AOP, deadlocks, performance — stretch |