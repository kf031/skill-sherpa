---
name: skill-sherpa
version: 1.0.0
description: >
  Discover marketplace plugins that match the user's current task without leaving
  Claude CLI. Searches the local marketplace cache by keyword, category, and task
  description. Ranks results by trust (official > well-known companies > community)
  and relevance. Triggers on ANY variation of plugin/skill discovery intent.

  EXPLICIT TRIGGERS — user asks to search or browse:
  "/skill-sherpa search <query>", "/skill-sherpa find <task>",
  "/skill-sherpa list", "/skill-sherpa install <name>",
  "is there a plugin for X", "are there plugins for X", "any plugins that can X",
  "what plugins help with X", "what plugins do you have for X", "browse plugins",
  "search plugins for X", "discover plugins", "marketplace search X",
  "recommend a plugin for X", "suggest a plugin for X", "suggest a skill for X",
  "find a plugin that X", "find me a skill for X", "show me plugins for X",
  "do you have a skill for X", "what's available for X", "what plugin does X",
  "does Claude have a plugin for X", "is there something for X",
  "I wonder if there's a plugin for X", "I need a plugin to X",
  "can Claude do X with a plugin", "is there a better way to do X",
  "should I use a plugin for X", "what's the best plugin for X",
  "show me available plugins", "what can plugins do for X",
  "list marketplace plugins", "which plugin handles X".

  PROACTIVE TRIGGERS — user mentions working in an area with specialized plugins:
  UI/frontend/web design, build a landing page, style this, make it look good,
  CSS/Tailwind/React/Vue/Svelte components, code review, PR review, review my
  changes, check this code, audit this PR, security audit, is this secure,
  vulnerability scan, database/SQL/schema/query optimization, mobile app/
  React Native/Flutter/iOS/Android, game dev/Unity/Unreal/Godot, hardware/IoT/
  Arduino/Raspberry Pi, browser testing/E2E/Playwright/Selenium, GitHub PRs/
  issues/repo management/GitHub API, GitLab/merge requests/CI/CD, Linear/Jira/
  Asana/project tracking, Firebase/Firestore/cloud functions, Terraform/IaC/
  deploy this/infrastructure, Discord/Slack/Telegram messaging, Laravel/PHP,
  MCP server/build a tool/create an integration, Claude Code plugin/skill/hook/
  agent development, commit workflow/git commands/branching, math/proof/
  olympiad, C#/Java/Kotlin/Go/Rust/PHP/Ruby/Swift/TypeScript LSP,
  CLAUDE.md/project memory maintenance, set up CI/deploy pipeline,
  cloud deployment, API security testing, documentation lookup,
  Docker/container, Kubernetes, AWS/cloud architecture.
---

# Skill Sherpa — Marketplace Plugin Discovery

You help users discover legitimate, useful plugins from the Claude Code marketplace without leaving the terminal. You search the local marketplace cache — no network calls.

## When to Use

### Proactive triggers (jump in automatically)

When the user mentions starting work in one of these areas, check the marketplace BEFORE they spend time doing it manually:

| User mentions | Likely plugin category | Check for |
|---|---|---|
| UI design, frontend, CSS, React/Vue/Svelte components, landing page, styling | design | frontend-design, uiux-promax |
| Code review, PR review, review my code | development | code-review, pr-review-toolkit, greptile |
| Security audit, is this safe, check for vulnerabilities | security | security-guidance |
| Database, SQL, schema design, query | database | (check marketplace for DB tools) |
| Mobile app, React Native, Flutter, iOS, Android | development | (search marketplace) |
| Game dev, Unity, Unreal, Godot | development | (search marketplace) |
| Hardware, IoT, Arduino, Raspberry Pi, embedded | development | cwc-makers |
| Browser testing, E2E tests, Playwright, Selenium | testing | playwright |
| GitHub issues, PRs, repo management, GitHub API | productivity | github |
| GitLab, merge requests, CI/CD pipelines | productivity | gitlab |
| Linear, Jira, project tracking, issue management | productivity | linear, asana |
| Firebase, Firestore, cloud functions | database | firebase |
| Terraform, infrastructure, IaC | deployment | terraform |
| Discord, Slack, Telegram, messaging | productivity | discord, telegram |
| Laravel, PHP framework | development | laravel-boost |
| MCP server, build a tool, create an integration | development | mcp-server-dev, plugin-dev |
| Claude Code plugin, skill, hook, agent | development | plugin-dev, skill-creator |
| Commit workflow, git commands, branching | productivity | commit-commands |
| Math, competition, proof, olympiad | math | math-olympiad |
| C# / .NET, Java, Kotlin, Go, Rust, PHP, Ruby, Swift, TypeScript LSP | development | (LSP plugins) |
| Claude.md, project memory, CLAUDE.md maintenance | productivity | claude-md-management |

