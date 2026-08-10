# Feature Proposal: Default extras targets and base builds from the git root

Issue: not yet filed. Per [CONTRIBUTING.md](../CONTRIBUTING.md) an issue should precede this proposal; opening one is the first step if the direction looks worth pursuing.

## Problem

Two gaps that show up together when a whole agent setup — skills, agents, and extras — is kept in one git repository under `git_root: root`.

### 1. Extras have no defaults

Skills and agents are backed by a built-in target registry. `internal/config/targets.yaml` carries 66 targets, each with a global and project path:

```yaml
- name: claude
  skills:
    global: "~/.claude/skills"
    project: ".claude/skills"
  agents:
    global: "~/.claude/agents"
    project: ".claude/agents"
```

That registry is what makes `skillshare init --all-targets` possible — the user names no paths at all, and the knowledge of where each of those CLIs keeps its skills lives in one reviewable data file.

Extras get none of this. Every extra is hand-wired, one literal absolute path per tool:

```bash
skillshare extras init rules --target ~/.claude/rules --target ~/.cursor/rules --target ~/.codex/rules
```

The only defaults anywhere in the codebase are three hardcoded placeholder strings in the interactive wizard (`cmd/skillshare/extras_init_tui.go:370-377`), used as display hints and not reachable from the non-interactive path. So the same task — "point this resource at the tools I use" — is one flag for skills and N hand-written paths for extras. Those paths are exactly the sort of per-tool trivia the registry exists to absorb, and today the fastest way to learn that Claude Code keeps slash commands in `~/.claude/commands` is to read skillshare's TUI source.

### 2. Nothing derives configuration from the root

With `git_root: root`, the versioned directory already has a conventional shape:

```
<root>/
  skills/
  agents/
  extras/
    rules/
    commands/
  config.yaml
```

The layout is self-describing. The tool does not read it. Cloning that repo onto a second machine restores every file but no configuration — `sources.skills`, `sources.agents`, `sources.extras`, and one `extras:` block per extra all have to be re-entered by hand, correctly, before a single `sync` does anything. The information needed to write that config is sitting in the directory listing.

Who is affected: anyone using `git_root: root` to version an agent setup as a single repo, and anyone syncing rules or commands to more than one tool.

## Proposed Solution

Two parts. Part 1 is independently useful; Part 2 builds on its lookups.

### Part 1 — extras in the target registry

Extend `targetSpec` with an `extras` map keyed by extra kind, each holding the same `{global, project}` pair the existing resource kinds use:

```yaml
- name: claude
  skills:
    global: "~/.claude/skills"
    project: ".claude/skills"
  agents:
    global: "~/.claude/agents"
    project: ".claude/agents"
  extras:
    rules:
      global: "~/.claude/rules"
      project: ".claude/rules"
    commands:
      global: "~/.claude/commands"
      project: ".claude/commands"
```

Scoped initially to two kinds, **`rules`** and **`commands`**. Both are broadly supported across tools, both are plain markdown — no executable bits, no format conversion, none of the complications that `hooks` or `mcp` would bring — and both are already the canonical examples throughout the docs and the wizard.

New lookups, mirroring the existing skills/agents functions in `internal/config/targets.go` one for one:

```go
func DefaultExtraTargets(kind string) map[string]TargetConfig   // global
func ProjectExtraTargets(kind string) map[string]TargetConfig   // project
func LookupGlobalExtraTarget(kind, name string) (TargetConfig, bool)
func LookupProjectExtraTarget(kind, name string) (TargetConfig, bool)
func KnownExtraKinds() []string
```

On the CLI, `extras init` learns to accept target **names** where it currently accepts only paths, plus `--all-targets` to match `init`:

```bash
skillshare extras init rules --all-targets
skillshare extras init commands --target claude --target cursor
```

Resolution rule: an argument that contains a path separator, starts with `~`, or starts with `.` is a literal path and behaves exactly as it does today. A bare word is looked up in the registry for that kind. An unknown name — or a known target that defines no path for that kind — is an error that lists the targets which do. Every existing invocation keeps working unchanged.

The wizard's three hardcoded strings are replaced by the registry, so it starts suggesting real paths for the tools the user actually has configured.

### Part 2 — base build from the git root

`skillshare init --from-root <path>` derives a configuration from a directory that already has the conventional layout.

Given `<path>`:

- Set `sources.skills` to `<path>/skills`, `sources.agents` to `<path>/agents`, `sources.extras` to `<path>/extras` — **each only if that directory exists**. A repo that is only `agents/` and `extras/`, with no `skills/`, is a valid base build; so is the reverse. This is what makes the "agent root path" case fall out for free rather than needing its own flag.
- Set `git_root: root` when `<path>/.git` exists and every resolved source lives beneath it.
- For each `<path>/extras/<name>/`, emit an `extras:` entry whose targets come from the Part 1 registry for that kind — **restricted to targets already present in `targets:`**, so a base build never invents targets for tools the user does not have.
- Report and skip any extras directory whose name is not a known kind, rather than guessing at a path for it.

