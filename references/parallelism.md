# Parallelism & Concurrency in Clojure

Clojure has one of the richest concurrency toolkits of any language, precisely *because* its data is immutable — values can be shared across threads with zero locking. Read this when the user wants to use multiple cores, run work asynchronously, or coordinate shared state safely.

## Table of contents
1. Parallelism vs. concurrency (pick the right tool)
2. Easy data parallelism: `pmap`, `pvalues`, `pcalls`
3. `future`, `promise`, `delay`
4. Managing shared state: atoms, refs (STM), agents
5. Reducers & `fold` (parallel reduction)
6. core.async (channels & CSP)
7. Thread pools & raw threads
8. Decision guide

---

## 1. Parallelism vs. concurrency

- **Parallelism** = doing many computations *at the same time* to go faster (multiple CPU cores). Tools: `pmap`, reducers/`fold`, futures over a thread pool.
- **Concurrency** = structuring a program as independent activities that may interleave (not necessarily for speed). Tools: atoms/refs/agents for shared state, core.async for coordination.

A pure, CPU-bound `map` over a big collection wants **parallelism**. A web server juggling many requests that touch shared state wants **concurrency** primitives. Don't reach for atoms to make a pure computation "parallel" — that's the wrong axis.

## 2. Easy data parallelism: `pmap`, `pvalues`, `pcalls`

The lowest-effort way to use multiple cores. `pmap` is `map` that evaluates in parallel across a thread pool.

```clojure
(defn expensive [x]
  (Thread/sleep 100)   ;; simulate work
  (* x x))

(map  expensive (range 8))   ;; ~800ms, sequential
(pmap expensive (range 8))   ;; ~roughly 100-200ms, parallel
```

```clojure
(pvalues (expensive 1) (expensive 2) (expensive 3))  ;; eval N exprs in parallel
(pcalls #(expensive 1) #(expensive 2))               ;; eval N thunks in parallel
```

**When `pmap` helps:** the per-item work is substantial and CPU-bound or blocking (a network call per item). **When it hurts:** tiny per-item work — the coordination overhead dominates and `pmap` is *slower* than `map`. It's also still semi-lazy and ordered, so it's not ideal for fully unordered streaming. For chunky CPU work, reducers (§5) usually beat `pmap`.

## 3. `future`, `promise`, `delay`

Three ways to defer or background a computation. `@x` / `(deref x)` retrieves the value.

```clojure
;; future: run a body on another thread NOW; deref blocks until it's done
(def f (future (Thread/sleep 1000) (+ 1 2)))
@f                       ;; blocks ~1s the first time, then => 3 instantly

;; promise: an empty box; one thread delivers, others block on deref
(def p (promise))
(future (Thread/sleep 500) (deliver p :ready))
@p                       ;; => :ready (after ~500ms)

;; delay: lazy + cached; body runs at most once, on first deref
(def d (delay (println "computing...") 42))
@d                       ;; prints "computing..." then => 42
@d                       ;; => 42 (no print; cached)
```

Use `future` for "go do this in the background." Use `promise` to hand a result between threads/callbacks. Use `delay` for expensive values you may not need. Add a timeout to a blocking deref with `(deref f 2000 :timed-out)`.

## 4. Managing shared state: atoms, refs, agents

Clojure separates **identity** (a thing that changes over time) from **value** (an immutable snapshot). A reference type is a managed, thread-safe identity that always *holds* an immutable value. Choose by the two axes — coordinated? synchronous?

| Type | Coordination | Sync/Async | Use when |
|---|---|---|---|
| **atom** | uncoordinated | synchronous | one independent piece of state |
| **ref** | coordinated (STM) | synchronous | multiple states that must change together |
| **agent** | uncoordinated | asynchronous | fire-and-forget updates, serialized per agent |

### Atoms — the 90% case

```clojure
(def counter (atom 0))

(swap! counter inc)            ;; apply fn → (inc @counter), returns new value
(swap! counter + 10)           ;; extra args passed to the fn
@counter                       ;; => 11
(reset! counter 0)             ;; set a value directly

;; compare-and-set! / swap! retry automatically under contention,
;; so the update fn MUST be pure (it may run more than once).
(swap! counter (fn [n] (* n 2)))
```

Atoms are lock-free and fast. The update function must be side-effect-free because it can be retried.

### Refs & STM — coordinated changes

When two pieces of state must update atomically (the classic bank transfer), wrap them in a `dosync` transaction. Either all changes commit or none do; the transaction retries on conflict.

```clojure
(def account-a (ref 100))
(def account-b (ref 0))

(defn transfer! [from to amount]
  (dosync
    (alter from - amount)
    (alter to   + amount)))

(transfer! account-a account-b 30)
[@account-a @account-b]         ;; => [70 30]
```

