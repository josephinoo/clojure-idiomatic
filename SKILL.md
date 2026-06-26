---
name: clojure-idiomatic
description: Write, review, and refactor idiomatic Clojure. Use this skill whenever the user is working in Clojure or ClojureScript — generating code, designing functions, handling concurrency or parallelism, modeling data with immutable/persistent structures, or asking "how do I do X the Clojure way." Trigger on any mention of .clj/.cljs/.cljc files, deps.edn, Leiningen, the REPL, threading macros, atoms/refs/agents, core.async, transducers, multimethods, protocols, or "make this more idiomatic." Prefer this skill over generic functional-programming advice for anything Clojure-specific.
license: MIT
---

# Idiomatic Clojure

A skill for writing Clojure the way experienced Clojurists write it: data-first, immutable by default, built from small pure functions, and developed live at the REPL.

This file is the orchestrator. It holds the core philosophy, a decision workflow, and a quick reference of the idioms you reach for constantly. For depth on each of the four big topics, read the matching reference file **only when the task calls for it**:

| Topic | When to read | File |
|---|---|---|
| Generating idiomatic code | Writing new code, "make this idiomatic", structuring a namespace | `references/code-generation.md` |
| Parallelism & concurrency | `pmap`, `future`, atoms/refs/agents, core.async, reducers, threads | `references/parallelism.md` |
| Creating functions | `defn`, arities, destructuring, multimethods, protocols, recursion | `references/functions.md` |
| Immutability & change | persistent data structures, structural sharing, update patterns | `references/immutability.md` |

---

## Core philosophy

These five ideas drive almost every "is this idiomatic?" decision. Internalize them before writing code.

1. **Data over functions over macros.** Reach for plain maps, vectors, sets, and keywords first. Write a function only when behavior is needed. Write a macro only when a function genuinely cannot do the job (it needs to control evaluation or generate code). Most "I need a macro" instincts are wrong.

2. **Immutability is the default, not a feature you opt into.** Values never change. "Updating" a map returns a *new* map that shares structure with the old one. This is cheap (see `references/immutability.md`) and it eliminates whole categories of bugs, so don't fight it with mutable Java collections or `atom`-everywhere.

3. **Small pure functions, composed.** A function that takes data and returns data — no side effects — is easy to test, reuse, and reason about. Push side effects (I/O, state mutation) to the edges of the system and keep the core pure.

4. **The REPL is the development environment.** Clojurists don't write a file and run it; they evaluate forms interactively, growing the program piece by piece. Code should be written so it's easy to poke at: small top-level functions, data you can inspect, no hidden global state.

5. **Simplicity over ease.** Prefer the simple construct (a function, a map) over the clever or "powerful" one (a macro, a stateful object, deftype with mutable fields). Less machinery means less to go wrong. When two solutions work, ship the one a reviewer would call boring.

---

## Workflow for writing Clojure

When asked to write or change Clojure, follow this loop:

```
1. Model the data first.
   → What's the shape of the input? The output? Sketch the maps/vectors.
   → verify: can you write a literal example of the data by hand?

2. Write the transformation as pure functions on that data.
   → Compose core sequence functions before writing custom recursion.
   → verify: does it work on your literal example at the REPL?

3. Add state/IO only where genuinely needed, at the edges.
   → Choose the right reference type (atom/ref/agent) — see parallelism ref.
   → verify: is the core still pure and testable without the IO?

4. Refactor toward threading macros and core functions.
   → If a let-chain rebinds the same value, it probably wants -> or ->>.
   → verify: is it shorter and flatter without losing clarity?
```

**Before writing**, state assumptions about the data shape if it's ambiguous — guessing the shape of a map silently is the most common source of wrong Clojure. **Make surgical changes**: match the surrounding style (1-space vs 2-space alignment, `:require` ordering) rather than reformatting code you weren't asked to touch.

---

## Quick reference: idioms you use constantly

### Threading macros — flatten nested calls

Instead of reading inside-out, thread the value through steps left-to-right.

```clojure
;; ->   threads as the FIRST argument (good for maps/objects)
(-> {:a 1}
    (assoc :b 2)
    (update :a inc))
;; => {:a 2, :b 2}

;; ->>  threads as the LAST argument (good for sequences)
(->> (range 10)
     (filter even?)
     (map #(* % %))
     (reduce +))
;; => 120

;; some->  short-circuits on nil (no NullPointerException)
(some-> {:user {:name "Ana"}} :user :name)   ;; => "Ana"
(some-> {:user nil} :user :name)             ;; => nil

;; cond->  conditionally applies steps; value threads through the truthy ones
(cond-> {}
  true  (assoc :always 1)
  false (assoc :never 2))
;; => {:always 1}

;; as->  when the threaded position varies, bind it to a name
(as-> 5 $
  (+ $ 1)
  (* 2 $))
;; => 12
```

