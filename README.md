# Claude Code Secure Development Template

A ready-to-use template for AI-assisted software development with [Claude Code](https://claude.ai/code) and the [Get Shit Done (GSD)](https://github.com/pchaganti/gsd-cc) meta-prompting framework, running inside a hardened dev container with egress network restrictions.

## What's included

| Component                        | Purpose                                                            |
| -------------------------------- | ------------------------------------------------------------------ |
| `.devcontainer/`                 | Docker-based dev container with Claude Code pre-installed          |
| `.devcontainer/init-firewall.sh` | iptables/ipset firewall that limits network egress to an allowlist |

## Security model

The dev container enforces **network-level egress restrictions** at startup via `init-firewall.sh`. All outbound traffic is dropped by default; only the following destinations are permitted:

- `api.anthropic.com` — Claude API
- `registry.npmjs.org` — npm packages
- GitHub IP ranges (fetched dynamically from `api.github.com/meta`)
- `marketplace.visualstudio.com`, `vscode.blob.core.windows.net`, `update.code.visualstudio.com` — VS Code extensions
- `statsig.anthropic.com`, `statsig.com`, `sentry.io` — Anthropic telemetry

This prevents Claude Code from exfiltrating code or credentials to arbitrary hosts, even in fully autonomous mode.

## Getting started

### Prerequisites

- [VS Code](https://code.visualstudio.com/) with the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
- Docker Desktop (or Docker Engine on Linux)
- A GitHub personal access token (for the firewall to fetch GitHub IP ranges)

### Setup

1. **Use this template** — click "Use this template" on GitHub, or clone directly:

   ```sh
   git clone <this-repo> my-project
   cd my-project
   ```

2. **Add your credentials** — copy the example and fill in your tokens:

   ```sh
   cp credentials.env.example credentials.env
   # Edit credentials.env and add your GH_TOKEN and ANTHROPIC_API_KEY
   ```

3. **Open in VS Code and reopen in container:**

   ```
   Ctrl+Shift+P → Dev Containers: Reopen in Container
   ```

   The container will build, install Claude Code, apply GSD, and start the firewall automatically.

4. **Start a new project with GSD:**
   ```
   /gsd-new-project
   ```
   GSD will walk you through defining your project, creating a roadmap, and planning your first phase.

## credentials.env

The `credentials.env` file is loaded into the container at startup (never committed to git). It should contain:

```env
GH_TOKEN=ghp_your_github_token
ANTHROPIC_API_KEY=sk-ant-your-api-key
```

A `credentials.env.example` file is provided as a reference. The actual `credentials.env` is in `.gitignore`.

## GSD workflow overview

GSD structures development into **phases** with a discuss → plan → execute → verify cycle:

```
/gsd-new-project      # Initialize project with roadmap and requirements
/gsd-discuss-phase    # Gather context and constraints before planning
/gsd-plan-phase       # Generate a detailed PLAN.md for a phase
/gsd-execute-phase    # Execute the plan with atomic commits
/gsd-verify-work      # Validate against acceptance criteria
/gsd-next             # Automatically advance to the next step
```

Key commands for day-to-day use:

| Command            | What it does                                               |
| ------------------ | ---------------------------------------------------------- |
| `/gsd-progress`    | Show current phase status and route to next action         |
| `/gsd-fast`        | Execute a small task inline without full planning overhead |
| `/gsd-debug`       | Systematic bug investigation with persistent state         |
| `/gsd-resume-work` | Restore full context from a previous session               |
| `/gsd-help`        | List all available GSD commands                            |

## Dev container details

The container is based on `node:20` and includes:

- Claude Code CLI (latest)
- Git with [delta](https://github.com/dandavison/delta) diff viewer
- Zsh with Powerline10k theme, fzf, and git plugin
- GitHub CLI (`gh`)
- VS Code extensions: Claude Code, ESLint, Prettier, GitLens
- `iptables`, `ipset`, `aggregate`, `dnsutils` for firewall management

Claude Code runs as the unprivileged `node` user. The firewall script is the only command granted passwordless `sudo`.

The Claude config directory (`/home/node/.claude`) and bash history are stored in named Docker volumes so they persist across container rebuilds.

## Claude Code hooks

The following hooks run automatically and are configured in `.claude/settings.json`:

| Hook                     | Trigger                     | Purpose                                               |
| ------------------------ | --------------------------- | ----------------------------------------------------- |
| `gsd-check-update.js`    | Session start               | Notify when a GSD update is available                 |
| `gsd-context-monitor.js` | After Bash/Edit/Write/Agent | Track context window usage                            |
| `gsd-prompt-guard.js`    | Before Write/Edit           | Prevent accidental overwrites of planning artifacts   |
| `gsd-read-guard.js`      | Before Write/Edit           | Validate file read before edit                        |
| `gsd-validate-commit.sh` | Before Bash                 | Enforce GSD commit message format                     |
| `gsd-statusline.js`      | Status line                 | Show current phase and plan in Claude Code status bar |

## Customizing the allowlist

To permit additional hosts, edit `init-firewall.sh` and add them to the domain loop:

```sh
for domain in \
    "registry.npmjs.org" \
    "api.anthropic.com" \
    "your-internal-registry.example.com" \   # add here
    ...
```

Rebuild and restart the container to apply changes.
