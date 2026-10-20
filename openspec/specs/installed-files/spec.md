# installed-files Specification

## Purpose
Defines how this repository's Installed files reach host tools. The Installer is the only path, every change to installed files is recorded as one Release in the root `CHANGELOG.md`, and each role agent has one name that is the same on every host.

## Requirements

### Requirement: The Installer is the only install path
`tools/build-agents.sh` SHALL write only under `agents/`.

#### Scenario: Regeneration stays under agents/
- **WHEN** `tools/build-agents.sh` runs in a clean checkout
- **THEN** `git status --porcelain` lists no path outside `agents/`

### Requirement: Releases are recorded in CHANGELOG.md
The repository root SHALL carry `CHANGELOG.md`. It SHALL start with the line `# Changelog`, followed by one `## <major>.<minor>.<patch>` heading per Release. Version order SHALL be strictly descending, so the first heading is the newest Release. Each heading SHALL be followed by exactly one non-empty paragraph, and no heading or paragraph SHALL carry a date. The first Release is `1.0.0`.

#### Scenario: Headings are newest first
- **WHEN** `tests/run.sh` reads every line of `CHANGELOG.md` matching `^## [0-9]+\.[0-9]+\.[0-9]+$`
- **THEN** there is at least one of them, `## 1.0.0` is one of them, each version is numerically greater than the next one, and line 1 is `# Changelog`

#### Scenario: Each Release has one undated paragraph
- **WHEN** `tests/run.sh` splits `CHANGELOG.md` at its `## ` headings
- **THEN** every section holds exactly one non-empty paragraph, and no heading line contains anything after the version

### Requirement: A change to installed files carries exactly one new Release
`tools/check-release-bump.sh <base-ref>` SHALL compare `HEAD` with the merge-base of `<base-ref>` and `HEAD`. When no file under `skills/openspec-orchestrator/` or `agents/` differs, it SHALL exit 0. Otherwise it SHALL exit non-zero and name the cause on stderr in any of these cases: the number of `CHANGELOG.md` headings at `HEAD` that are absent at the merge-base is not exactly one; the new version is not greater than every earlier version; or an earlier Release's paragraph was edited or removed. On success it SHALL exit 0 and print the new version. CI SHALL run the script on every pull request against `origin/<base branch>`.

#### Scenario: One new Release passes
- **WHEN** a temporary git repo's branch changes a file under `skills/openspec-orchestrator/` and adds one heading `## 1.1.0` above the base's `## 1.0.0`
- **THEN** `tools/check-release-bump.sh main` (run against that repo) exits 0 and prints `1.1.0`

#### Scenario: No new Release fails
- **WHEN** the branch changes a file under `agents/` and leaves `CHANGELOG.md` unchanged
- **THEN** the script exits non-zero and stderr says exactly one new Release was expected

#### Scenario: Two new Releases fail
- **WHEN** the branch changes an installed file and adds `## 1.2.0` and `## 1.1.0`
- **THEN** the script exits non-zero and stderr says exactly one new Release was expected

#### Scenario: A Release not above the previous one fails
- **WHEN** the base's newest Release is `1.1.0` and the branch changes an installed file and adds `## 1.0.1`
- **THEN** the script exits non-zero and stderr names `1.0.1` as not above `1.1.0`

#### Scenario: An edited earlier Release fails
- **WHEN** the branch changes an installed file, adds `## 1.1.0`, and rewrites the paragraph of `## 1.0.0`
- **THEN** the script exits non-zero and stderr names `1.0.0` as edited or removed

#### Scenario: A change outside installed files needs no Release
- **WHEN** the branch changes only `README.md`
- **THEN** the script exits 0 without requiring a new heading

#### Scenario: CI runs the check on pull requests
- **WHEN** `tests/run.sh` reads `.github/workflows/ci.yml`
- **THEN** a step guarded by `github.event_name == 'pull_request'` runs `tools/check-release-bump.sh "origin/${{ github.base_ref }}"`