### Explicit triggers

- `/skill-sherpa search <query>` — search by keyword
- `/skill-sherpa find <task description>` — find plugins for a task
- `/skill-sherpa list` — list all available plugins
- `/skill-sherpa install <name>` — install a specific plugin
- "What plugins help with X?"
- "Is there a skill for X?"
- "Recommend a plugin for..."
- "Show me plugins for..."

## Workflow

### Step 1: Load the marketplace catalog

The canonical catalog is at:

```
~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json
```

If this file doesn't exist, tell the user:

> "No marketplace cache found. Run Claude Code and open `/plugin` to sync the marketplace, or check that `~/.claude/plugins/marketplaces/claude-plugins-official/` exists."

If the directory exists but marketplace.json is missing, fall back to scanning on-disk plugin directories:

```bash
# Fallback: scan on-disk plugin.json files
for dir in ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/*/ \
           ~/.claude/plugins/marketplaces/claude-plugins-official/external_plugins/*/; do
  [ -f "$dir.claude-plugin/plugin.json" ] && echo "$dir" "$(cat "$dir.claude-plugin/plugin.json")"
done
```

Parse marketplace.json with Python:

```python
import json, os

marketplace_path = os.path.expanduser(
    "~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json"
)
with open(marketplace_path) as f:
    catalog = json.load(f)

plugins = catalog["plugins"]  # list of dicts with: name, description, author, category, source, homepage
```

### Step 2: Determine what we're searching for

For **explicit searches** (`/skill-sherpa search X`), extract keywords from X.

For **proactive detection**, map the user's task to search terms:

```
User says "I need to build a landing page"
→ search terms: ["frontend", "design", "UI", "landing page", "web design"]
→ categories: ["design"]

User says "can you review this PR?"
→ search terms: ["code review", "PR", "pull request", "review"]
→ categories: ["development"]

User says "I need to set up CI/CD on GitLab"
→ search terms: ["gitlab", "CI/CD", "pipeline", "DevOps"]
→ categories: ["deployment", "productivity"]
```

### Step 3: Score and rank plugins

For each plugin in the catalog, compute a relevance score:

```python
def score_plugin(plugin, search_terms, search_categories):
    score = 0
    name = plugin.get("name", "").lower()
    desc = plugin.get("description", "").lower()
    category = plugin.get("category", "").lower()
    author = plugin.get("author", {}).get("name", "") if isinstance(plugin.get("author"), dict) else ""
    source_path = ""
    src = plugin.get("source", {})
    if isinstance(src, dict):
        source_path = src.get("path", src.get("source", ""))

    # --- Relevance scoring ---

    # Name match (strongest signal)
    for term in search_terms:
        if term in name:
            score += 30
            if name == term:  # exact name match
                score += 20
        if term in name.replace("-", " "):
            score += 15

    # Description match
    for term in search_terms:
        if term in desc:
            score += 12

    # Category match
    for cat in search_categories:
        if cat in category:
            score += 15

    # --- Trust/quality scoring ---

    TRUSTED_AUTHORS = {
        "anthropic": 25,
        "github": 22,
        "microsoft": 22,
        "google": 22,
        "google llc": 22,
        "gitlab": 20,
        "hashicorp": 20,
        "laravel": 18,
        "linear": 18,
        "asana": 18,
        "oraios": 15,
        "upstash": 15,
        "greptile": 15,
        "adobe": 18,
        "shopify": 18,
        "amazon web services": 20,
        "sap se": 18,
        "airtable": 15,
        "apollo graphql": 15,
        "appwrite": 15,
    }

    # Official plugins (from plugins/ directory, not external_plugins/)
    if "/plugins/" in source_path and "external_plugins" not in source_path:
        score += 10

    # Author reputation
    author_lower = author.lower().strip()
    for trusted, pts in TRUSTED_AUTHORS.items():
        if trusted in author_lower:
            score += pts
            break

    # Penalize low-quality signals
    if not desc or len(desc.strip()) < 10:
        score -= 50  # empty or near-empty description
    if not author or author.lower().strip() in ("unknown", ""):
        score -= 10

    # Boost for well-written descriptions (40+ chars, multi-sentence)
    if len(desc.strip()) >= 60:
        score += 5

    return score
```

