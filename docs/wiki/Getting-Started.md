# Getting Started

## Requirements

- macOS (Apple Silicon / M1+)
- One or more supported AI coding tools installed

## Installation

1. Download the latest DMG from [Releases](https://github.com/4over7/Memora/releases)
2. Open the DMG
3. Drag **Memora** to **Applications**
4. Launch Memora

## First Launch

### Hook Installation

On first launch, Memora detects installed AI tools and asks permission to install hooks:

- **Claude Code**: Adds shell command hooks to `~/.claude/settings.json`
- **Codex CLI**: Adds hooks to `~/.codex/hooks.json`
- **OpenCode**: Installs a TypeScript plugin

These hooks run in the background during your coding sessions, capturing conversation transcripts to `~/.memora/raw/`.

> You can skip hook installation and add it later, but Memora won't capture new conversations until hooks are installed.

### Interactive Guide

A 6-step guided tour introduces the main interface areas. You can replay it anytime by clicking the **?** button in the top-right corner.

### Data Location

All Memora data is stored locally at `~/.memora/`:

```
~/.memora/
├── events.jsonl    # Hook event log
├── raw/            # Original conversation transcripts
│   ├── claude-code/
│   ├── codex/
│   └── ...
├── memora.db       # SQLite index (rebuilt from raw/ if deleted)
└── .git/           # Auto-versioned backup
```

## Automatic Sync

Memora syncs every 15 seconds:
1. Reads new hook events
2. Imports conversation transcripts
3. Syncs git commits from all projects
4. Auto-commits data changes to `~/.memora/.git/`
