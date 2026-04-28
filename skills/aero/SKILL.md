---
name: aero
description: Use when working with Aero — a Clojure/ClojureScript EDN configuration library. Activate when the user asks about read-config, tag literals (#env, #profile, #include, #ref, #or, #join, #merge, #envf, #hostname, #user, #long, #double, #keyword, #boolean, #prop, #read-edn), resolvers, profiles, the alpha macro API, or structuring config files in a Clojure project using Aero.
version: 1.0.0
---

# Aero

Aero is a small Clojure/ClojureScript library for explicit, intentional EDN configuration. It extends EDN with tag literals that handle environment variables, profiles, file inclusion, cross-references, and more — without resorting to Turing-complete configuration languages.

Coordinates: `[aero "1.1.6"]` / `{aero/aero {:mvn/version "1.1.6"}}`

## Quick Decision: What do you need?

| Task | Go to |
|------|-------|
| Load config, understand `read-config`, resolvers, profiles | [getting-started.md](getting-started.md) |
| Full reference for every built-in tag literal | [tag-literals.md](tag-literals.md) |
| Patterns: secrets, components, feature flags, schema validation | [patterns.md](patterns.md) |
| Define your own tag literals via `reader` multimethod or alpha macro API | [extending.md](extending.md) |
| Common mistakes and things to avoid | [anti-patterns.md](anti-patterns.md) |

## Core Mental Model

Aero reads an EDN file and resolves tag literals in a single pass (with a fixpoint loop to handle forward `#ref` references). The result is a plain Clojure map — no special types, no lazy resolution at call sites. Tag literals are resolved at read time, not at access time.

`read-config` accepts a source (file path, `java.io.File`, `java.net.URL`, classpath resource) and an optional options map:

```clojure
(require '[aero.core :refer [read-config]])

;; From classpath (recommended for production):
(read-config (clojure.java.io/resource "config.edn"))

;; With profile:
(read-config (clojure.java.io/resource "config.edn") {:profile :prod})
```

The default profile is `:default`. Every `#profile` map should include a `:default` key as a fallback.
