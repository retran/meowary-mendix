# Meowary Mendix

A plugin for [Meowary](https://github.com/retran/meowary) that adds Mendix development support — mxcli integration, MDL authoring skills, and project lifecycle workflows for Mendix applications.

## What This Is

Meowary Mendix is a port of mxcli's original agentic harness — the skills, workflows, and commands that ship with mxcli for AI-assisted Mendix development — repackaged as a Meowary plugin.

It extends Meowary with everything needed to build Mendix applications using [mxcli](https://github.com/mendixlabs/mxcli) and MDL (Mendix Definition Language). It adds:

- **1 slash command** — `/mendix` routes freeform intent to the right workflow
- **11 workflows** — entity creation, CRUD scaffolding, linting, testing, validation, and more
- **50+ skill files** — domain model, microflows, pages, security, navigation, integrations, migration, and testing
- **1 reference file** — complete MDL command reference with syntax quick-reference and linting rules

The agent never shows raw MDL. It describes changes in plain language, gets your approval, validates the script, and executes silently.

## Installation

Copy the `.opencode/` directory over your Meowary instance:

```sh
cp -R .opencode/ ~/your-meowary-instance/.opencode/
```

This adds new files without modifying existing Meowary configuration. No changes to `AGENTS.md`, `opencode.json`, or any base skill are required.

### Prerequisites

- A working [Meowary](https://github.com/retran/meowary) instance
- A Mendix `.mpr` project file
- `mxcli` installed in your Mendix project (see below)

### Setting up mxcli

[mxcli](https://github.com/mendixlabs/mxcli) is the CLI tool that reads and writes Mendix projects. The plugin expects it as a local binary in your Mendix project root.

1. **Download mxcli** from [GitHub releases](https://github.com/mendixlabs/mxcli/releases) for your platform (macOS, Linux, Windows)

2. **Place the binary** in your Mendix project root (next to `project.mpr`):

   ```sh
   # Example for macOS
   curl -L https://github.com/mendixlabs/mxcli/releases/latest/download/mxcli-darwin-arm64 -o ./mxcli
   chmod +x ./mxcli
   ```

3. **Verify it works:**

   ```sh
   ./mxcli -p project.mpr -c "SHOW MODULES"
   ```

4. **(Optional) Set up mxbuild** for full project validation (same checks as Studio Pro):

   ```sh
   ./mxcli setup mxbuild -p project.mpr
   ```

The binary is always invoked as `./mxcli` — it is intentionally not added to PATH. This ensures each project can pin its own version.

### Codebase context

Create a codebase file for your Mendix project at `codebases/<project>.md` in your Meowary instance. The mendix skill reads it to determine:

- Project file name (`<project>.mpr`)
- Working directory
- App URL (default: `http://localhost:8080`)

## Usage

### Quick start

```
/mendix explore                    # Browse project structure
/mendix create entity Customer     # Create an entity
/mendix create crud Order          # Scaffold entity + overview + edit pages
/mendix lint                       # Run linting and best-practices report
/mendix test                       # Run Playwright tests against running app
```

### Available workflows

| Command | Workflow | What it does |
|---------|----------|--------------|
| `/mendix explore` | `mendix-explore` | Browse structure, describe elements, catalog queries |
| `/mendix create entity` | `mendix-create-entity` | Create entity with attributes and associations |
| `/mendix create crud` | `mendix-create-crud` | Full CRUD scaffold (entity + pages + microflows) |
| `/mendix lint` | `mendix-lint` | Lint project + scored best-practices report |
| `/mendix test` | `mendix-test` | Playwright UI tests + OQL data assertions |
| `/mendix check script` | `mendix-check-script` | Validate MDL script syntax and references |
| `/mendix validate` | `mendix-validate-project` | Run `mx check` (same checks as Studio Pro) |
| `/mendix refresh catalog` | `mendix-refresh-catalog` | Rebuild metadata catalog for queries |
| `/mendix diff script` | `mendix-diff-script` | Preview what a script would change |
| `/mendix diff local` | `mendix-diff-local` | Show uncommitted git changes |
| Any other write intent | `mendix-write` | Create/modify microflows, pages, security, navigation |

### Skills

The `mendix` skill loads automatically when working in a Mendix project. It provides:

- Core conventions (quoted identifiers, validation-before-execution, never show raw MDL)
- Docker lifecycle commands (run, reload, fresh, CSS-only)
- Sub-skill routing — 50 domain-specific skills loaded on demand

Key sub-skills:

| Domain | Skills |
|--------|--------|
| Domain model | `generate-domain-model`, `mdl-entities`, `write-oql-queries` |
| Microflows | `write-microflows`, `write-nanoflows`, `patterns-crud`, `patterns-data-processing` |
| Pages | `create-page`, `alter-page`, `overview-pages`, `master-detail-pages` |
| Security | `manage-security`, `manage-navigation` |
| Integration | `rest-client`, `odata-data-sharing`, `database-connections`, `java-actions` |
| Testing | `test-app`, `test-microflows`, `verify-with-oql` |
| Quality | `assess-quality`, `write-lint-rules`, `check-syntax` |
| Migration | `assess-migration`, `migrate-k2-nintex`, `migrate-oracle-forms` |

## Structure

```
meowary-mendix/
└── .opencode/
    ├── commands/
    │   └── mendix.md                  # /mendix router command
    ├── workflows/
    │   ├── mendix-check-script.md
    │   ├── mendix-create-crud.md
    │   ├── mendix-create-entity.md
    │   ├── mendix-diff-local.md
    │   ├── mendix-diff-script.md
    │   ├── mendix-explore.md
    │   ├── mendix-lint.md
    │   ├── mendix-refresh-catalog.md
    │   ├── mendix-test.md
    │   ├── mendix-validate-project.md
    │   └── mendix-write.md
    └── skills/mendix/
        ├── SKILL.md                   # Main skill — conventions, Docker, sub-skill index
        ├── reference/
        │   └── mdl-commands.md        # Full command reference + syntax + linting rules
        ├── agents/
        ├── alter-page/
        ├── ... (50 sub-skill directories)
        └── xpath-constraints/
```

## License

[Apache 2.0](LICENSE)
