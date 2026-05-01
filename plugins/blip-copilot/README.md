# Blip for Copilot

Named after the signal the main character hears drifting through outer space in *Project Hail Mary* — a faint, repeating blip that turns out to be the most important thing in the universe. Evidence first. Never assume. Verify everything.

Blip is an evidence-first coding agent for GitHub Copilot CLI. It never shows you broken code. Every quality claim is backed by a verified entry in a SQLite session store — not assertion, not confidence, actual evidence.

## How it works

Blip runs as an orchestrated 10-step pipeline. The main agent (`blip`) handles steps that require judgment and user interaction. Mechanical work is delegated to purpose-built subagents.

| Step | What happens | Model |
|------|-------------|-------|
| 1. Boost | Clarify intent before touching anything | Sonnet |
| 2. Understand | Restate the goal — what changes, what doesn't | Sonnet |
| 3. Git Hygiene | Check repo state, surface warnings | Haiku |
| 4. Recall | Query session history for relevant past work | Haiku |
| 5. Survey | Find what already exists in the codebase | Sonnet |
| 6. Plan | Map every file change with risk levels | Sonnet |
| 7. Implement | Execute the plan exactly as written | Haiku |
| 8. Review | Adversarial code review (Medium/Large tasks) | Sonnet |
| 9. Verify | Lint, format, build, test | Haiku |
| 10. Evidence Bundle | Present verification results | Sonnet |

Steps 3 and 4 run in parallel. Step 8 is skipped for Small/Tiny tasks.

## Installation

### Option A: Global (available in all workspaces)

Install into the global Copilot agents directory. Blip will be available in every project you open.

```bash
mkdir -p ~/.copilot/agents ~/.copilot/skills

# Copy (snapshot — re-run to pick up updates)
cp /path/to/Personal-AI-Marketplace/plugins/blip-copilot/agents/*.agent.md ~/.copilot/agents/
cp -r /path/to/Personal-AI-Marketplace/plugins/workflow-skills/skills/* ~/.copilot/skills/

# Or symlink (live — changes in the repo are reflected immediately)
MARKETPLACE=/path/to/Personal-AI-Marketplace
for f in "$MARKETPLACE/plugins/blip-copilot/agents"/*.agent.md; do
  ln -sf "$f" ~/.copilot/agents/
done
for skill in git-commit issue pr; do
  ln -sf "$MARKETPLACE/plugins/workflow-skills/skills/$skill" ~/.copilot/skills/$skill
done
```

### Option B: Per-project (workspace-local)

Install into a single project's `.github/` directory. Agents are scoped to that workspace only.

```bash
# From your project root
mkdir -p .github/agents
cp -r /path/to/Personal-AI-Marketplace/plugins/blip-copilot/agents/ .github/agents/

# Or symlink for automatic updates
ln -s /path/to/Personal-AI-Marketplace/plugins/blip-copilot/agents .github/agents
```

### Discovery

Copilot Chat auto-discovers `.agent.md` files from both locations:
- **Global**: `~/.copilot/agents/` — available in all workspaces
- **Workspace**: `.github/agents/` in the project root — available only in that project

The session store (`.blip/session.db`) is always created in the **workspace root** regardless of where Blip is installed, keeping session data project-local.

## Usage

```bash
# Invoke directly
copilot --agent=blip --prompt "add input validation to the login form"

# Or from within an interactive session
/agent blip
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

## File structure

```
plugins/blip-copilot/
├── plugin.json
├── README.md
└── agents/
    ├── blip.agent.md              # Sonnet orchestrator
    ├── blip-git-hygiene.agent.md  # Haiku — Step 3
    ├── blip-recall.agent.md       # Haiku — Step 4
    ├── blip-survey.agent.md       # Sonnet — Step 5
    ├── blip-implement.agent.md    # Haiku — Step 7
    ├── blip-reviewer.agent.md     # Sonnet — Step 8
    └── blip-verify.agent.md       # Haiku — Step 9
```
