# History Branching

Go back to any point in your project's history and create a new branch — with full context from that moment.

## Use Cases

- **"I should have taken a different approach 3 days ago"** — Branch from that commit, with the conversation context from that time
- **"Let me try Vue instead of React"** — Fork from before the React decision
- **Code review discovered a wrong turn** — Go back to before the wrong turn with full hindsight

## How It Works

1. In the project view, find a git commit in the **OUTCOME** section
2. Right-click → **Continue from here**
3. Fill in:
   - **Reason**: Why you're going back (optional, written into the branch context)
   - **Mode**: Worktree (lightweight, linked) or Clone (full copy)
   - **Branch name**: Name for the new git branch
   - **Target directory**: Where to create it
   - **Include hindsight**: Show what commits happened *after* this point

## Modes

### Worktree
- Lightweight: Creates a linked directory inside `.branches/`
- Shares git history with the original project
- Easy to merge changes back
- Cannot delete the original project while worktree exists

### Clone
- Full copy: Independent project at any location
- Larger disk usage
- Safe to delete the original
- No merge path back (manual cherry-pick needed)

## Context Written

Memora writes a `CLAUDE.md` in the new branch containing:
- **Why this branch was created** — your reason
- **State at branch point** — conversation + git context from that moment
- **Hindsight** (optional) — what commits happened after, so you know what to avoid or reconsider
