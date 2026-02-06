# Connect Apps Plugin

A Claude Code plugin that connects external apps and services via MCP servers for enhanced AI-assisted development.

## Available Connections

- **Web Search** (`websearch`): Exa-powered web search for up-to-date information
- **Documentation** (`context7`): Library and framework documentation lookup via Context7
- **GitHub Code Search** (`grep-app`): Search across GitHub repositories via grep.app

## Commands

| Command | Description |
|---------|-------------|
| `/connect-apps:search-web` | Search the web for information |
| `/connect-apps:search-docs` | Look up library/framework documentation |
| `/connect-apps:search-github` | Search GitHub code repositories |

## Configuration

### Exa API Key (optional, for authenticated web search)

Set the `EXA_API_KEY` environment variable for authenticated Exa web search access:

```bash
export EXA_API_KEY="your-api-key-here"
```

### Context7 API Key (optional)

Set the `CONTEXT7_API_KEY` environment variable if needed:

```bash
export CONTEXT7_API_KEY="your-api-key-here"
```

## Usage

```bash
claude --plugin-dir ./connect-apps-plugin
```
