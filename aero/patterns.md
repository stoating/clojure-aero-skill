# Aero — Recommended Patterns

## Contents
- Config namespace wrapper
- Hiding secrets in private files
- Feature toggles
- Component / Integrant / Mount integration
- Schema validation with Malli or Plumatic Schema
- Single config file discipline
- Watching config for changes

---

## Config namespace wrapper

Always wrap `read-config` in a dedicated namespace. This insulates the rest of your code from the config structure, so if the EDN layout changes you only fix one place.

```clojure
(ns myapp.config
  (:require [aero.core :as aero]
            [clojure.java.io :as io]))

(defn config
  ([] (config :default))
  ([profile]
   (aero/read-config (io/resource "config.edn") {:profile profile})))

;; Accessors — thin wrappers that encode the config shape
(defn db-url     [cfg] (get-in cfg [:db :url]))
(defn server-port [cfg] (get-in cfg [:server :port]))
(defn feature?   [cfg feature] (contains? (get cfg :features #{}) feature))
```

Call `(config)` once at startup and pass the resulting map through the system — do not call `read-config` repeatedly at access time.

---

## Hiding secrets in private files

**Do not** put passwords in env vars or in config files committed to version control.

Instead, maintain a private file outside the repo (e.g. `~/.secrets.edn`) and include it:

```clojure
;; config.edn
{:secrets #include #join [#env HOME "/.secrets.edn"]

 :db {:url #profile
       {:prod #ref [:secrets :db-prod-url]
        :test #ref [:secrets :db-test-url]
        :default "jdbc:h2:mem:dev"}}}
```

```clojure
;; ~/.secrets.edn  (NOT in version control)
{:db-prod-url "jdbc:postgresql://prod-host/mydb?user=app&password=s3cret"
 :db-test-url "jdbc:postgresql://test-host/mydb?user=app&password=t3st"}
```

Add `~/.secrets.edn` to your global `.gitignore`.

---

## Feature toggles

Use `#profile` or a plain set in config:

```clojure
;; config.edn
{:features #profile {:default   #{:basic-search}
                     :dev       #{:basic-search :dark-mode :beta-ui}
                     :staging   #{:basic-search :dark-mode}
                     :prod      #{:basic-search}}}
```

```clojure
;; In code
(defn feature-enabled? [config feature]
  (contains? (:features config) feature))

(when (feature-enabled? config :dark-mode)
  (enable-dark-mode!))
```

---

## Component / Integrant / Mount integration

### With Integrant

Align your config map keys with your Integrant system keys so you can pass config directly to `ig/init`:

```clojure
;; config.edn
{:myapp/db     {:url #profile {:default "jdbc:h2:mem:dev"
                               :prod    #env DATABASE_URL}}
 :myapp/server {:port #long #or [#env PORT 8080]
                :db   #ref [:myapp/db :url]}}
```

```clojure
(ns myapp.system
  (:require [aero.core :as aero]
            [integrant.core :as ig]
            [clojure.java.io :as io]))

(defn system-config [profile]
  (aero/read-config (io/resource "config.edn") {:profile profile}))

(defn start! [profile]
  (ig/init (system-config profile)))
```

### With Stuart Sierra's Component

Use `merge-with merge` to avoid threading config through constructors:

```clojure
;; config.edn
{:database {:uri "datomic:mem://dev"}
 :listener {:port 8080}}
```

```clojure
(defn configure [system profile]
  (let [config (aero/read-config (io/resource "config.edn") {:profile profile})]
    (merge-with merge system config)))

(defn new-system [profile]
  (-> (new-system-map)           ; creates component instances with empty config
      (configure profile)         ; merges aero config into matching keys
      (component/system-using (dependency-map))))
```

---

## Schema validation with Malli

Validate after reading config so you get clear errors at startup:

```clojure
(ns myapp.config
  (:require [aero.core :as aero]
            [clojure.java.io :as io]
            [malli.core :as m]
            [malli.error :as me]))

(def Config
  [:map
   [:db [:map
         [:url :string]]]
   [:server [:map
             [:port [:int {:min 1 :max 65535}]]]]])

(defn config [profile]
  (let [cfg (aero/read-config (io/resource "config.edn") {:profile profile})]
    (when-let [errors (m/explain Config cfg)]
      (throw (ex-info "Invalid config" (me/humanize errors))))
    cfg))
```

### With Plumatic Schema

```clojure
(require '[schema.core :as s])

(def Config
  {:db     {:url s/Str}
   :server {:port s/Int}})

(defn config [profile]
  (s/validate Config
    (aero/read-config (io/resource "config.edn") {:profile profile})))
```

---

## Single config file discipline

Keep all configuration in one file (`resources/config.edn`). Use `#include` only to separate secrets, not to scatter config across many files. Benefits:

- Easier to grep and audit
- Can be emailed / attached for debugging
- Changes are visible in one diff

---

## Watching config for changes

Aero itself does not watch files. If you want live config reload in development, use a file-watch library (e.g. [juxt/dirwatch](https://github.com/juxt/dirwatch)) and call `read-config` again on change, then restart or reconfigure affected components.

```clojure
(require '[juxt.dirwatch :refer [watch-dir close-watcher]])

(watch-dir
  (fn [event]
    (when (= (:file event) (io/file "resources/config.edn"))
      (reset! app-config (config :dev))))
  (io/file "resources"))
```
