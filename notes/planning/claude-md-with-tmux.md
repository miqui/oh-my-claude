# Project Instructions

## Architecture

- Monorepo: `apps/api` (Go), `apps/web` (Next.js), `packages/shared` (TypeScript)
- Infrastructure: AWS EKS, Crossplane, ArgoCD
- Database: PostgreSQL 16 via managed RDS
- API spec: OpenAPI 3.2, generated SDK via openapi-generator

## Coding Conventions

- Go: follow `golangci-lint` defaults, error wrapping with `fmt.Errorf("context: %w", err)`
- TypeScript: strict mode, no `any`, prefer `zod` for runtime validation
- Commits: conventional commits (`feat:`, `fix:`, `chore:`, `docs:`)
- PRs: one concern per PR, squash merge to `main`

## Build & Test

```bash
# API
cd apps/api && go test ./...

# Web
cd apps/web && pnpm test

# Full suite
make test-all
```

## Agent Team Rules

When working as part of a multi-agent team (agent teams enabled), ALL agents
MUST read and follow the tmux workflow instructions before beginning any work:

👉 **See [AGENTS-TMUX.md](./AGENTS-TMUX.md) for tmux session, visibility, and communication rules.**

Key points (details in AGENTS-TMUX.md):
- Use `agent-<role>` session naming
- Echo intent before every significant command
- Tag all output with your role: `>>> [role] message`
- Never run `clear` — preserve scrollback for human review
- Human input to your pane overrides all other tasks

## Agent Role Definitions

When spawning a team, use these roles unless the task requires something different:

| Role        | Session Name     | Scope                                    |
|-------------|------------------|------------------------------------------|
| Lead        | `agent-lead`     | Coordination, PR assembly, final review  |
| Backend     | `agent-backend`  | `apps/api`, DB migrations, API contracts |
| Frontend    | `agent-frontend` | `apps/web`, components, state management |
| Infra       | `agent-infra`    | IaC, Crossplane compositions, ArgoCD     |
| Tests       | `agent-tests`    | Integration tests, E2E, coverage gates   |

## File Ownership (Conflict Prevention)

Agents MUST NOT edit files outside their scope without explicit lead approval:

- `apps/api/**` → agent-backend only
- `apps/web/**` → agent-frontend only
- `infra/**`, `crossplane/**` → agent-infra only
- `tests/integration/**` → agent-tests only
- `packages/shared/**` → requires lead coordination (multiple consumers)

## Memory

- Auto-memory is enabled — Claude will persist learnings in `~/.claude/projects/<project>/memory/`
- For session-specific notes, update MEMORY.md via `/memory`
- Do NOT store secrets or credentials in any memory file
