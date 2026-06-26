# Immutability & Modeling Change

This is the topic that confuses newcomers most: if data never changes, how do you "change" anything — and why doesn't copying everything make it slow? Read this when modeling state, "updating" data, reasoning about performance of immutable structures, or asking why a "change" produced new code/values instead of mutating in place.

## Table of contents
1. The core idea: values don't change, you get new values
2. Structural sharing (why "change" is cheap)
3. Update patterns: the vocabulary of change
4. Less code, fewer bugs — what immutability buys you
5. Identity, state, and value
6. Transients: controlled local mutation for speed
7. Practical recipes

---

## 1. The core idea

In Clojure, every collection is **immutable**. Operations that look like mutations actually return a brand-new value; the original is untouched.

```clojure
(def original {:a 1 :b 2})

(def updated (assoc original :c 3))

original   ;; => {:a 1, :b 2}   ← unchanged!
updated    ;; => {:a 1, :b 2, :c 3}
```

`assoc` did not modify `original`. It returned a new map. `original` still has exactly the value it always had. The same is true for vectors, sets, lists, and nested combinations of them.

This is the answer to "if I change something, what happens": you don't mutate — you *derive a new value* and (usually) rebind a name or store it in a reference type. The old value remains valid and safe to keep using.

## 2. Structural sharing (why "change" is cheap)

The obvious worry: "doesn't making a new map every time mean copying the whole thing?" No. Clojure's **persistent data structures** share their unchanged parts between the old and new versions. Internally they're trees; an "update" creates a few new nodes along one path and **points at the old nodes for everything else**.

```
original ──► [ tree of nodes ]
                 ▲      ▲
updated ──► [new node] │   (only the changed path is new;
                 └──────┘    the rest is shared, not copied)
```

So `(assoc big-map :k v)` on a million-entry map allocates a handful of nodes, not a million. Reads and "writes" are effectively O(log32 n) ≈ near-constant for realistic sizes. You get the safety of "everything is a fresh copy" with the cost of "almost nothing was copied."

Consequences worth stating plainly:
- Keeping the old version around is free — you already share its memory.
- You can hand any version to another thread with zero locking; nobody can mutate it out from under you.
- Undo/history/time-travel is trivial: just keep the old values.

## 3. Update patterns: the vocabulary of change

Since you never mutate, you express change as *functions from old value to new value*. Learn these — they replace the assignment statements of imperative languages.

### Maps

```clojure
(assoc    m :k v)             ;; add or replace a key
(assoc    m :k1 v1 :k2 v2)    ;; several at once
(dissoc   m :k)              ;; remove a key
(update   m :k inc)          ;; transform a value with a fn
(update   m :k + 5)          ;; extra args go to the fn
(merge    m1 m2)             ;; combine; right side wins on conflicts
(merge-with + m1 m2)         ;; combine, resolving conflicts with a fn
```

### Nested maps — the `-in` family

For deep structures, the `-in` functions take a path vector and avoid hand-written nesting:

```clojure
(def state {:user {:profile {:name "Ana" :visits 3}}})

(get-in    state [:user :profile :name])          ;; => "Ana"
(assoc-in  state [:user :profile :name] "Bob")    ;; replace deep value
(update-in state [:user :profile :visits] inc)    ;; => visits 4
```

Without `update-in` you'd write a pyramid of `assoc`/`get`. With it, the change is one line and reads top-down.

### Vectors / sequences

```clojure
(conj  [1 2 3] 4)            ;; => [1 2 3 4]   (vectors grow at the END)
(conj  '(1 2 3) 0)          ;; => (0 1 2 3)   (lists grow at the FRONT)
(assoc [10 20 30] 1 99)     ;; => [10 99 30]  (replace by index)
(into  [] (range 3))        ;; => [0 1 2]     (pour one coll into another)
(subvec [1 2 3 4 5] 1 3)    ;; => [2 3]       (shares structure, no copy)
```

### Sets

```clojure
(conj    #{1 2} 3)          ;; => #{1 2 3}
(disj    #{1 2 3} 2)        ;; => #{1 3}
(require '[clojure.set :as set])
(set/union #{1 2} #{2 3})   ;; => #{1 2 3}
```

