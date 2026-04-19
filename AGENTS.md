# Agent instructions

This repository contains the **`aero`** skill — a structured set of Markdown files that teach an AI coding agent how to work with Aero, the Clojure/ClojureScript EDN configuration library by JUXT.

**Any agent that understands this `AGENTS.md` convention should:**

1. Treat `aero/SKILL.md` as the entry point — it contains a decision table pointing to the right reference file for the task.
2. Load reference files on demand based on that table:
   - `getting-started.md` — `read-config`, sources, options map, resolvers, profiles, default opts
   - `tag-literals.md` — full reference for every built-in tag: `#env`, `#envf`, `#prop`, `#or`, `#profile`, `#hostname`, `#user`, `#include`, `#ref`, `#join`, `#merge`, `#read-edn`, `#long`, `#double`, `#keyword`, `#boolean`
   - `patterns.md` — config namespace wrapper, secrets, feature toggles, Component/Integrant integration, Malli/Schema validation
   - `extending.md` — custom tag literals via `reader` multimethod and `aero.alpha.core` macro API
   - `anti-patterns.md` — common mistakes and how to fix them
3. Always recommend `(clojure.java.io/resource "config.edn")` over bare string paths in production code.
4. Always recommend a `:default` key in every `#profile` map.
5. Never suggest storing passwords in environment variables — recommend `#include` pointing at a private file outside version control instead.

This file follows the [agents.md](https://agents.md/) convention and is honored by OpenAI Codex CLI, Cursor, Aider, Zed, Amp, Gemini CLI, Google Jules, Windsurf, Factory, RooCode, and many others.

For Claude Code, the richer native format is `.claude-plugin/` + `aero/SKILL.md`.
