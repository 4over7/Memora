# Supported Tools

Memora captures conversations from multiple AI coding tools through different mechanisms.

## Claude Code

| | |
|---|---|
| **Capture** | Hook (real-time) |
| **Hook location** | `~/.claude/settings.json` |
| **Data format** | JSONL transcript |
| **Events** | User prompt, AI response, tool use, session end, pre-compact |

Claude Code hooks fire on every conversation turn, giving Memora real-time access to the full transcript.

## Codex CLI

| | |
|---|---|
| **Capture** | Hook (real-time) |
| **Hook location** | `~/.codex/hooks.json` |
| **Data format** | JSONL transcript |

Similar to Claude Code, Codex fires hooks on conversation events.

## OpenCode

| | |
|---|---|
| **Capture** | Hook + SQLite |
| **Hook location** | TypeScript plugin |
| **Data format** | SQLite database |

OpenCode stores sessions in a SQLite database. Memora reads directly from it and also installs a TypeScript plugin for real-time event capture.

## Cursor

| | |
|---|---|
| **Capture** | SQLite polling |
| **Data format** | `state.vscdb` |

Cursor does not support hooks. Memora reads from Cursor's internal SQLite database (`state.vscdb`) during each sync cycle.

> Cursor project attribution uses a multi-level matching algorithm (workspaceId → conversationId → contextProviders) since Cursor doesn't store project paths directly.
