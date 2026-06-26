# Generating Idiomatic Clojure Code

How to structure and write Clojure that reads as if a seasoned Clojurist wrote it. Read this when creating new code, organizing a namespace, or making existing code idiomatic.

## Table of contents
1. Namespace structure
2. Naming conventions
3. Data-oriented design
4. Transducers (efficient pipelines)
5. Error handling
6. Validation with spec / Malli
7. REPL-driven scaffolding
8. Formatting conventions

---

## 1. Namespace structure

A typical namespace declaration. Order `:require` clauses and alias consistently; alias rather than `:refer` so call sites stay greppable.

```clojure
(ns myapp.orders
  "One-line purpose of this namespace."
  (:require
   [clojure.string :as str]
   [clojure.set :as set]
   [myapp.db :as db])
  (:import
   (java.time Instant)))
```

Conventions:
- Use `:as` aliases. Reserve `:refer` for a tiny, well-known set (e.g. `[clojure.test :refer [deftest is testing]]`).
- Avoid `(:refer-clojure :exclude ...)` unless you really must shadow a core name.
- Put pure helper functions near the top, public API lower, or group by feature — be consistent within the file.
- One namespace = one file, path mirrors the name: `myapp.orders` → `src/myapp/orders.clj`.

## 2. Naming conventions

| Thing | Convention | Example |
|---|---|---|
| functions / vars | `kebab-case` | `parse-order`, `total-price` |
| predicates | trailing `?` | `valid?`, `empty?`, `even?` |
| side-effecting / unsafe | trailing `!` | `swap!`, `reset!`, `delete-file!` |
| conversion "to" | `->` in the middle | `order->json`, `cents->dollars` |
| private | `defn-` or `^:private` | `(defn- helper …)` |
| dynamic vars | `*earmuffs*` | `*current-user*` |
| constants | plain kebab-case | `default-timeout` |
| record/protocol types | `PascalCase` | `Order`, `Storage` |

Don't abbreviate aggressively — `customer` beats `cust`. Clarity over keystrokes.

## 3. Data-oriented design

Model your domain with plain maps and keywords before reaching for records or classes. A "customer" is just a map until you have a concrete reason for more.

```clojure
;; Idiomatic: data is transparent, printable, and works with all core fns
(def order
  {:order/id 1001
   :order/customer "Ana"
   :order/items [{:sku "A1" :qty 2 :price 9.99}
                 {:sku "B2" :qty 1 :price 19.99}]})

(defn order-total [order]
  (->> (:order/items order)
       (map #(* (:qty %) (:price %)))
       (reduce + 0)))
```

Use **namespaced keywords** (`:order/id`, `:customer/email`) in real systems — they prevent collisions when maps are merged and document where a key belongs. They also pair naturally with spec.

Reach for `defrecord` only when you need (a) protocol dispatch performance, or (b) Java interop expecting a typed object. Even then, records still behave like maps.

## 4. Transducers (efficient pipelines)

When you chain `map`/`filter`/etc. over a large collection, each step builds an intermediate sequence. Transducers describe the *transformation* once, with no intermediates, and can be reused across contexts (sequences, channels, reductions).

```clojure
;; Lazy seq version — allocates an intermediate seq per step
(->> coll (filter even?) (map inc) (take 5))

;; Transducer — composed once, no intermediate collections
(def xform (comp (filter even?) (map inc) (take 5)))

(into [] xform coll)          ;; eager, into a vector
(sequence xform coll)         ;; lazy
(transduce xform + 0 coll)    ;; reduce with the same transform
```

Note: in a transducer, `comp` reads **left-to-right** (filter runs first), unlike ordinary function `comp`. Use transducers when the collection is large or the same pipeline feeds multiple sinks; for small/one-off cases plain `->>` is clearer.

## 5. Error handling

Prefer returning data over throwing when failure is expected. Throw (and let it propagate) for truly exceptional cases.

```clojure
;; Expected failure → return a result map the caller can branch on
(defn parse-int [s]
  (try
    {:ok (Integer/parseInt s)}
    (catch NumberFormatException _
      {:error :not-a-number :input s})))

;; ex-info carries structured data with the exception
(defn withdraw [balance amount]
  (when (> amount balance)
    (throw (ex-info "Insufficient funds"
                    {:type :insufficient-funds
                     :balance balance
                     :requested amount})))
  (- balance amount))

;; Read the data back out with ex-data
(try
  (withdraw 100 150)
  (catch clojure.lang.ExceptionInfo e
    (ex-data e)))  ;; => {:type :insufficient-funds, :balance 100, :requested 150}
```

Avoid bare `(catch Exception e)` that swallows everything — catch the specific type, or rethrow what you can't handle.

## 6. Validation with spec / Malli

For describing and checking the shape of data. `clojure.spec.alpha` ships with Clojure:

```clojure
(require '[clojure.spec.alpha :as s])

(s/def :order/id pos-int?)
(s/def :order/customer string?)
(s/def :order/total (s/and number? (complement neg?)))
(s/def :order/order (s/keys :req [:order/id :order/customer]
                            :opt [:order/total]))

(s/valid? :order/order {:order/id 1 :order/customer "Ana"})  ;; => true
(s/explain-str :order/order {:order/id -1 :order/customer "Ana"})
;; → human-readable description of what failed

;; Conform/destructure with specs
(s/conform (s/cat :op keyword? :args (s/* any?)) [:add 1 2 3])
;; => {:op :add, :args [1 2 3]}
```

Many teams prefer **Malli** for runtime, data-driven schemas (`[:map [:id pos-int?]]`). If the project already uses one, follow it; don't introduce a validation library unprompted.

## 7. REPL-driven scaffolding

Generate code that's easy to grow interactively. End data/logic-heavy namespaces with a `(comment …)` block (a "rich comment") holding example calls — it's ignored on load but evaluable form-by-form:

```clojure
(comment
  (def sample {:order/id 1 :order/items [{:qty 2 :price 5.0}]})
  (order-total sample)            ;; => 10.0
  (s/valid? :order/order sample)  ;; eval this to check the spec
  )
```

Keep functions small enough that each can be evaluated and inspected on its own. This is the single biggest difference between code that "works" and code that's pleasant to develop.

## 8. Formatting conventions

- **2-space indentation**; never tabs.
- Closing parens hug together on the last line — no "dangling" closing paren on its own line (`)` stacks like `}))`).
- Align function arguments under the first argument or indent by 2; let the user's editor (cljfmt / clojure-lsp) decide and just stay consistent with the file.
- Idiomatic `let` puts each binding pair on its own line:

```clojure
(let [total   (order-total order)
      taxed   (* total 1.21)
      rounded (Math/round taxed)]
  rounded)
```

- A blank line between top-level forms; a docstring on public functions.
- Don't add a comment that just restates the code. Comment the *why*, not the *what*.
