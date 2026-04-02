# Changelog

All notable changes to Memora are documented here.

## [0.1.7] — 2026-04-02

### Fixed (Independent Review)
- **F1**: commit-link processing moved after git sync to prevent silent association loss; design doc synced to actual implementation
- **F3**: Drift verdict cancellation now persists (DELETE from DB instead of UI-only clear)
- **F5**: flutter analyze 0 issues, flutter test 52/52 pass (fixed Ollama URL + custom_anthropic assertions)
- **F6**: Settings "Reinstall Hooks" button for retry after skip/failure
- **F7**: Branch filter shows empty state instead of silently falling back to all sessions

### Improved
- **F2**: Git auto-backup commands log failures to stderr via run_git() helper
- **F4**: Drift reads strategy files full text (up to 5000 chars); continue context expanded to 50 commits + 10 sessions
- Blackboard view toggle: Insights | vs Plan (replaces separate Drift card)
- Prev/next commit navigation buttons (replaces inaccurate scrollbar markers)
- Search: keyword highlighting, snippet around match, result count, clear project selection
- Session timeline: right-click commits for Recall/Branch
- Branch dialog: Browse button for target directory in Clone mode
- Responsive Blackboard cards (horizontal >700px, vertical otherwise)
- Titlebar: fullSizeContentView eliminates gap between traffic lights and content

## [0.1.5] — 2026-04-01

### Added
- **Continue V2**: Cross-tool workspace creation — git clone + MEMORA_CONTEXT.md with full conversation history, git log, and project files
- **Drift Awareness**: Plan vs actual work comparison panel in Blackboard (3+1 layout), with user verdict buttons (OK / Circle back / Adopt)
- **Context-Linked Commits**: Hook detects commits during AI sessions and links them to conversation sessions; linked commits show 💬 icon
- **Onboarding tour**: 6-step interactive guide with spotlight overlay, replayable via ? button
- **i18n**: English + Chinese language support, switchable in Settings
- **Scrollbar commit markers**: Orange minimap on scrollbar showing git commit positions
- **Auto git protection**: ~/.memora/ data auto-committed on every sync (DB excluded)
- **Pretty DMG installer**: Dark background, drag-to-Applications layout via create-dmg
- **GitHub Release**: Public repo at github.com/4over7/Memora with wiki documentation

### Fixed
- Scrollbar markers now use estimated pixel heights instead of item index ratio
- Scrollbar thumb thickens on hover (6px → 10px) for easier mouse grabbing
- Version display shows full git hash (e.g. 0.1.5+998237f)
- Target directory in Clone mode now has Browse button for folder picker
- Recall/Branch context menu items have subtitle descriptions
- Excluded *.db from ~/.memora/ git protection (was causing 73GB disk bloat)

### Design Documents
- `docs/drift-awareness-design.md` — Three-layer drift detection design
- `docs/context-linked-commits-design.md` — Commit ↔ session binding via trailer
- `docs/vision-version-control-for-ai-era.md` — Vision: Git's consciousness layer

## [0.1.2] — 2026-03-31

### Added
- **Dual-track timeline**: Conversations and git commits side by side with draggable split ratio
- **Blackboard**: AI-powered project analysis (outcomes, discussions, unlanded ideas)
- **History Branching**: Worktree and Clone modes from any historical commit
- **Message-level Continue**: Right-click to export conversation as Markdown
- **Six-layer knowledge model**: L1 Persona through L6 Outcome
- **Multi-provider LLM**: 12 providers + Ollama + 2 custom modes
- **Full-text search**: Across conversations and git commits (FTS5)
- **SpeakOut import/export**: Bulk API key management
- **Sidebar collapse**: Collapsible project list
- **Manual project add**: Add projects not auto-detected
- **Brand colors**: Tool-specific colors (Claude orange, Codex purple, etc.)
- **Settings**: LLM providers, Persona configs, About page
- **52 Flutter tests** + Rust integration tests
- **Code signing**: Developer ID Application certificate
- **DMG packaging**: Automated build.sh with version auto-increment

### Supported Tools
- Claude Code (hook, real-time)
- Codex CLI (hook, real-time)
- OpenCode (hook + SQLite)
- Cursor (SQLite polling)

## [0.1.0] — 2026-03-29

### Added
- Initial release
- Flutter desktop (macOS) + Rust backend via flutter_rust_bridge
- Three Rust crates: memora-core, memora-ingest, memora-hook
- Hook-driven conversation capture
- SQLite storage with events.jsonl pipeline
- Basic project view with session list
