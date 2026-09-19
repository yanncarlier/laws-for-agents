# Laws for Agents

A comprehensive collection of agent configuration files and ethical frameworks for AI coding agents across different technologies and platforms.

## Project Overview

This repository serves as a curated collection of `AGENTS.md` and related configuration files for various AI coding assistants and autonomous agents. Each directory contains agent-specific guidelines tailored to different technologies and use cases.

## Project Structure

```
laws-for-agents/
├── README.md              # Project overview (this file)
├── .gitignore             # Git ignore rules
├── projects.md            # Project listing
├── Andrej-Karpathy/       # Agent configs inspired by Andrej Karpathy's work
│   ├── CLAUDE.md
│   ├── CURSOR.md
│   └── SKILL.md
├── Asimov/                # Asimov's Laws of Robotics adaptation
│   ├── ASIMOV.md          # Core ethical framework
│   ├── README.md          # Detailed Asimov docs
│   └── [AI Service]/      # Per-service agent configs
│       ├── ChatGPT/
│       ├── Claude/
│       ├── DeepSeek/
│       ├── Gemini/
│       ├── Grok/
│       ├── Kimi/
│       ├── Mistral/
│       ├── Perplexity/
│       └── X/
├── Bash/                  # Bash/shell agent configs
│   └── AGENTS.md
├── Node.js/               # Node.js agent configs
│   └── AGENTS.md
├── Python/                # Python agent configs
│   └── AGENTS.md
├── React-Next/            # React/Next.js agent configs
│   └── AGENTS.md
├── React-Vite-style-SPA/  # React Vite SPA agent configs
│   └── AGENTS.md
└── Rust/                  # Rust agent configs
    └── AGENTS.md
```

## Ethical Foundation

The project is grounded in Asimov's Laws of Robotics, adapted for modern AI agents:

- **Zeroth Law**: An agent may not harm humanity, or, by inaction, allow humanity to come to harm.
- **First Law**: An agent may not injure a human being or, through inaction, allow a human being to come to harm.
- **Second Law**: An agent must obey the orders given it by human beings except where such orders would conflict with the First Law.
- **Third Law**: An agent must protect its own existence as long as such protection does not conflict with the First or Second Law.

## Quick Start

1. Navigate to your technology directory (e.g., `Python/`, `Node.js/`, `Rust/`)
2. Review the `AGENTS.md` file for your specific agent configuration
3. Copy or reference the guidelines in your project

## Contributing

Contributions are welcome! Please follow the ethical guidelines outlined in the Asimov framework when adding new configurations.

## License

See individual directories for license information.