# agent-skills

A collection of agent skills for [Claude Code](https://claude.ai/code) marketplace.

## Installation

**CLI:**
```bash
claude plugin install masaki39/agent-skills
```

**Claude Code UI:**
Type `/plugin` in the chat, search for `masaki39/agent-skills`, and install.

## Skills

| Name | Description |
|------|-------------|
| pdf-extract | Extract figures and charts from PDF using pdfimages CLI |
| simple-citations-setup | Setup the simple-citations Obsidian plugin with Zotero Better BibTeX integration |

## Structure

```
agent-skills/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   ├── pdf-extract/
│   │   └── SKILL.md
│   └── simple-citations-setup/
│       └── SKILL.md
└── README.md
```

## License

MIT
