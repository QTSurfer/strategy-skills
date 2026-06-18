# Contributing to QTSurfer Strategy Skills

Thanks for helping improve the skills that teach AI agents how to write
[QTSurfer](https://qtsurfer.com) trading strategies. This repo is documentation
first: the "code" is the guidance and the worked examples inside each skill, so
accuracy matters as much as it would in source code.

## Repository layout

```
.claude-plugin/marketplace.json   # Claude Code plugin marketplace manifest (versions live here)
skills/<skill-name>/
  SKILL.md                        # skill manifest: frontmatter + instructions
  references/                     # supporting docs (indicators, patterns, examples)
CHANGELOG.md                      # Keep a Changelog 1.1.0
```

## How to contribute

1. Fork the repo and create a branch off `main` (`fix/...`, `docs/...`, `feat/...`).
2. Make your change following the conventions below.
3. Open a pull request. The PR template walks you through the checklist.

For anything larger than a typo — a new skill, a reworked example, an API
change — please open an issue first so we can align on the approach.

## Conventions

### Java examples must be correct

Every Java snippet is something a user will paste and submit for backtesting, so
it must compile against the QTSurfer engine API. The most common mistake is the
wrong import path. In particular:

- Strategies extend `com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy`.
- `@StrategyProperty` is `com.wualabs.qtsurfer.engine.strategy.StrategyProperty`.

When in doubt, submit the strategy through the
[MCP server](https://github.com/QTSurfer/mcp-java) (`submit_backtest`) — it
compiles server-side and will reject bad code.

### `SKILL.md` frontmatter

- `name` — kebab-case, no spaces (required by the Agent Skills schema).
- `description` — one paragraph; it is what the agent uses to decide when the
  skill applies, so keep it specific.
- `license` — `Apache-2.0`.
- `metadata.version` — mirror the version in `marketplace.json` (see below).

Keep `SKILL.md` lean and push depth into `references/`.

### Versioning

We follow [Semantic Versioning](https://semver.org/). When a skill's content
changes, bump its version in **all** of these, in lockstep:

- `.claude-plugin/marketplace.json` → `plugins[].version`
- `skills/<skill-name>/SKILL.md` → `metadata.version`
- `CHANGELOG.md` → a new entry

| Change | Bump |
|---|---|
| Fix a typo / broken example / clarify wording | patch (`x.y.Z`) |
| Add a new pattern, example, or reference section | minor (`x.Y.0`) |
| Restructure or rename a skill, breaking guidance | major (`X.0.0`) |

### Changelog

`CHANGELOG.md` follows [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/).
Add your entry under `## [Unreleased]`; it is promoted to a version on release.
Use the section emojis already in the file: `### Added ✨`, `### Changed 🔄`,
`### Fixed 🐛`, `### Removed 🗑️`.

### Commit messages

Conventional Commits, scoped to the skill where it helps:

```
fix(qtsurfer-java-strategy): correct @StrategyProperty import in examples
docs(qtsurfer-java-strategy): add cross-instrument pattern
chore: release 1.0.1
```

## Testing your changes locally

Install the skill from your fork before opening the PR:

```bash
# Agent Skills CLI (Claude Code, Codex, Cursor, Cline, …)
npx skills add <your-fork>/strategy-skills --skill qtsurfer-java-strategy

# Claude Code plugin marketplace
claude plugin marketplace add <your-fork>/strategy-skills
claude plugin install qtsurfer-java-strategy@qtsurfer-strategy-skills
```

Then ask your agent to write a strategy that exercises the part you changed, and
submit it through the MCP server to confirm it compiles and behaves as
documented.

## Adding a new skill

New skills are welcome — see the roadmap in the [README](./README.md). A skill
is a new directory under `skills/` with its own `SKILL.md` and `references/`,
plus an entry in `.claude-plugin/marketplace.json`. Match the structure and
conventions of `qtsurfer-java-strategy`.

## License

By contributing, you agree that your contributions are licensed under the
[Apache-2.0](./LICENSE) license.
