# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UI/UX Pro Max** (package: `uipro-cli`, version 2.5.0) is an AI-powered design intelligence toolkit. It distributes searchable databases of UI styles, color palettes, font pairings, chart types, icons, and UX guidelines as a skill/workflow for 18 AI coding assistants (Claude Code, Cursor, Windsurf, Copilot, Gemini, Codex, Roocode, Kiro, Qoder, Trae, OpenCode, Continue, CodeBuddy, Droid/Factory, Kilocode, Warp, Augment, Antigravity).

The runtime is a Python BM25 + regex hybrid search engine over CSV datasets. The npm CLI (`uipro-cli`, built with Bun) installs the skill into a target project by rendering platform-specific templates and copying the data + scripts.

## Search Command

```bash
python3 src/ui-ux-pro-max/scripts/search.py "<query>" [--domain <d>] [--stack <s>] [-n <max_results>] [--json]
```

**Domains** (auto-detected when `--domain` is omitted, configured in `core.py:CSV_CONFIG`):
- `style` — UI styles (glassmorphism, minimalism, brutalism…) with AI prompt + CSS keywords
- `color` — Color palettes by product type
- `typography` — Font pairings with Google Fonts imports
- `google-fonts` — Full Google Fonts catalog
- `product` — Product-type recommendations (SaaS, e-commerce, portfolio…)
- `landing` — Page structures and CTA strategies
- `chart` — Chart types and library recommendations
- `ux` — Best practices and anti-patterns
- `icons` — Icon libraries and import code
- `react` — React performance guidelines
- `web` — App/web interface guidelines

**Stack search** (`STACK_CONFIG` in `core.py`, 16 stacks):
`react`, `nextjs`, `vue`, `svelte`, `astro`, `swiftui`, `react-native`, `flutter`, `nuxtjs`, `nuxt-ui`, `html-tailwind` (default), `shadcn`, `jetpack-compose`, `threejs`, `angular`, `laravel`.

**Design system generation** (uses `design_system.py`):
```bash
# Generate a full design system recommendation
python3 src/ui-ux-pro-max/scripts/search.py "<query>" --design-system -p "<Project Name>"

# Persist to design-system/<slug>/MASTER.md (Master + Overrides pattern)
python3 src/ui-ux-pro-max/scripts/search.py "<query>" --design-system --persist -p "<Project Name>" [--page "dashboard"]
```

## Architecture

```
src/ui-ux-pro-max/                # Source of Truth
├── data/                         # Canonical CSV databases
│   ├── styles.csv, colors.csv, typography.csv, products.csv,
│   ├── landing.csv, charts.csv, ux-guidelines.csv, icons.csv,
│   ├── google-fonts.csv, react-performance.csv, app-interface.csv,
│   ├── design.csv, draft.csv, ui-reasoning.csv
│   ├── _sync_all.py              # Internal data sync helper
│   └── stacks/                   # 16 stack-specific CSVs
├── scripts/
│   ├── search.py                 # CLI entry point (argparse)
│   ├── core.py                   # BM25 + regex hybrid search
│   └── design_system.py          # Design system generation + persistence
└── templates/
    ├── base/
    │   ├── skill-content.md      # Common SKILL.md body (placeholders)
    │   └── quick-reference.md    # Quick reference (Claude only)
    └── platforms/                # 18 platform configs (claude.json, cursor.json, ...)

cli/                              # npm package: uipro-cli
├── src/
│   ├── index.ts                  # Commander entry: init | versions | update | uninstall
│   ├── commands/                 # Each subcommand
│   ├── utils/
│   │   ├── template.ts           # Renders platform configs into SKILL files
│   │   ├── extract.ts            # Legacy ZIP install path
│   │   ├── github.ts             # GitHub release fetcher
│   │   ├── detect.ts             # Auto-detect installed AI tools
│   │   └── logger.ts
│   └── types/index.ts            # AIType union + AI_FOLDERS map
├── assets/                       # Bundled at publish time (~564KB)
│   ├── data/, scripts/, templates/   # Copies of src/ui-ux-pro-max/*
└── package.json                  # bun build → dist/index.js (bin: uipro)

.claude/skills/ui-ux-pro-max/     # Local Claude Code skill
│   ├── SKILL.md                  # Generated from templates (committed)
│   ├── data → ../../../src/ui-ux-pro-max/data        (symlink)
│   └── scripts → ../../../src/ui-ux-pro-max/scripts  (symlink)
.claude/skills/{banner-design,brand,design,design-system,slides,ui-styling}/
                                  # Other Claude skills bundled in this repo
.claude-plugin/
├── plugin.json                   # Claude marketplace plugin manifest
└── marketplace.json              # Marketplace listing
.github/workflows/                # claude-code-review, claude, python-package-conda
skill.json                        # Top-level skill metadata (uupm.cc homepage)
```

