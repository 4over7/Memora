<p align="center">
  <img src="icon-rounded.png" width="128" height="128" alt="Memora">
</p>

<h1 align="center">Memora 忆码</h1>

<p align="center">
  <strong>Own your AI coding memory. Continue anywhere.</strong><br>
  <sub>Capture, organize, and restore your AI coding conversations across every tool.</sub>
</p>

<p align="center">
  <a href="https://github.com/4over7/Memora/releases/latest">Download for macOS</a>
</p>

---

<p align="center">
  <img src="docs/screenshots/project-overview.png" width="800" alt="Project Overview">
</p>

## The Problem

Every day, developers have rich conversations with AI coding assistants — discussing architecture, debugging issues, making design decisions. But these conversations are **ephemeral**:

- Close the terminal? Context gone.
- Switch from Claude Code to Cursor? Start over.
- Come back after a week? Forgot everything.
- Compact kicked in? Details lost forever.

Your AI conversations are valuable **decision-making records**, not disposable chat logs.

## The Solution

Memora runs quietly in the background, capturing conversations from all your AI coding tools and organizing them by project. When you need to continue work — in any tool — Memora gives you back your full context.

### Supported Tools

| Tool | Capture Method | Status |
|------|---------------|--------|
| Claude Code | Hook (real-time) | Supported |
| Codex CLI | Hook (real-time) | Supported |
| OpenCode | Hook + SQLite | Supported |
| Cursor | SQLite polling | Supported |

## Key Features

### Continue V2 — Cross-Tool Workspace Creation

The core differentiator. Select conversations, git history, and project files from any combination of tools and sessions, then generate a new workspace with complete context:

- Git clone of your codebase
- `MEMORA_CONTEXT.md` with full conversation history (raw, uncompressed)
- Automatic `CLAUDE.md` / `AGENTS.md` references so AI tools read the context
- Works with **any** AI coding tool

<p align="center">
  <img src="docs/screenshots/continue-v2.png" width="700" alt="Continue V2 Wizard">
</p>

### Dual-Track Timeline

View conversations and git commits side by side in a synchronized timeline. See what was discussed and what was built, together.

<p align="center">
  <img src="docs/screenshots/dialogue.png" width="700" alt="Dual-Track Timeline">
</p>

### Blackboard

AI-powered project analysis: recent outcomes, active discussions, and unlanded ideas — generated from your conversation and git history.

### Six-Layer Knowledge Model

Organizes project knowledge into layers:

| Layer | Name | Example |
|-------|------|---------|
| L1 | Persona | User preferences, coding style |
| L2 | Context | CLAUDE.md, project rules |
| L3 | Strategy | Design docs, plans |
| L4 | Execution | Tasks, TODOs |
| L5 | Dialogue | Conversations (the process) |
| L6 | Outcome | Git commits (the results) |

### History Branching

Go back to any point in your project's history and create a new branch — with full conversation context from that moment, plus optional "hindsight" about what happened after.

## Installation

### Requirements

- macOS (Apple Silicon)
- One or more supported AI coding tools

### Download

Download the latest DMG from [Releases](https://github.com/4over7/Memora/releases).

1. Open the DMG
2. Drag **Memora** to **Applications**
3. Launch Memora — it will ask to install hooks into your AI tools
4. Start coding with any supported tool — Memora captures automatically

### First Launch

On first launch, Memora will:
1. Ask permission to install hooks into detected AI tools
2. Import existing conversation history
3. Show an interactive guide of the interface

## Architecture

```
Hook CLI → ~/.memora/events.jsonl + ~/.memora/raw/
                    ↓
              SQLite (index)
                    ↓
            Flutter Desktop UI
```

- **Frontend**: Flutter (macOS desktop)
- **Backend**: Rust (3 crates: memora-core, memora-ingest, memora-hook)
- **Bridge**: flutter_rust_bridge v2.12
- **Data**: All data stays local. Nothing is uploaded anywhere.

## Privacy

Memora is **100% local**. Your conversations, code, and data never leave your machine. There is no cloud sync, no telemetry, no analytics.

## Language

Memora supports English and Chinese (中文). Switch in Settings.

## Author

**Leon Xu (云梦泽)** — EastWorld ltd.

## License

Memora is proprietary software. All rights reserved.
