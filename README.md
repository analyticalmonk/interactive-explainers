# Interactive Explainers

Clear, interactive essays on topics in AI, science, and computing. Inspired by [distill.pub](https://distill.pub).

Pure HTML/CSS/vanilla JS - no dependencies, no build step, no bundler.

## Articles

- **[Embedding Codex: The App Server and the Two SDKs](codex-embedding/)** - How to embed OpenAI's Codex agent in your own software: the agent loop, the two controls that fence it in, the app-server JSON-RPC protocol, and why the TypeScript and Python SDKs are not the same thing underneath. 5 interactive figures.
- **[Stagehand: Inside the AI-Driven Browser Automation Framework](stagehand/)** - How Browserbase's Stagehand framework mixes deterministic code with AI-resolved instructions so browser automations survive redesigns. 5 interactive figures.
- **[Superpowers: The Anatomy of an Agent Skill](superpowers/)** - How the 200k-star Superpowers framework bootstraps itself into every session and what makes one agent skill stick where another gets ignored. 4 interactive figures.
- **[Artemis II: Why Going Back to the Moon Is a Big Deal](artemis-ii/)** - The first crewed lunar mission in 50+ years. What Artemis II actually did, how Orion works, and why this reshapes the next decade of human spaceflight. 5 interactive figures.
- **[World Models: How AI Learns to Simulate Reality](world-models/)** - Explore how Genie, JEPA, and World Labs are building AI that understands physical reality. 5 interactive figures.
- **[Pi & OpenClaw: The Self-Extending Agent](pi-agent/)** - How pi's minimal 4-tool agent architecture powers OpenClaw's multi-channel AI platform. 4 interactive figures.
- **[Autoresearch: AI That Does ML Research While You Sleep](autoresearch/)** - How Karpathy's autoresearch lets AI agents run autonomous ML experiments overnight. 3 interactive figures.
- **[DSPy: Programming - Not Prompting - Language Models](dspy/)** - How DSPy's signatures, modules, and optimizers replace hand-crafted prompts with compilable programs. 3 interactive figures.

## Development

Serve files with any static HTTP server:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000`.

## License

MIT
