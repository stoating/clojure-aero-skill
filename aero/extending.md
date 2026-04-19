# Aero — Extending with Custom Tag Literals

## Contents
- Stable API: extending `aero.core/reader`
- Alpha macro API: `aero.alpha.core`
- Case-like tag literals
- Custom conditional constructs
- Alpha API primitives reference

---

## Stable API: extending `aero.core/reader`

`reader` is a multimethod that dispatches on the tag symbol. Extend it to add a new tag literal.

```clojure
(require '[aero.core :refer [reader]])

(defmethod reader 'mytag
  [opts tag value]
  ;; opts  — the map passed to read-config (contains :profile, :resolver, etc.)
  ;; tag   — the tag symbol, e.g. 'mytag
  ;; value — the form following the tag in the EDN file
  (do-something-with value))
```

### Example: #decrypt

```clojure
(defmethod reader 'decrypt
  [opts tag value]
  ;; value is a base64-encoded ciphertext string
  (decrypt-string value (System/getenv "APP_SECRET_KEY")))
```

```clojure
;; config.edn
{:db-password #decrypt "U2FsdGVkX1+..."}
```

### Example: #slurp

```clojure
(defmethod reader 'slurp
  [opts tag value]
  (slurp value))
```

```clojure
;; config.edn
{:cert #slurp "/etc/ssl/my-cert.pem"}
```

**Limitation of the stable API:** Custom `reader` methods do not participate in Aero's fixpoint resolution loop. If your tag needs to compose with `#ref`, `#or`, `#profile`, or other tags that require deferred resolution, use the alpha macro API instead.

---

## Alpha macro API: `aero.alpha.core`

`aero.alpha.core` exposes a lower-level extension API that participates in Aero's resolution loop. It is marked alpha — the API may change, but has been stable in practice.

### When to use it

- Your tag is a conditional construct (like `#profile`, `#or`) that needs to suppress evaluation of non-selected branches.
- Your tag needs to compose with `#ref` (i.e. it may hold values that reference other config keys not yet resolved).
- You are building a library of custom tag literals for others to use.

### eval-tagged-literal multimethod

```clojure
(require '[aero.alpha.core :refer [eval-tagged-literal expand-case]])

(defmulti eval-tagged-literal
  (fn [tagged-literal opts env ks] (:tag tagged-literal)))
```

Your method receives:
- `tagged-literal` — a Clojure tagged literal record with `:tag` and `:form`
- `opts` — the map passed to `read-config`
- `env` — map of already-resolved `ks → value` pairs (the resolution environment)
- `ks` — vector of keys representing the current position in the config tree

Your method must return a map with:
- `::aero.core/value` — the resolved value, or a new tagged literal if resolution is incomplete
- `::aero.core/incomplete?` — `true` if the value could not yet be fully resolved
- `::aero.core/env` — updated env map (assoc your resolved key)

---

## Case-like tag literals

The most common pattern. Use `expand-case` to implement `#profile`-style dispatch:

```clojure
(require '[aero.alpha.core :refer [eval-tagged-literal expand-case]])

;; #myenv dispatches on a custom key in opts
(defmethod eval-tagged-literal 'myenv
  [tagged-literal opts env ks]
  (expand-case (:myenv opts) tagged-literal opts env ks))
```

```clojure
;; config.edn
{:log-level #myenv {:ci    :warn
                    :local :debug
                    :default :info}}
```

```clojure
(read-config "config.edn" {:myenv :ci})
;; => {:log-level :warn}
```

`expand-case` handles:
- Lookup of `case-value` in the map (with `:default` fallback)
- Set-valued keys (match any member of the set)
- Deferred resolution if the dispatch map itself contains unresolved tags

---

## Custom conditional constructs

For non-case-like conditionals (more like `#or`), implement `eval-tagged-literal` directly using the expansion primitives.

```clojure
(require '[aero.alpha.core :refer [eval-tagged-literal expand expand-scalar-repeatedly]])

;; #first-env — returns the value of the first env var in the list that is set
(defmethod eval-tagged-literal 'first-env
  [tl opts env ks]
  (let [{:keys [:aero.core/incomplete? :aero.core/value] :as expansion}
        (expand-scalar-repeatedly (:form tl) opts env ks)]
    (if incomplete?
      (update expansion :aero.core/value #(tagged-literal (:tag tl) %))
      (loop [[var-name & rest] value]
        (let [v (System/getenv (str var-name))]
          (cond
            v {:aero.core/value v
               :aero.core/env (assoc env ks v)}
            (seq rest) (recur rest)
            :else {:aero.core/value nil
                   :aero.core/env (assoc env ks nil)}))))))
```

---

## Alpha API primitives reference

All in `aero.alpha.core`:

| Var | Purpose |
|---|---|
| `eval-tagged-literal` | Multimethod — implement this for a new tag |
| `expand` | Expand any value (scalar or collection) |
| `expand-coll` | Expand a collection, resolving each element |
| `expand-scalar` | Expand a single scalar or tagged literal |
| `expand-scalar-repeatedly` | Expand a scalar until stable (handles chained tags like `#or #profile ...`) |
| `expand-case` | Dispatch on a case-value over a map (powers `#profile`, `#hostname`, `#user`) |
| `kv-seq` | Implementation detail — returns key-value pairs for reassembly |
| `reassemble` | Implementation detail — rebuilds a collection from processed key-value pairs |

### expand return shape

All `expand-*` functions return a map:

```clojure
{:aero.core/value      <resolved-value-or-tagged-literal>
 :aero.core/env        <updated-env-map>
 :aero.core/incomplete? <boolean>
 :aero.core/incomplete  <map describing what blocked resolution>}
```

When your `eval-tagged-literal` returns `:aero.core/incomplete? true`, Aero re-queues the tag and retries after other tags in the same pass have resolved. This is how `#ref` handles forward references.
