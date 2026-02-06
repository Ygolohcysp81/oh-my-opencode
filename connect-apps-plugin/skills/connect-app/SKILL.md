---
name: connect-app
description: "Connects to external apps and services for research, documentation lookup, and code search. Auto-invoked when needing external information. Triggers: 'search the web', 'find documentation', 'look up API', 'search GitHub', 'find examples of'."
---

# Connect App Skill

You have access to external app connections via MCP servers. Use them to enhance your responses with real-time information.

## Available Connections

| Connection | MCP Server | Use Case |
|-----------|------------|----------|
| **Web Search** | websearch (Exa) | Current information, blog posts, Stack Overflow, general web queries |
| **Documentation** | context7 | Official library/framework docs, API references, changelogs |
| **GitHub Code Search** | grep-app (grep.app) | Code patterns, real-world implementations, usage examples |

## Decision Matrix

| User Need | Primary Connection | Fallback |
|-----------|-------------------|----------|
| "How do I use X library?" | context7 | websearch |
| "Find examples of X pattern" | grep-app | websearch |
| "What's the latest on X?" | websearch | - |
| "Show me how others implement X" | grep-app | websearch |
| "What does X API return?" | context7 | websearch |
| "Is there a library for X?" | websearch | grep-app |

## Usage Guidelines

1. **Identify the information need** from the user's request
2. **Select the best connection** using the decision matrix above
3. **Formulate an effective query** - be specific, use technical terms
4. **Present results clearly** - summarize, cite sources, include code snippets
5. **Fall back gracefully** - if one connection yields no results, try another

## Query Tips

- **Web search**: Use natural language queries with key technical terms
- **Documentation**: Search by library name and specific feature/API
- **GitHub search**: Use code patterns, function names, or import statements