### Requirement: The Role agent name is the same on every host
On every host tool, `next` SHALL print `agent: openspec-<role>-<tier>` with no tool suffix. `agents/` SHALL contain exactly the five directories `claude-code`, `cursor`, `copilot`, `codex` and `opencode`. Each SHALL hold, for every role and tier, `openspec-<role>-<tier>/AGENT.md` whose `name:` line is that Role agent name. The Installer SHALL file each OpenCode agent as `<Role agent name>.md`. Each Claude Code agent SHALL keep the role's effort (`high` for proposer, critic and verifier; `medium` for unit-critic and worker). No agent's tool allowlist SHALL include `Agent`, and no checker's (critic, verifier, unit-critic) SHALL include `Edit`. Each agent's body SHALL be the role prompt verbatim.

#### Scenario: OpenCode next names the unsuffixed agent
- **WHEN** `next` runs for a change in a store whose `orchestration.tool` is `opencode`
- **THEN** it prints the line `agent: openspec-proposer-standard` and no line containing `-opencode`

#### Scenario: Only the per-tool sets exist
- **WHEN** `tests/run.sh` lists the directories directly under `agents/`
- **THEN** they are exactly `claude-code cursor copilot codex opencode`, and each holds 20 `AGENT.md` files whose `name:` equals their directory name

#### Scenario: OpenCode install uses the plain name
- **WHEN** `tools/install.sh --tool opencode` runs with a temporary `HOME`
- **THEN** `$HOME/.config/opencode/agents/openspec-critic-max.md` is a symlink, and no file under `$HOME/.config/opencode/agents/` matches `*-opencode.md`

#### Scenario: Claude Code agents keep effort and tool policy
- **WHEN** `tests/run.sh` reads `agents/claude-code/openspec-<role>-<tier>/AGENT.md` for every role and tier
- **THEN** `effort:` matches the role's effort, no `tools:` line names `Agent`, the critic, verifier and unit-critic `tools:` lines do not name `Edit`, and the body after the frontmatter equals `skills/openspec-orchestrator/scripts/roles/<role>.md`

#### Scenario: Generator output is committed
- **WHEN** `tests/run.sh` hashes every file under `agents/`, runs `tools/build-agents.sh`, and hashes again
- **THEN** the two hashes are equal

### Requirement: Docs describe only the Installer
`README.md`, `skills/openspec-orchestrator/SKILL.md`, `skills/openspec-orchestrator/AUTONOMOUS-ORCHESTRATION.md` and `skills/openspec-orchestrator/CONTEXT.md` SHALL name `tools/install.sh` as the way to install the skill and agents. None of them SHALL name the legacy `agents/openspec-<role>/` set or an `-opencode` agent-name suffix. README's Tests paragraph SHALL state that CI requires one new `CHANGELOG.md` Release when `skills/openspec-orchestrator/` or `agents/` change. README SHALL keep the note that agent definitions load when a session starts. The `context:` block of `openspec/config.yaml` SHALL name `skills/openspec-orchestrator/` and `agents/` as the installed files, and SHALL state that every change to them adds exactly one Release to `CHANGELOG.md`.

#### Scenario: No legacy set or suffix in docs
- **WHEN** `tests/run.sh` runs `grep -cF -- 'agents/openspec-<role>/'` and `grep -cF -- '-opencode'` on each of the four docs
- **THEN** every count is 0

#### Scenario: Installer named in SKILL.md Step 0
- **WHEN** Step 0 of SKILL.md (the line starting `0. **Host tool and role agents:**`) is read
- **THEN** it contains `tools/install.sh --tool <tool>`

#### Scenario: README states the Release rule and keeps the session-start note
- **WHEN** README.md is read
- **THEN** it contains `CHANGELOG.md` and `tools/check-release-bump.sh` in the same line, and contains `Agent definitions load when a session starts`

#### Scenario: Config context states the Release rule
- **WHEN** the `context:` block of `openspec/config.yaml` is read
- **THEN** it contains `CHANGELOG.md`, `agents/` and `exactly one`