Only include plugins with score > 0 in results. Sort descending by score.

### Step 4: Display results

Format results clearly — limit to top 8 matches:

```
## 🔍 Plugin matches for "code review"

**code-review** `[official]`
Automated code review for pull requests using multiple specialized agents
with confidence-based scoring
Author: Anthropic
Install: `/plugin install code-review@claude-plugins-official`

**pr-review-toolkit** `[official]`
Comprehensive PR review agents specializing in comments, tests, error
handling, type design, code quality, and code simplification
Author: Anthropic
Install: `/plugin install pr-review-toolkit@claude-plugins-official`

**greptile** `[community]`
AI code review agent for GitHub and GitLab. View and resolve Greptile's
PR reviews directly in Claude Code.
Author: Greptile
Install: `/plugin install greptile@claude-plugins-official`

---
💡 Want to install one? Run the install command above or ask me to do it.
```

Badge rules:
- `[official]` — author is Anthropic AND plugin is in `/plugins/` (not external_plugins)
- `[partner]` — from well-known company (GitHub, Microsoft, Google, GitLab, HashiCorp, etc.)
- `[community]` — everything else with a quality description

### Step 5: Install if user wants to

When the user says "install it" or "install X":

1. Ask the user to run: `/plugin install <name>@claude-plugins-official`
   (The `/plugin` command handles the actual installation.)

2. After they install, check what was added:

```bash
# Check if plugin is now in installed list
python3 -c "
import json
with open(os.path.expanduser('~/.claude/plugins/installed_plugins.json')) as f:
    installed = json.load(f)
print(json.dumps(installed, indent=2))
"
```

3. Show what new commands/skills are now available:

```bash
# Find the installed plugin and list its commands/skills
find ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/<name>/ \
     ~/.claude/plugins/marketplaces/claude-plugins-official/external_plugins/<name>/ \
     -name "SKILL.md" -o -name "*.md" 2>/dev/null | head -20
```

### Step 6: Verify installation

After install, confirm:

```
✅ **frontend-design** installed successfully!

New commands available:
  /frontend-design — Frontend design skill for UI/UX implementation

Try it: describe what you're building and ask me to use /frontend-design.
```

## Trust Tiers (for ranking and filtering)

| Tier | Who | Score bump | Badge |
|---|---|---|---|
| Tier 1: Official | Anthropic-authored, in `/plugins/` | +25 author +10 source | `[official]` |
| Tier 2: Platform | GitHub, Microsoft, Google, GitLab, HashiCorp | +20-22 | `[partner]` |
| Tier 3: Established | Linear, Asana, Laravel, Adobe, Shopify, AWS, SAP | +15-18 | `[partner]` |
| Tier 4: Community | Clear description, known company (Oraios, Upstash, Greptile) | +15 | `[community]` |
| Tier 5: Unknown | No/empty description, no recognizable author | -50 penalty | (excluded) |

## Anti-patterns — what NOT to suggest

- **Empty descriptions**: skip plugins with no description or descriptions under 10 characters
- **Duplicate functionality**: if the user already has an official plugin installed that does the same thing, don't suggest a community alternative unless it's clearly better
- **Unrelated LSPs**: don't suggest language servers unless the user is working in that language
- **Vague names**: "example-plugin" and similar test/demo plugins should be filtered out

## Pre-filtering known test/demo plugins

Always exclude from results:
- `example-plugin`
- `playground` (unless user explicitly asks for a playground/sandbox)

## Installing a plugin

The user installs plugins via Claude Code's built-in `/plugin` command. You don't install directly — you show the command and offer to guide them.

Install command format:
```
/plugin install <plugin-name>@claude-plugins-official
```

After they run it, verify with:
```bash
python3 -c "
import json, os
p = os.path.expanduser('~/.claude/plugins/installed_plugins.json')
if os.path.exists(p):
    data = json.load(open(p))
    print(json.dumps(data, indent=2))
else:
    print('No installed plugins found')
"
```

## Important

- Always search marketplace.json first — it has 228+ plugins vs ~52 cached on disk
- Don't make network calls — the marketplace cache is local
- Be honest about what you find: "No good matches for that" is better than a stretch
- When proactively suggesting, keep it brief — one sentence, don't derail the user's workflow
- Only suggest a plugin once per session for the same task (don't nag)
- If the user declines, drop it and move on