Rule of thumb: `->` for data/maps, `->>` for sequences/collections. If you find yourself mixing, `as->` or splitting the pipeline is cleaner than forcing it.

### Working with maps (the workhorse data structure)

```clojure
(assoc    {:a 1} :b 2)              ;; => {:a 1, :b 2}      add/replace key
(dissoc   {:a 1 :b 2} :a)           ;; => {:b 2}            remove key
(update   {:n 1} :n inc)            ;; => {:n 2}            apply fn to value
(get-in   {:a {:b 1}} [:a :b])      ;; => 1                nested read
(assoc-in {:a {:b 1}} [:a :c] 9)    ;; => {:a {:b 1 :c 9}}  nested write
(update-in {:a {:b 1}} [:a :b] + 10);; => {:a {:b 11}}      nested update
(merge    {:a 1} {:b 2} {:a 3})     ;; => {:a 3, :b 2}      right wins
(select-keys {:a 1 :b 2 :c 3} [:a :c]) ;; => {:a 1, :c 3}
```

Keywords are functions of maps, which is why `(:name person)` and `(map :name people)` work. Prefer `(:name person)` over `(get person :name)` for literal keyword lookups.

### Sequence processing — describe *what*, not *how*

```clojure
(map inc [1 2 3])                   ;; => (2 3 4)
(filter even? (range 10))           ;; => (0 2 4 6 8)
(remove even? (range 10))           ;; => (1 3 5 7 9)
(reduce + 0 [1 2 3 4])              ;; => 10
(reduce conj [] '(1 2 3))           ;; => [1 2 3]
(into {} [[:a 1] [:b 2]])           ;; => {:a 1, :b 2}
(group-by even? (range 6))          ;; => {true [0 2 4], false [1 3 5]}
(frequencies "hello")               ;; => {\h 1, \e 1, \l 2, \o 1}
(sort-by :age [{:age 30} {:age 20}]);; => ({:age 20} {:age 30})
(take 3 (iterate inc 0))            ;; => (0 1 2)   lazy & infinite-safe
```

Reach for these before writing a `loop`/`recur`. Hand-rolled recursion is a smell when a one-liner of `map`/`filter`/`reduce` exists. When you chain several of these over a big collection, consider **transducers** (`references/code-generation.md`) to avoid intermediate sequences.

### Destructuring — pull apart data in bindings

```clojure
;; map destructuring with defaults
(defn greet [{:keys [name greeting] :or {greeting "Hi"}}]
  (str greeting ", " name))
(greet {:name "Sam"})               ;; => "Hi, Sam"

;; sequential destructuring
(let [[first second & rest] [1 2 3 4 5]]
  {:first first :second second :rest rest})
;; => {:first 1, :second 2, :rest (3 4 5)}
```

Destructure in the parameter list rather than reaching into the map repeatedly inside the body. See `references/functions.md` for `:as`, nested destructuring, and namespaced `:keys`.

---

## Common anti-patterns to avoid

- **Mutable Java collections** (`java.util.ArrayList`, etc.) inside Clojure logic. Use persistent collections; if you need speed for a tight local build, use `transient`/`persistent!` (immutability ref).
- **`atom` as a global variable** to thread state through pure code. Pass values as arguments and return new values instead. State belongs at the edges.
- **Writing a macro** when a higher-order function would do. If your "macro" doesn't need to control evaluation, it's a function.
- **`loop`/`recur` for things `reduce` does.** Manual accumulators are usually a `reduce`.
- **Throwing on `nil`** instead of using `some->`, `fnil`, or `(get m k default)`.
- **`(if x true false)`** — `x` (or `(boolean x)`) is already the answer. Likewise `(when (not p) ...)` → `(when-not p ...)`.
- **Deeply nested `let` rebinding the same thing** — that's a threading macro waiting to happen.

---

## REPL-driven habits to encode

When generating Clojure, write it so the user can develop it interactively:

- Keep top-level functions small and independently callable.
- Include a **rich comment block** with example calls so the reader can evaluate them:

```clojure
(comment
  ;; evaluate these forms one at a time in the REPL
  (greet {:name "Sam"})
  (->> (range 10) (filter even?) (reduce +)))
```

- Prefer returning data you can inspect over side-effecting `println`.
- Don't bury logic in macros — it can't be inspected value-by-value at the REPL.

---

## Choosing the right reference file

- "Write me a function/module/parser that…" → start here, then **code-generation.md** for namespace structure, transducers, error handling, spec.
- "Make this run in parallel / faster across cores" or "how do I share state safely" → **parallelism.md**.
- "How do I express this as a function / polymorphism / recursion" → **functions.md**.
- "How does updating work without mutation / why is this fast / how do I model change" → **immutability.md**.

Read at most the files you need. Each is self-contained.