`alter` reads-then-writes (order matters across refs). `commute` is for commutative updates that don't conflict (counters, accumulators) and reduces retries. `ref-set` sets directly. Keep transaction bodies pure — they can replay.

### Agents — asynchronous, serialized

State you update by *sending* actions; they're applied one at a time, in order, on a thread pool. `deref` never blocks (you read the latest value).

```clojure
(def log (agent []))

(send log conj :event-1)        ;; queue an update (CPU-bound pool)
(send-off log (fn [es]          ;; use send-off for blocking/IO work
                (spit "log.txt" "...")
                es))
@log                            ;; latest value, no blocking

(await log)                     ;; block until queued actions for this agent finish
```

Agents serialize updates per agent, so they're a clean way to own a resource (a file, a connection) without locks. Errors are captured; check with `agent-error` and clear with `restart-agent`.

## 5. Reducers & `fold` (parallel reduction)

`clojure.core.reducers/fold` parallelizes a reduce by splitting the collection (a fork/join tree), reducing chunks on separate cores, then combining. Best for CPU-bound aggregation over large **vectors/maps** (foldable, splittable collections).

```clojure
(require '[clojure.core.reducers :as r])

(def data (vec (range 10000000)))

;; sequential
(reduce + (map inc data))

;; parallel: r/map is a transformation, r/fold runs it across cores
(r/fold + (r/map inc data))

;; fold with separate combine and reduce fns:
;; (r/fold combinef reducef coll)
(r/fold + (fn [acc x] (+ acc (* x x))) data)
```

For `fold` to actually parallelize, the source must be foldable (vectors, hash-maps — not lazy seqs) and the work per element should be non-trivial. For composing transformations, transducers (`references/code-generation.md`) are the modern, more general tool; reach for `fold` specifically when you want parallel reduction.

## 6. core.async (channels & CSP)

For **concurrency** — coordinating independent processes that communicate over channels (Go-style CSP). Not primarily a speed tool; it's about structure: pipelines, producers/consumers, backpressure, timeouts.

```clojure
(require '[clojure.core.async :as a
           :refer [chan go go-loop >! <! >!! <!! close! timeout]])

;; A channel is a thread-safe queue. go blocks are lightweight "processes".
(let [c (chan 1)]
  (go (>! c "hello"))            ;; park until a slot is free, then put
  (<!! c))                      ;; => "hello"  (<!! blocks the real thread)

;; Producer/consumer with a buffered channel
(let [c (chan 10)]
  (go-loop [i 0]                 ;; producer
    (when (< i 5)
      (>! c i)
      (recur (inc i))))
  (go-loop [acc []]              ;; consumer
    (if-let [v (<! c)]
      (recur (conj acc v))
      acc)))

;; timeout channel for deadlines / debouncing
(<!! (timeout 1000))            ;; returns nil after ~1s
```

Inside `go` blocks use the **parking** ops `<!` / `>!` (they yield the thread instead of blocking). Outside go blocks use the **blocking** ops `<!!` / `>!!`. `pipeline` and `pipeline-blocking` apply a transducer across a channel with parallelism. Reach for core.async when you have genuine coordination needs; for a one-shot background job, a `future` is simpler.

## 7. Thread pools & raw threads

`future`/`pmap`/`send` use built-in pools. For custom control, use Java executors directly:

```clojure
(import '(java.util.concurrent Executors))

(let [pool (Executors/newFixedThreadPool 4)]
  (try
    (->> (range 8)
         (map (fn [x] (.submit pool ^Callable (fn [] (* x x)))))
         doall                      ;; submit all
         (map #(.get %)))           ;; collect results
    (finally (.shutdown pool))))
```

On modern JVMs you can also use virtual threads for massive numbers of blocking tasks. Don't hand-roll this unless the standard tools genuinely don't fit.

## 8. Decision guide

```
Need to use multiple cores for a pure computation?
  • simple per-item work over a seq        → pmap (if work is chunky) or just map
  • aggregating a large vector/map         → reducers r/fold
  • a reusable, composable pipeline        → transducers

Need to run something in the background?
  • one result, later                      → future (+ deref, optional timeout)
  • hand a value between threads            → promise
  • expensive value maybe-never-needed     → delay

Need shared mutable state?
  • one independent value                  → atom
  • several values, change together        → refs + dosync (STM)
  • async, serialized, owns a resource     → agent

Need to coordinate independent processes / pipelines / backpressure?
                                           → core.async channels
```

**Golden rule:** because values are immutable, you only need these tools at the *boundaries* where real state or coordination lives. Keep the core of your program pure functions on plain data, and most "concurrency bugs" simply can't occur.
