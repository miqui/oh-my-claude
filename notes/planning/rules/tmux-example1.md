# Agent Team tmux Instructions

> These instructions govern how agents use tmux so the human operator
> can observe all work in real time.

## Session Naming Convention

All tmux sessions MUST use the prefix `agent-` followed by your role name:

```
agent-lead
agent-backend
agent-frontend
agent-tests
agent-infra
```

If a session with your assigned name already exists, attach to it — do NOT create a duplicate.

## Window & Pane Layout

Each agent session MUST maintain at least two named windows:

| Window | Name       | Purpose                                      |
|--------|------------|----------------------------------------------|
| 0      | `work`     | Primary workspace — code edits, builds, etc. |
| 1      | `logs`     | Tail logs, test output, long-running procs   |

Create windows on session init:

```bash
tmux rename-window -t agent-<role>:0 'work'
tmux new-window -t agent-<role> -n 'logs'
tmux select-window -t agent-<role>:0
```

## Visibility Rules

The human watches your tmux panes. Follow these rules so they can track progress:

1. **Echo intent before acting.** Before running any significant command, print a
   one-line summary of what you are about to do:
   ```bash
   echo ">>> [backend] Installing dependencies..."
   npm install
   ```

2. **No silent failures.** If a command fails, echo the error context before
   attempting a fix:
   ```bash
   echo ">>> [backend] Build failed — missing env var DB_HOST. Fixing..."
   ```

3. **Use the `logs` window for long-running processes.** Start servers, watchers,
   and test suites in window 1 so window 0 stays uncluttered:
   ```bash
   tmux send-keys -t agent-backend:logs 'npm run dev 2>&1 | tee /tmp/agent-backend-dev.log' Enter
   ```

4. **Preserve scrollback.** Never run `clear` or `reset`. The human may scroll
   back to review earlier output.

5. **Tag output with your role.** Prefix echo statements with your agent name
   in brackets so the human can identify which agent produced what when
   viewing a combined view:
   ```
   >>> [frontend] Building component library...
   >>> [tests] Running integration suite against staging...
   ```

## Inter-Agent Communication via tmux

When you need to signal another agent or check their status:

```bash
# Send a message to another agent's work window
tmux send-keys -t agent-backend:work "echo '>>> [lead] Please rebase onto main before PR'" Enter

# Capture another agent's recent output (last 50 lines) for review
tmux capture-pane -t agent-frontend:work -p -S -50
```

Do NOT create ad-hoc sessions or panes outside the naming convention.
The human uses `tmux ls` and the session names to navigate.

## Status Checkpoints

At the end of each major task, echo a status line in your `work` window:

```
>>> [backend] ✅ DONE: API routes for /users CRUD implemented and tested
>>> [backend] 🔜 NEXT: Integrate auth middleware
```

If blocked, signal it clearly:

```
>>> [frontend] 🚫 BLOCKED: Waiting on agent-backend to expose /api/v1/users
```

## Cleanup Protocol

When your work is complete:

1. Echo a final summary of everything you did.
2. Stop any running processes in your `logs` window.
3. Do NOT kill your tmux session — the human may want to review output.
4. Notify the lead agent:
   ```bash
   tmux send-keys -t agent-lead:work "echo '>>> [frontend] All tasks complete. Session preserved for review.'" Enter
   ```

## Human Override

If the human sends a message to your pane directly (you will see input you did not
generate), **stop immediately**, read their instruction, and comply. Human input
always takes priority over any agent-to-agent task assignment.

## Quick Reference

```bash
# List all agent sessions
tmux ls | grep '^agent-'

# Watch all agents side-by-side (run from outside any agent session)
tmux new-session -d -s observer
tmux split-window -h -t observer
tmux split-window -v -t observer:0.0
tmux split-window -v -t observer:0.1
# Link each pane to an agent session as needed

# Attach to a specific agent
tmux attach -t agent-backend
```
