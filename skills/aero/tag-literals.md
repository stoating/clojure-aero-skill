# Aero — Tag Literals Reference

All built-in tag literals provided by `aero.core`.

---

## #env

Read an environment variable. Returns `nil` if the variable is not set.

```clojure
{:db-url #env DATABASE_URL}
```

**Note:** Avoid storing passwords in env vars — they can leak via `ps`, logs, and monitoring tools. Use `#include` to bring in a private secrets file instead.

---

## #envf

Format a string with one or more environment variables (uses `clojure.core/format`).

```clojure
{:db-url #envf ["jdbc:postgresql://%s:%s/mydb" DB_HOST DB_PORT]}
```

Arguments: a vector where the first element is the format string and the rest are env var names.

---

## #prop

Read a Java system property (JVM only; returns `nil` in ClojureScript).

```clojure
{:home #prop "user.home"}
```

---

## #or

Try each value in order; return the first truthy one.

```clojure
{:port #or [#env PORT 8080]}
```

Works with any combination of tag literals and plain values. The last element is typically a literal default.

```clojure
{:level #or [#env LOG_LEVEL "info"]}
```

---

## #profile

Dispatch on the `:profile` key from opts. The value must be a map; Aero looks up the current profile, falling back to `:default`.

```clojure
{:server {:port #profile {:default 8000
                          :dev     8001
                          :test    8002
                          :prod    #or [#env PORT 80]}}}
```

```clojure
(read-config (io/resource "config.edn") {:profile :dev})
;; => {:server {:port 8001}}
```

Keys can be sets to match multiple profiles:

```clojure
{:debug? #profile {#{:dev :test} true
                   :default      false}}
```

---

## #hostname

Dispatch on the machine's hostname (from `HOSTNAME` env var on JVM, `os/hostname` on Node). The value must be a map.

```clojure
{:db-pool-size #hostname {"prod-1" 20
                          #{"prod-2" "prod-3"} 10
                          :default 2}}
```

Override in tests by passing `:hostname` in opts:

```clojure
(read-config "config.edn" {:hostname "prod-1"})
```

---

## #user

Dispatch on the current OS user (`USER` env var). Same map dispatch semantics as `#profile` and `#hostname`.

```clojure
{:data-dir #user {"alice" "/home/alice/data"
                  "bob"   "/home/bob/data"
                  :default "/tmp/data"}}
```

Override with `:user` in opts.

---

## #include

Include another EDN config file. The included file is read and resolved with the same opts (including profile and resolver).

```clojure
{:web    #include "web.edn"
 :db     #include "db.edn"
 :secrets #include #join [#env HOME "/.secrets.edn"]}
```

Resolution is controlled by `:resolver` in opts (default: `adaptive-resolver` — classpath first, then relative to parent).

---

## #ref

Reference another part of the config by path. The path is a vector usable with `get-in`.

```clojure
{:db-uri "datomic:dynamo://dynamodb"
 :server {:db #ref [:db-uri]}
 :worker {:db #ref [:db-uri]}}
```

`#ref` is resolved after the full map is read, so forward references work. Circular refs produce a warning and resolve to `nil`.

Nested paths:

```clojure
{:hosts {:primary "db.internal"}
 :replica-db {:host #ref [:hosts :primary]}}
```

---

## #join

Concatenate a sequence of values into a string.

```clojure
{:jdbc-url #join ["jdbc:postgresql://"
                  #env DB_HOST
                  ":"
                  #env DB_PORT
                  "/mydb"]}
```

---

## #merge

Deep-merge a sequence of maps.

```clojure
{:config #merge [{:a 1 :b 2}
                 {:b 3 :c 4}]}
;; => {:config {:a 1 :b 3 :c 4}}
```

Useful for composing config fragments, e.g. merging a base config with an override.

---

## #read-edn

Parse a string value as EDN. Useful when a value arrives as a string (e.g. from an env var) but needs to be structured data.

```clojure
{:features #read-edn #env FEATURE_FLAGS}
;; If FEATURE_FLAGS="#{:search :dark-mode}", result is the set #{:search :dark-mode}
```

---

## Type coercions

These tags parse a string value into a specific type. Typically combined with `#env` or `#or`.

### #long

```clojure
{:port #long #or [#env PORT 8080]}
```

### #double

```clojure
{:ratio #double #env RATIO}
```

### #keyword

```clojure
{:env #keyword #or [#env APP_ENV "production"]}
```

### #boolean

```clojure
{:debug? #boolean #or [#env DEBUG "false"]}
```

`"true"` (case-insensitive) → `true`; everything else → `false`.

---

## Chaining tag literals

Tag literals can be composed freely. Evaluation is inside-out (innermost first):

```clojure
{:port    #long #or [#env PORT 8080]
 :debug?  #boolean #or [#env DEBUG "false"]
 :mode    #keyword #or [#env APP_MODE "production"]}
```

`#profile` and `#or` are resolved lazily in a fixpoint loop, so they can reference values not yet known at the start of evaluation.