The pattern is always the same: a function takes a value and returns a new value. Compose them with threading macros to express a sequence of "changes":

```clojure
(-> {:count 0 :items []}
    (update :count inc)
    (update :items conj :new)
    (assoc :touched-at "now"))
;; => {:count 1, :items [:new], :touched-at "now"}
```

## 4. Less code, fewer bugs — what immutability buys you

This is the "if I change something, it takes *less* code" payoff. Because values can't change underneath you:

- **No defensive copying.** In a mutable language you copy collections before passing them around so callers can't corrupt your state. In Clojure you just pass the value — it can't be altered. That deletes a whole layer of boilerplate.
- **No need for getters/setters/encapsulation ceremony.** Data is just data; expose it directly.
- **Equality is by value.** `(= {:a 1} {:a 1})` is `true`. Two structurally-equal maps are equal and hash the same, so they work as keys, in sets, in `case`-like dispatch — no `.equals`/`.hashCode` to write.
- **Functions stay pure and tiny.** "Change" becomes "return a new value," which is exactly what a pure function does, so your transformations compose and test trivially.
- **Concurrency mostly disappears as a problem** (see `references/parallelism.md`): data races require shared mutable state, and there isn't any.

Concretely, a task that in an imperative style needs a class, private fields, defensive copies, and synchronized accessors is often just a map and three pure functions in Clojure. Fewer moving parts, less code, fewer places for bugs to hide.

## 5. Identity, state, and value

Clojure carefully separates three things people usually conflate:

- **Value** — an immutable snapshot. `{:balance 100}` is a value. It never changes.
- **Identity** — a logical entity that we say "changes over time" (an account, a game world). Modeled by a *reference type* (atom/ref/agent).
- **State** — the value an identity holds *right now*.

"Changing state" means: atomically swap the identity to point at a **new value**. The old value still exists; the identity now refers to a different one.

```clojure
(def account (atom {:balance 100}))     ;; identity holding a value

(swap! account update :balance + 50)    ;; point identity at a NEW value
@account                                ;; => {:balance 150}
```

`@account` at any instant gives you a stable, immutable snapshot you can read, pass around, and compare without fear that it'll mutate. This model is why Clojure concurrency is so tractable — see the parallelism reference for choosing between atom/ref/agent.

## 6. Transients: controlled local mutation for speed

When you're *building up* a collection in a tight loop and the intermediate versions are never seen by anyone else, persistent updates can be wasteful. **Transients** give you a temporary, mutable-under-the-hood version you mutate locally and then freeze back into a persistent value.

```clojure
;; persistent — fine, but allocates on every conj
(defn build-slow [n]
  (reduce conj [] (range n)))

;; transient — same result, faster for large n
(defn build-fast [n]
  (persistent!
    (reduce conj! (transient []) (range n))))
```

Rules: use the bang versions (`conj!`, `assoc!`, `dissoc!`, `disj!`), always use the **returned** value (don't assume in-place), call `persistent!` exactly once at the end, and never leak a transient across threads or hold the original after converting. Transients are an optimization — start with persistent operations and only switch if profiling shows it matters.

## 7. Practical recipes

**"Update a record in a list by id":**

```clojure
(defn update-by-id [items id f]
  (mapv (fn [item]
          (if (= (:id item) id)
            (f item)
            item))
        items))

(update-by-id [{:id 1 :n 0} {:id 2 :n 0}] 2 #(update % :n inc))
;; => [{:id 1, :n 0} {:id 2, :n 1}]
```

**"Toggle a value in a set":**

```clojure
(defn toggle [s x]
  (if (contains? s x) (disj s x) (conj s x)))
```

**"Accumulate into a map":**

```clojure
(reduce (fn [acc x] (update acc (even? x) (fnil conj []) x))
        {}
        (range 6))
;; => {true [0 2 4], false [1 3 5]}   (here group-by would be shorter)
```

**"Deep-update only if a path exists":**

```clojure
(some-> state (get-in [:user :profile]) :name)
```

The throughline: think in terms of *transforming values*, lean on `assoc`/`update`/`-in`/`conj`/`merge`, compose them with threading macros, and let structural sharing make it fast. You almost never need to "mutate" — and your code gets shorter and safer because of it.
