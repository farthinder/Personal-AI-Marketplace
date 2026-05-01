# Blip for VS Code Copilot Chat

Named after the signal the main character hears drifting through outer space in *Project Hail Mary* — a faint, repeating blip that turns out to be the most important thing in the universe. Evidence first. Never assume. Verify everything.

Blip is an evidence-first coding agent for GitHub Copilot Chat in VS Code. It never shows you broken code. Every quality claim is backed by a verified entry in a SQLite session store — not assertion, not confidence, actual evidence.

## How it works

Blip runs as an orchestrated 10-step pipeline. The main agent (`blip`) handles steps that require judgment and user interaction. Mechanical work is delegated to purpose-built subagents via `runSubagent`.

| Step | What happens | Subagent |
|------|-------------|----------|
| 1. Boost | Clarify intent before touching anything | — (orchestrator) |
| 2. Understand | Restate the goal — what changes, what doesn't | — (orchestrator) |
| 3. Git Hygiene | Check repo state, surface warnings | `blip-git-hygiene` |
| 4. Recall | Query session history for relevant past work | `blip-recall` |
| 5. Survey | Find what already exists in the codebase | `blip-survey` |
| 6. Plan | Map every file change with risk levels | — (orchestrator) |
| 7. Implement | Execute the plan exactly as written | `blip-implement` |
| 8. Review | Adversarial code review (Medium/Large tasks) | `blip-reviewer` + `blip-reviewer-quick` |
| 9. Verify | IDE diagnostics, lint, build, test | `blip-verify` |
| 10. Evidence Bundle | Present verification results | — (orchestrator) |

Steps 3 and 4 run in parallel. Step 8 is skipped for Small/Tiny tasks.

## Installation

### Option A: Global (available in all workspaces)

Install into the global Copilot agents directory. Blip will be available in every project you open.

```bash
mkdir -p ~/.copilot/agents ~/.copilot/skills

# Copy files
cp plugins/blip-vscode/agents/*.agent.md ~/.copilot/agents/
cp -r plugins/blip-vscode/skills/* ~/.copilot/skills/

# Or symlink for automatic updates
for f in plugins/blip-vscode/agents/*.agent.md; do
  ln -sf "$(pwd)/$f" ~/.copilot/agents/
done
ln -sf "$(pwd)/plugins/blip-vscode/skills/blip-git-commit" ~/.copilot/skills/blip-git-commit
ln -sf "$(pwd)/plugins/blip-vscode/skills/blip-issue" ~/.copilot/skills/blip-issue
ln -sf "$(pwd)/plugins/blip-vscode/skills/blip-pr" ~/.copilot/skills/blip-pr
```

### Option B: Per-project (workspace-local)

Install into a single project's `.github/` directory:

```bash
# From your project root
mkdir -p .github
cp -r /path/to/Personal-AI-Marketplace/plugins/blip-vscode/agents/ .github/
cp -r /path/to/Personal-AI-Marketplace/plugins/blip-vscode/skills/ .github/

# Or symlink for automatic updates
ln -s /path/to/Personal-AI-Marketplace/plugins/blip-vscode/agents .github/agents
ln -s /path/to/Personal-AI-Marketplace/plugins/blip-vscode/skills .github/skills
```

### Discovery

VS Code Copilot Chat auto-discovers `.agent.md` and `SKILL.md` files from both locations:
- **Global**: `~/.copilot/agents/` and `~/.copilot/skills/` — available in all workspaces
- **Workspace**: `.github/` in the project root — available only in that project

The session store (`.blip/session.db`) is always created in the **workspace root** regardless of where Blip is installed, keeping session data project-local.

## Usage

In VS Code Copilot Chat, invoke Blip by switching to the agent:

```
@blip add input validation to the login form
```

Or reference it in chat:

```
@blip fix the race condition in the websocket handler
```

## Session store

Blip maintains a SQLite database at `.blip/session.db` in the project root. The `.blip/` directory is automatically added to `.gitignore` — session data is local and never committed. Fallback: `.blip/session.jsonl` if sqlite3 is unavailable.

## Task classification

| Size | Examples | Review | Min verification signals |
|------|----------|--------|--------------------------|
| Tiny | Single line, rename, config value | None | 0 |
| Small | Single function, obvious fix | None | 1 |
| Medium | Bug fix, feature addition, refactor | 1 reviewer | 2 |
| Large | New feature, multi-file, auth / crypto / payments | 3 reviewers | 3 |

Tiny tasks skip the subagent pipeline entirely.

## Included skills

- **blip-git-commit** — Conventional Commits with issue references and "why" explanations
- **blip-issue** — Turn rough ideas into well-structured, actionable issues
- **blip-pr** — Create pull requests with full reviewer context

## VS Code-specific adaptations

Compared to the Claude Code variant (`plugins/blip/`):

| Aspect | blip (Claude Code) | blip-vscode |
|--------|-------------------|-------------|
| Tool names | `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `Agent` | `run_in_terminal`, `read_file`, `replace_string_in_file`, `create_file`, `grep_search`, `semantic_search`, `file_search`, `runSubagent` |
| Subagent dispatch | Claude `Agent` tool | VS Code `runSubagent` |
| IDE integration | None | `get_errors` for real-time diagnostics |
| Model control | Per-agent model pinning (Sonnet/Haiku) | Single model (user's selected model) |
| Entry point | `/blip` command | `@blip` in Copilot Chat |

## Requirements

- VS Code with GitHub Copilot Chat extension
- `sqlite3` on PATH (for session store; falls back to JSONL)
- `gh` CLI (for issue/PR skills; optional)
