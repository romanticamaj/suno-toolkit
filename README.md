# suno-toolkit

Tooling for programmatic operations on Suno — extractors, exporters, automation.
Spec-driven development with [GitHub spec-kit](https://github.com/github/spec-kit).

## Status

Bootstrap phase. First tool: rewrite of `Suno Tracks Exporter` Chrome extension in TypeScript + Vite + MV3.

## Repo Layout

```
suno-toolkit/
├── .specify/         spec-kit workspace (constitution, specs, plans, tasks)
├── docs/             reverse-engineered architecture, Suno API reference
└── tools/            each subfolder is one tool (TBD by /speckit.plan)
```

## Workflow

```
/speckit.constitution   →   立 repo 原則
/speckit.specify        →   寫 what + why
/speckit.plan           →   寫 how + tech stack
/speckit.tasks          →   拆 TDD tasks
/speckit.implement      →   執行
```

See `docs/ARCHITECTURE.md` for the reverse-engineering of the original 1.0.5 extension that seeds the rewrite.
