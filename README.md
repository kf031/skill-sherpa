# Skill Sherpa

**You're leaving 10x on the table — and you don't even know it.**

Claude Code has 228+ plugins in the marketplace. Specialized ones for code review that catch bugs before they merge. Design tools that make UIs not look AI-generated. Security auditors that flag injection vulnerabilities in your diffs. Browser automation. Database tooling. CI/CD integration. Firebase management.

The problem: you're coding away, doing something the hard way, and you have no idea a plugin exists that would handle it in seconds. You shouldn't have to stop, open a browser, browse GitHub, read docs, and compare options just to discover what's already sitting on your disk.

**Skill Sherpa lives in your terminal and taps you on the shoulder.** It watches what you're working on and says "hey — there's a plugin for that. Want it?" No network calls. No context switching. Just the marketplace cache you already have, finally working for you.

## What it actually does

| When you... | Skill Sherpa... |
|---|---|
| Say "I need to build a landing page" | Suggests `frontend-design` [official] before you write a line of CSS |
| Ask "can you review this PR?" | Surfaces `code-review`, `pr-review-toolkit`, and `greptile` with trust badges |
| Mention "setting up Firebase auth" | Points you to `firebase` [partner] from Google |
| Type `/skill-sherpa search security` | Ranks 228 plugins by relevance + trust, shows top 8 with install commands |
| Say "is there a plugin for browser testing?" | Finds `playwright` [partner] from Microsoft, shows what it does |
| Wonder "what plugins exist for project management?" | Lists `linear`, `asana`, `github` with badges and one-line descriptions |

## Why trust the rankings?

Not all plugins are equal. Skill Sherpa scores across two dimensions:

**Relevance** — name match, description match, category match against what you're doing.

**Trust** — five tiers, weighted:
- `[official]` — Anthropic-authored, in the core plugins directory
- `[partner]` — GitHub, Microsoft, Google, GitLab, HashiCorp, AWS, Shopify, Adobe
- `[community]` — Established companies with well-written descriptions (Linear, Asana, Greptile, Upstash)
- **Fully excluded** — empty descriptions, test/demo plugins, unrecognizable authors

No hallucinated recommendations. No sketchy plugins slipping through. If nothing matches well, it tells you that instead of stretching.

## Install

Two ways:

### Plugin (recommended)

```bash
git clone https://github.com/kf031/skill-sherpa.git
cd skill-sherpa
claude plugins install .
```

### Quick skill copy

```bash
mkdir -p ~/.claude/skills/skill-sherpa
cp skills/skill-sherpa/SKILL.md ~/.claude/skills/skill-sherpa/skill.md
```

Restart Claude Code. That's it — no dependencies, no API keys, no config. It reads from the marketplace cache already on your disk.

## Usage

It works automatically — just start coding. Skill Sherpa detects when you're in plugin territory and suggests.

Or be explicit:
```
/skill-sherpa search security
/skill-sherpa find browser testing
/skill-sherpa list
/skill-sherpa install frontend-design
```

Or just ask naturally: "is there a plugin for X?", "what's the best plugin for X?", "any plugins that can X?", "recommend a plugin for X", "show me plugins for X".

## How it works

1. Reads the 228-plugin catalog from `~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json`
2. Falls back to scanning on-disk `plugin.json` files if the catalog is missing
3. Scores every plugin by keyword relevance and trust tier
4. Returns the top 8 matches with name, badge, description, author, and exact install command
5. After install, verifies it succeeded and shows new available commands

No network calls. Everything is local.

## Project structure

```
skill-sherpa/
├── .claude-plugin/
│   └── plugin.json          # Marketplace manifest
├── skills/
│   └── skill-sherpa/
│       └── SKILL.md         # 370-line skill definition
└── README.md
```

## License

MIT
