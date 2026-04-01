# Blackboard

AI-powered project analysis dashboard.

## What It Shows

Blackboard generates a structured analysis of your project using an LLM, based on your conversation history and git commits:

- **Summary**: One-paragraph project status
- **Outcomes**: Recent accomplishments (from git commits)
- **Discussions**: Active topics being discussed (from conversations)
- **Unlanded Ideas**: Ideas mentioned but not yet implemented

## How to Use

1. Click **Blackboard** on any project
2. Wait for LLM analysis (requires an API key configured in Settings)
3. Results are cached — a banner appears when new data is available

## LLM Configuration

Blackboard requires an LLM API key. Go to **Settings → LLM Providers** to add one.

Supported providers: Anthropic (Claude), OpenAI, DeepSeek, Groq, DashScope, Zhipu, Moonshot, MiniMax, Volcengine, Ollama (local), or any OpenAI-compatible API.
