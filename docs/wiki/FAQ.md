# FAQ

## General

### Is my data uploaded anywhere?
No. Memora is 100% local. All data stays on your machine at `~/.memora/`. There is no cloud sync, telemetry, or analytics.

### Does Memora require an internet connection?
Only for the Blackboard feature (which calls an LLM API). All other features work completely offline.

### What happens if I delete `~/.memora/`?
Memora will recreate the directory on next launch and re-import available history from your AI tools. However, hook event logs and some conversation history may be lost if the tools have already cleaned up their own data.

### Can I use Memora with tools not listed as supported?
Not yet. Support for additional tools (Windsurf, Aider, etc.) is planned.

## Troubleshooting

### Memora shows "Waiting for first dialogue to be captured"
- Check that hooks are installed: look for the hook entries in your tool's config
- Try having a conversation with your AI tool, then wait 15 seconds for auto-sync
- Click the refresh button in Memora's title bar

### macOS says "Memora can't be opened because it is from an unidentified developer"
Right-click the app → Open → Open. This only needs to be done once. We're working on Apple notarization to eliminate this warning.

### The app is using too much disk space
Check `~/.memora/.git/` size. If it's unexpectedly large, run:
```bash
rm -rf ~/.memora/.git
```
Memora will recreate a clean git repository on next sync.

### Blackboard shows "Configure LLM API key in Settings first"
Go to Settings → LLM Providers → Add, and configure at least one API key.

## Data & Privacy

### Can I back up my Memora data?
Yes. Copy `~/.memora/` to your backup location. The `raw/` directory contains the original conversation transcripts (most valuable). `memora.db` is a cache that can be rebuilt.

### Can I move `~/.memora/` to a different location?
Not currently. The data directory is hardcoded to `~/.memora/`.

### Does Memora modify my project files?
Only when you explicitly use Continue V2 (creates a new directory) or History Branching (creates a new branch). Memora never modifies your existing project code.