`--dry-run` prints the derived config without writing. Re-running reconciles rather than duplicates; `internal/config/reconcile.go` and `project_reconcile.go` already establish the shape for this.

## Alternatives Considered

- **Hardcode default extras paths inside `extras init`** instead of the registry. Less code, but it splits per-tool path knowledge across two places. `targets.yaml` is already the single file contributors know to edit when a new CLI ships, and it is reviewable as data rather than as Go.
- **A separate `skillshare base` or `skillshare build` verb** instead of an `init` flag. Conceptually tidier, but it adds a top-level command for something that is a variant of what `init` already does — `init` handles `--all-targets`, `--git-root root`, `--source`, `--subdir`, and a headless scope switch on an existing config.
- **Ship a template repository** and document the layout. Gets the shape right once, but does nothing for the second machine: the config still has to be reconstructed by hand, which is the actual pain.
- **Derive extras targets from every registry target** rather than only configured ones. Would write all 66 targets' worth of paths into `config.yaml` for a user with two tools installed.
- **Document the pattern and change nothing.** Leaves the extras path list undiscoverable and the root layout inert. Both problems are about knowledge that the tool could hold but currently expects the user to.

## Scope

- [ ] Small (1-3 files, < 200 lines)
- [x] Medium (3-10 files, 200-500 lines) — Part 1
- [ ] Large (10+ files, 500+ lines) — both parts together

Part 1:

- `internal/config/targets.yaml` — extras block for the targets that have verified paths
- `internal/config/targets.go` — `targetSpec.Extras` field and the five lookups
- `cmd/skillshare/extras_init.go` — name-vs-path resolution, `--all-targets`
- `cmd/skillshare/extras_init_tui.go` — replace hardcoded placeholders with registry lookups
- `schemas/config.schema.json` — no change expected; the registry is internal and output is ordinary `extras:` entries
- Docs: `website/docs/reference/commands/extras.md`, `website/docs/reference/targets/configuration.md`
- Tests alongside `targets_test.go`, `config_extras_test.go`

Part 2:

- `cmd/skillshare/init.go` — `--from-root` flag, scan, derivation, `--dry-run`
- `internal/config` — reconcile derived config against existing
- Docs: `website/docs/reference/commands/init.md`, `website/docs/understand/source-and-targets.md`
- Tests

Landing Part 1 first is suggested — it stands alone, and Part 2 depends on its lookups.

## Open Questions

- **Path verification.** The paths below are the ones this repository's own docs already use as examples, which makes them a starting point, not a verified set. Each needs checking against the tool's official documentation before shipping as a default, because a wrong default writes files into a directory the tool never reads — worse than having no default at all. Which of these are confirmed?

  | Kind | claude | cursor | codex | opencode |
  |---|---|---|---|---|
  | `rules` | `~/.claude/rules` | `~/.cursor/rules` | `~/.codex/rules` | — |
  | `commands` | `~/.claude/commands` | — | — | `~/.config/opencode/commands` |

- **Project-only kinds.** Does `rules` have a genuine *global* directory for Cursor, or is it project-scoped only (`.cursor/rules/`)? The schema should be able to express a kind that exists for one scope and not the other, rather than forcing an invented global path.
- **Kind naming vs. tool naming.** The registry would key on `commands`, but Codex calls the same concept *prompts* (`~/.codex/prompts`). Should a kind carry aliases so one `<root>/extras/commands/` directory can feed both, or should users name directories per-tool and give up cross-tool sync for that kind?
- **Should registry defaults ever apply implicitly?** That is, may `sync extras` fall back to registry paths for a configured extra whose `targets` list is empty, or must targets always be materialized into `config.yaml` first? Explicit-in-config is more debuggable; implicit is less to keep in sync.
- **Conflict policy for `--from-root`.** When the derived config disagrees with an existing `config.yaml` — a different `sources.skills`, or an extra whose targets were hand-edited — does it merge, prompt, or refuse without `--force`?
- **Reserved extra names.** `ValidateExtraName` currently reserves `skills` and `extras`. A root scan meets `<root>/agents/` as a sibling of `extras/`, not as an extra — but should `agents` be reserved too, so one word cannot mean two things?
- **Interaction with [proposal 171](./171-extras-git-integration.md).** A base build that sets `git_root: root` with all three sources beneath it is precisely the configuration 171 discusses for extras-aware `commit`/`push`/`pull`. Worth confirming the two land compatibly.
