# Aero — Anti-Patterns

Common mistakes when using Aero, and how to avoid them.

---

## Using a bare string path in production

**Wrong:**
```clojure
(read-config "config.edn")
```

**Why it breaks:** Resolves relative to the JVM working directory, which is fine in the REPL but undefined inside a JAR or uberjar.

**Fix:**
```clojure
(read-config (clojure.java.io/resource "config.edn"))
```

---

## Calling read-config on every access

**Wrong:**
```clojure
(defn get-db-url []
  (get-in (read-config (io/resource "config.edn")) [:db :url]))
```

**Why it's bad:** Re-reads and re-parses the file on every call. Can cause subtle inconsistencies if config or env vars change mid-run.

**Fix:** Read once at startup, pass the resulting map through your system.

```clojure
(defonce config (read-config (io/resource "config.edn") {:profile (env-profile)}))
```

---

## No :default in #profile maps

**Wrong:**
```clojure
{:port #profile {:dev 8001 :prod 80}}
```

**Why it breaks:** If `read-config` is called with any profile other than `:dev` or `:prod` (including the default `:default`), the lookup returns `nil` silently.

**Fix:** Always include `:default`:

```clojure
{:port #profile {:dev 8001 :prod 80 :default 8000}}
```

---

## Storing passwords in environment variables

**Wrong:**
```clojure
{:db-password #env DB_PASSWORD}
```

**Why it's a risk:** Process environment is readable via `ps e -f`, leaks into logs and monitoring tools, and is hard to rotate.

**Fix:** Use `#include` pointing at a private file outside version control:

```clojure
{:secrets #include #join [#env HOME "/.secrets.edn"]}
```

---

## Using #ref with a path that doesn't exist

**Wrong:**
```clojure
{:db #ref [:databsae :url]}   ;; typo in key
```

**Why it breaks:** Aero will emit a warning `WARNING: Unable to resolve ...` and substitute `nil`. The config looks valid but the value is missing.

**Fix:** Verify paths match the actual EDN structure. Use a schema validator (Malli, Spec, or Plumatic Schema) on the result of `read-config` to catch missing keys early.

---

## Circular #ref chains

**Wrong:**
```clojure
{:a #ref [:b]
 :b #ref [:a]}
```

**Why it breaks:** Aero detects the cycle, prints `WARNING: Unable to resolve "#ref [:b]" at [:a]` to stderr, and resolves both keys to `nil`. The config loads "successfully" but the values are silently missing. (The `Max attempts exhausted` exception fires for other unresolvable expansions, not for `#ref` cycles.)

**Fix:** Extract the shared value to a non-ref key:

```clojure
{:base-url "https://api.example.com"
 :a {:url #ref [:base-url]}
 :b {:url #ref [:base-url]}}
```

---

## Expecting #include to work inside JARs with relative-resolver

**Wrong:**
```clojure
(read-config (io/resource "config.edn")
             {:resolver aero.core/relative-resolver})
```

**Why it breaks:** `relative-resolver` tries to compute a path relative to the source file, which doesn't work for classpath URLs inside JARs.

**Fix:** Use `resource-resolver` when reading from the classpath:

```clojure
(read-config (io/resource "config.edn")
             {:resolver aero.core/resource-resolver})
```

Or rely on the default `adaptive-resolver`, which handles both cases.

---

## Overriding reader multimethod without requiring aero.core first

**Wrong:**
```clojure
(ns myapp.readers)
(defmethod aero.core/reader 'myns/mytag ...)  ;; aero.core never required
```

**Why it breaks:** The `defmethod` runs but `aero.core/reader` may not be initialized yet in certain load orders. The method can disappear.

**Fix:** Always `(:require [aero.core])` in the namespace that defines the method.

---

## Relying on alpha API stability

`aero.alpha.core` is explicitly marked alpha. The API is stable in practice, but breaking changes are not ruled out across major versions.

**Fix:** If you publish a library built on `aero.alpha.core`, document the Aero version range you support and test against each minor version.

---

## Scattering config across many #include files

Using many `#include` files for non-secret config makes the config hard to audit and debug.

**Fix:** Keep all non-secret configuration in a single `config.edn`. Use `#include` only for secrets or large environment-specific overrides that are legitimately managed separately.