The search engine in `core.py` ranks rows with BM25 over `search_cols` plus regex matching, then returns `output_cols`. Domain auto-detection runs when `--domain` is omitted.

## CLI (uipro-cli)

Built with Bun, distributed on npm as `uipro-cli`, exposes `uipro` binary.

```bash
cd cli
bun install
bun run dev -- init --ai claude        # Run from source
bun run build                          # Bundle to dist/index.js (target: node)
```

Subcommands (`cli/src/index.ts`):
- `uipro init [-a <ai>] [-f] [-o] [-g]` — Install skill (auto-detects AI; `-g` installs to `~/`)
- `uipro versions` — List available versions
- `uipro update [-a <ai>]` — Update to latest
- `uipro uninstall [-a <ai>] [-g]` — Remove skill

The default install path is **template generation** (`utils/template.ts`): it loads `templates/platforms/<ai>.json`, renders `templates/base/skill-content.md` with placeholders (`{{TITLE}}`, `{{DESCRIPTION}}`, `{{SCRIPT_PATH}}`, `{{SKILL_OR_WORKFLOW}}`, `{{QUICK_REFERENCE}}`), and copies `data/` + `scripts/` into the skill folder so each install is self-contained. `--legacy` falls back to the GitHub-release ZIP path.

Per-AI install folders are mapped in `cli/src/types/index.ts:AI_FOLDERS` (e.g. `claude → .claude`, `cursor → .cursor + .shared`, `droid → .factory`).

## Sync Rules

**Source of Truth:** `src/ui-ux-pro-max/`. Never edit the copies in `cli/assets/` or the symlink targets in `.claude/skills/ui-ux-pro-max/` directly.

1. **Data & Scripts** — Edit `src/ui-ux-pro-max/data/*.csv`, `data/stacks/*.csv`, or `scripts/*.py`. Symlinks under `.claude/skills/ui-ux-pro-max/{data,scripts}` resolve automatically.

2. **Templates** — Edit `src/ui-ux-pro-max/templates/base/*.md` (shared body) or `templates/platforms/<ai>.json` (per-platform metadata + frontmatter). When adding a new platform, also add it to `cli/src/types/index.ts` (`AIType`, `AI_TYPES`, `AI_FOLDERS`) and `cli/src/utils/template.ts` (`AI_TO_PLATFORM`).

3. **CLI Assets** — Sync `cli/assets/` from `src/ui-ux-pro-max/` before publishing to npm:
   ```bash
   cp -r src/ui-ux-pro-max/data/*      cli/assets/data/
   cp -r src/ui-ux-pro-max/scripts/*   cli/assets/scripts/
   cp -r src/ui-ux-pro-max/templates/* cli/assets/templates/
   ```

4. **`.claude/skills/ui-ux-pro-max/SKILL.md`** is generated from templates and committed for users who consume the repo directly. Regenerate it when `templates/base/skill-content.md` or `templates/platforms/claude.json` change.

5. **Versioning** — Bump `package.json` (`cli/`), `skill.json`, `.claude-plugin/plugin.json`, and `.claude-plugin/marketplace.json` together.

## Prerequisites

- **Python 3.x** — no external dependencies (stdlib only: `csv`, `re`, `math`, `pathlib`, `argparse`)
- **Bun** — for CLI development and building (Node ≥18 also works for the bundled `dist/`)

There is no automated test suite in this repo. Validate changes by running `search.py` directly against the CSVs (and `--design-system` for end-to-end recommendation flow) and by smoke-testing the CLI via `bun run dev -- init --ai claude` in a scratch directory.

## Git Workflow

Never push directly to `main`. Always:
1. Create a branch: `git checkout -b feat/...` or `fix/...`
2. Commit with descriptive messages
3. Push: `git push -u origin <branch>`
4. Open a PR via GitHub MCP tools (this environment does not have `gh` CLI)
