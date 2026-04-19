# Aero — Getting Started

## Contents
- Installation
- Reading config
- Sources: file path vs classpath
- Options map
- Resolvers
- Profiles
- Default opts

---

## Installation

### deps.edn
```clojure
{:deps {aero/aero {:mvn/version "1.1.6"}}}
```

### project.clj / Leiningen
```clojure
[aero "1.1.6"]
```

---

## Reading config

```clojure
(require '[aero.core :refer [read-config]])

;; Simplest form — file path string (works in REPL, unreliable in JARs)
(read-config "config.edn")

;; Classpath resource — works everywhere including uberjars
(read-config (clojure.java.io/resource "config.edn"))

;; With options
(read-config (clojure.java.io/resource "config.edn")
             {:profile :prod})
```

**Always use `clojure.java.io/resource` in production code.** A bare string path works relative to the current working directory — fine in REPL, breaks inside a JAR.

---

## Sources

`read-config` accepts any value that `clojure.java.io/reader` can open:

| Source type | Example |
|---|---|
| String file path | `"config.edn"` |
| `java.io.File` | `(io/file "config.edn")` |
| `java.net.URL` (classpath resource) | `(io/resource "config.edn")` |
| `java.io.InputStream` | any stream |

---

## Options map

```clojure
(read-config source {:profile  :dev        ; profile for #profile dispatch (default :default)
                     :user     "alice"      ; override for #user dispatch
                     :hostname "my-host"    ; override for #hostname dispatch
                     :resolver my-resolver  ; function or map for #include resolution
                     })
```

All keys are optional. The defaults are:

```clojure
{:profile  :default
 :resolver aero.core/adaptive-resolver}
```

---

## Resolvers

Resolvers control how `#include` finds the referenced file.

### Built-in resolvers

| Resolver | Behaviour |
|---|---|
| `adaptive-resolver` (default) | Tries classpath first, then relative to the including file |
| `relative-resolver` | Always resolves relative to the parent config file |
| `resource-resolver` | Always resolves from the classpath |
| `root-resolver` | Passes the include string through unchanged |

```clojure
(require '[aero.core :refer [read-config resource-resolver]])

(read-config "config.edn" {:resolver resource-resolver})
```

### Map resolver

Provide a map from include-string → source:

```clojure
(read-config "config.edn"
             {:resolver {"db.edn"  "resources/db/config.edn"
                         "web.edn" "resources/web/config.edn"}})
```

### Custom resolver function

Signature: `(fn [source include-string] -> resolvable-source)`

```clojure
(defn my-resolver [source include]
  (clojure.java.io/resource (str "config/" include)))

(read-config "config.edn" {:resolver my-resolver})
```

---

## Profiles

Profiles are arbitrary keywords passed as `:profile` in options and consumed by the `#profile` tag literal.

```clojure
;; config.edn
{:db {:url #profile {:default "jdbc:h2:mem:dev"
                     :test    "jdbc:h2:mem:test"
                     :prod    #env DATABASE_URL}}}
```

```clojure
;; In your code
(read-config (io/resource "config.edn") {:profile :prod})
```

There is no magic: `:profile` is just a key in the opts map. `#profile` does a map lookup using that key, falling back to `:default`.

---

## Default opts

```clojure
aero.core/default-opts
;; => {:profile :default, :resolver aero.core/adaptive-resolver}
```

You can inspect or extend this if you need a custom base set of options.
