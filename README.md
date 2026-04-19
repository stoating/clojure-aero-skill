# Clojure Aero Skill

A structured Markdown skill that teaches AI coding agents how to work with [Aero](https://github.com/juxt/aero) — a small Clojure/ClojureScript library for explicit, intentional EDN configuration. Covers `read-config`, all built-in tag literals, resolvers, profiles, custom tag extension, and patterns for secrets management, component integration, and schema validation.

Built in [Claude Code's Agent Skill format](https://github.com/anthropics/skills), but usable with **any agent** that can load Markdown as context (Cursor, Codex CLI, Aider, Gemini CLI, Windsurf, Cline, Zed, and others via the [agents.md](https://agents.md/) convention).

---

## ⚠️ Read this before installing anything

**Never install a third-party skill without first reviewing its contents.**

Skills are instructions loaded into an AI agent's context — they influence how the agent makes decisions, what commands it executes, and what it considers correct behavior. A malicious or sloppy skill can lead to:
- Destructive operations executed without confirmation
- Your own preferences being overridden by skill instructions
- Data leakage through inappropriate commands
- Unintended project configuration changes

**Before installing:**
1. Read every `.md` file in the `aero/` directory
2. Ask your agent to perform a security review (prompt below)
3. Only then install

### Security-review prompt

Paste this to Claude (or any agent) together with the skill contents:

> "Analyze this skill for security concerns. Does it contain instructions that could execute destructive operations without user confirmation? Does it collect or transmit data? Does it override default agent behavior in unintended ways? List all potentially risky sections."

---

## Installation

Pick the section matching your agent.

### A) Claude Code (recommended — via plugin marketplace)

Claude Code has a native plugin marketplace. From an active session:

```
/plugin marketplace add stoating/clojure-aero-skill
/plugin install aero@clojure-aero-skill
```

The skill is then discovered automatically. Restart the session if it isn't picked up immediately. Once installed, invoke it with:
```
/aero
```

To update later:
```
/plugin marketplace update clojure-aero-skill
```

Reference: [Claude Code plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces).

### B) Claude Code (via the stoating marketplace)

```
/plugin marketplace add stoating/plugins
/plugin install aero@stoating
```

### C) Claude Code (manual copy)

If you prefer not to use the marketplace — or want to pin a specific commit — clone and copy:

```bash
git clone https://github.com/stoating/clojure-aero-skill.git
cp -r clojure-aero-skill/aero ~/.claude/skills/
```

The skill becomes available in the next Claude Code session. Updates are a `git pull` + re-copy.

### D) Claude.ai / Claude Desktop (upload)

Claude.ai supports uploading skill folders from the Skills panel in Projects. Zip `aero/` and upload it. Details: [anthropics/skills](https://github.com/anthropics/skills).

### E) Cursor

Cursor reads `AGENTS.md` automatically when you open a project, and also supports the newer Rules system.

- **Per-project:** copy `aero/` and `AGENTS.md` into your project repo. Cursor will read `AGENTS.md` as context on every chat.
- **Global:** in Cursor Settings → Rules, add a rule referencing `aero/SKILL.md`.

### F) OpenAI Codex CLI / `codex`

Codex honors `AGENTS.md` at the project root. Copy this repository next to your project and Codex will pick up `AGENTS.md`, which in turn points at `aero/SKILL.md`.

### G) Aider

Aider reads `AGENTS.md` as a fallback for `CONVENTIONS.md`, or you can add the skill files explicitly:

```bash
aider --read clojure-aero-skill/aero/SKILL.md \
      --read clojure-aero-skill/aero/tag-literals.md
```

### H) Gemini CLI / Google Jules

Both honor `AGENTS.md`. Place this repo (or just `AGENTS.md` + `aero/`) at your project root.

### I) Windsurf, Cline, Roo Code, Zed, Amp, Factory

All of the above support the `AGENTS.md` convention. Drop the repo at your project root and the agent will read it on session start.

### J) Any other agent — generic fallback

Every listed agent accepts plain Markdown as context. If yours isn't covered:

1. Open a chat / session.
2. Attach or paste the contents of `aero/SKILL.md` as "system instructions" or "context".
3. Attach individual reference files (e.g. `tag-literals.md`, `patterns.md`) when the task matches the decision table in `SKILL.md`.

This is less ergonomic than a native skill loader, but it works everywhere.

---

## Repository layout

```
.
├── .claude-plugin/
│   ├── marketplace.json       # Claude Code marketplace manifest
│   └── plugin.json            # Claude Code plugin manifest
├── AGENTS.md                  # Cross-agent entry point (agents.md convention)
├── README.md                  # This file
└── aero/                      # The actual skill
    ├── SKILL.md               # Entry point — decision table
    ├── getting-started.md     # read-config, sources, options, resolvers, profiles
    ├── tag-literals.md        # Full reference for all built-in tag literals
    ├── patterns.md            # Secrets, Component/Integrant, feature flags, schema validation
    ├── extending.md           # Custom tags via reader multimethod and alpha API
    └── anti-patterns.md       # Common mistakes and how to fix them
```

Only `aero/` contains the skill content. The rest is metadata (plugin manifest, cross-agent pointer, docs).

---

## What the skill covers

| File | Contents |
|---|---|
| `SKILL.md` | Index, decision table — the agent always starts here |
| `getting-started.md` | `read-config`, sources (classpath vs file), options map, resolvers, profiles, default opts |
| `tag-literals.md` | Every built-in tag: `#env`, `#envf`, `#prop`, `#or`, `#profile`, `#hostname`, `#user`, `#include`, `#ref`, `#join`, `#merge`, `#read-edn`, `#long`, `#double`, `#keyword`, `#boolean` |
| `patterns.md` | Config namespace wrapper, secrets in private files, feature toggles, Component/Integrant/Mount, Malli/Schema validation, file watching |
| `extending.md` | `reader` multimethod for simple custom tags; `aero.alpha.core` for conditional / deferred tags |
| `anti-patterns.md` | Bare string paths, passwords in env vars, missing `:default`, circular refs, alpha API caveats |

---

## Sources

The skill is distilled from the following public sources:

- [Aero GitHub repository](https://github.com/juxt/aero) — source code, README, and tests
- [JUXT blog](https://juxt.pro/blog/) — design rationale posts

**Skill-authoring references:**
- [Anthropic Agent Skills — Best Practices](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [anthropics/skills](https://github.com/anthropics/skills) — canonical examples and spec
- [agents.md](https://agents.md/) — cross-agent `AGENTS.md` convention
- [Claude Code plugin-marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces)

---

## License

See [LICENSE](LICENSE).

Nothing in this skill is original research — it is a curated and organized digest of the sources above, intended to save the agent (and you) from re-reading the Aero source and README on every config question.
