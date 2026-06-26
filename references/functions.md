# Creating Functions in Clojure

Functions are the unit of everything in Clojure. This file covers how to define them well, give them flexible arities, pull apart their arguments, make them polymorphic, and handle recursion correctly.

## Table of contents
1. `defn`, `fn`, and the `#()` shorthand
2. Multiple arities
3. Variadic (rest) arguments
4. Destructuring arguments
5. Pre/post conditions & docstrings
6. Higher-order functions & composition
7. Polymorphism: multimethods
8. Polymorphism: protocols & records
9. Recursion done right
10. Memoization & closures

---

## 1. `defn`, `fn`, and `#()`

```clojure
;; named function with docstring
(defn square
  "Returns x multiplied by itself."
  [x]
  (* x x))

;; private function (not part of the namespace's public API)
(defn- helper [x] (inc x))

;; anonymous function — two equivalent forms
(fn [x] (* x x))
#(* % %)            ;; % is the first arg, %1 %2 … for more, %& for rest
```

Use the `#()` reader macro only for *short* one-liners. The moment it nests or needs more than one or two args, switch to `(fn [x y] …)` for readability. You cannot nest `#()` inside `#()`.

## 2. Multiple arities

A single function can have several argument lists. This is the idiomatic way to provide defaults — define the smaller arity in terms of the larger.

```clojure
(defn greet
  "Greets someone, with an optional greeting word."
  ([name]          (greet "Hello" name))     ;; 1-arg delegates to 2-arg
  ([greeting name] (str greeting ", " name "!")))

(greet "Sam")              ;; => "Hello, Sam!"
(greet "Hi" "Sam")         ;; => "Hi, Sam!"
```

This is cleaner than optional/keyword arguments in most other languages, and the arity is checked at compile time.

## 3. Variadic (rest) arguments

`&` collects remaining arguments into a sequence.

```clojure
(defn sum-all [& nums]
  (reduce + 0 nums))

(sum-all 1 2 3 4)          ;; => 10
(sum-all)                  ;; => 0

;; required args before the rest
(defn log-event [level & messages]
  {:level level :message (clojure.string/join " " messages)})

(log-event :info "user" "logged" "in")
;; => {:level :info, :message "user logged in"}
```

`apply` is the inverse — it spreads a sequence into positional args: `(apply + [1 2 3])` => `6`. A common "keyword args" pattern destructures the rest as a map:

```clojure
(defn make-conn [host & {:keys [port timeout] :or {port 80 timeout 5000}}]
  {:host host :port port :timeout timeout})

(make-conn "localhost" :port 8080)
;; => {:host "localhost", :port 8080, :timeout 5000}
```

## 4. Destructuring arguments

Destructuring lets a function pull values out of its arguments declaratively, in the parameter list.

### Map destructuring

```clojure
;; :keys grabs keys by name; :or supplies defaults; :as binds the whole map
(defn describe-user [{:keys [name age] :or {age 0} :as user}]
  (str name " is " age " (full record: " user ")"))

(describe-user {:name "Ana" :age 30})

;; namespaced keys
(defn order-id [{:order/keys [id customer]}]
  [id customer])
(order-id #:order{:id 1 :customer "Ana"})   ;; => [1 "Ana"]

;; bind a key under a different name with the long form
(defn point [{x :x, y :y}] [x y])

;; nested destructuring
(defn city-of [{{:keys [city]} :address}] city)
(city-of {:address {:city "Madrid"}})        ;; => "Madrid"
```

### Sequential destructuring

```clojure
(defn first-and-rest [[head & tail]]
  {:head head :tail tail})
(first-and-rest [1 2 3])     ;; => {:head 1, :tail (2 3)}

;; ignore positions with _
(defn third [[_ _ x]] x)
(third [10 20 30])           ;; => 30
```

Destructuring is the idiomatic alternative to repeatedly calling `(:key m)` inside a body. It documents exactly what shape the function expects.

## 5. Pre/post conditions & docstrings

A map literal at the top of a function body holding `:pre` / `:post` adds runtime assertions (`%` is the return value in `:post`).

```clojure
(defn sqrt
  "Square root of a non-negative number."
  [x]
  {:pre  [(>= x 0)]
   :post [(>= % 0)]}
  (Math/sqrt x))

(sqrt -1)   ;; throws AssertionError: Assert failed: (>= x 0)
```

Always give public functions a docstring (the string right after the name). It shows up in `(doc fn-name)` at the REPL and in tooling.

## 6. Higher-order functions & composition

Functions are values: pass them, return them, store them.

```clojure
(map inc [1 2 3])              ;; => (2 3 4)
(filter odd? (range 10))      ;; => (1 3 5 7 9)
(reduce max [3 1 4 1 5])      ;; => 5

;; comp — compose right-to-left (like math: (f∘g)(x) = f(g(x)))
(def inc-then-double (comp #(* 2 %) inc))
(inc-then-double 4)           ;; => 10

;; partial — fix leading arguments
(def add-10 (partial + 10))
(add-10 5)                    ;; => 15

;; juxt — run several fns on the same input, collect results in a vector
((juxt inc dec) 5)            ;; => [6 4]
((juxt :name :age) {:name "Ana" :age 30})  ;; => ["Ana" 30]

;; fnil — supply a default when an arg would be nil
((fnil + 0) nil 5)            ;; => 5

;; complement — negate a predicate
(def not-empty? (complement empty?))
```

Returning a function (a closure) is common and idiomatic:

```clojure
(defn multiplier [n]
  (fn [x] (* x n)))           ;; closes over n

(def triple (multiplier 3))
(triple 5)                    ;; => 15
```

## 7. Polymorphism: multimethods

When behavior should vary by an arbitrary property of the data, `defmulti` + `defmethod` give you open, runtime dispatch on *any* function of the args.

```clojure
;; dispatch on the :shape key
(defmulti area :shape)

(defmethod area :circle [{:keys [radius]}]
  (* Math/PI radius radius))

(defmethod area :rectangle [{:keys [width height]}]
  (* width height))

(defmethod area :default [shape]
  (throw (ex-info "Unknown shape" {:shape shape})))

(area {:shape :circle :radius 2})       ;; => ~12.566
(area {:shape :rectangle :width 3 :height 4})  ;; => 12
```

Strengths: anyone can add a new `defmethod` for a new case without touching the original code (open for extension), and dispatch can be on anything — a keyword, a vector of types, a computed value. The dispatch function can be arbitrarily clever:

```clojure
(defmulti convert (fn [from to _] [from to]))
(defmethod convert [:m :ft] [_ _ v] (* v 3.281))
(defmethod convert [:ft :m] [_ _ v] (/ v 3.281))
```

## 8. Polymorphism: protocols & records

When you want fast, type-based dispatch (the closest thing to interfaces), use `defprotocol` for the contract and `defrecord` (or `extend-protocol`) for implementations.

```clojure
(defprotocol Shape
  "Geometric operations."
  (area [this])
  (perimeter [this]))

(defrecord Circle [radius]
  Shape
  (area [_] (* Math/PI radius radius))
  (perimeter [_] (* 2 Math/PI radius)))

(defrecord Rectangle [width height]
  Shape
  (area [_] (* width height))
  (perimeter [_] (* 2 (+ width height))))

(area (->Circle 2))            ;; => ~12.566
(perimeter (map->Rectangle {:width 3 :height 4}))  ;; => 14
```

You can also extend a protocol to types you don't own, including core ones:

```clojure
(extend-protocol Shape
  String
  (area [s] (count s))
  (perimeter [s] (* 2 (count s))))
```

**Multimethods vs protocols:** use protocols for type-based dispatch where performance matters and the "axis" is the type. Use multimethods for dispatch on data values/properties, multiple arguments, or hierarchies. When in doubt, start with a plain function; reach for these only when you actually have multiple cases to dispatch between.

## 9. Recursion done right

Clojure runs on the JVM, which has no automatic tail-call optimization, so naive deep recursion blows the stack. Use the right tool:

```clojure
;; loop/recur — explicit tail recursion, constant stack. recur jumps to loop.
(defn factorial [n]
  (loop [acc 1, i n]
    (if (<= i 1)
      acc
      (recur (* acc i) (dec i)))))   ;; recur MUST be in tail position

;; recur straight to the fn args (no loop needed)
(defn count-down [n]
  (when (pos? n)
    (println n)
    (recur (dec n))))

;; mutual recursion → trampoline (functions return thunks)
(declare odd?* even?*)
(defn even?* [n] (if (zero? n) true  #(odd?*  (dec n))))
(defn odd?*  [n] (if (zero? n) false #(even?* (dec n))))
(trampoline even?* 1000000)          ;; => true, no stack overflow

;; processing a collection? usually you don't need recursion at all —
;; reduce expresses the same accumulation:
(defn factorial' [n] (reduce * 1 (range 1 (inc n))))
```

The compiler verifies `recur` is in tail position and that arity matches — if it complains, your `recur` isn't actually the last thing evaluated. For lazy, possibly-infinite sequences, build with `lazy-seq`, `iterate`, or `cons` instead of recurring eagerly.

## 10. Memoization & closures

`memoize` wraps a pure function so repeated calls with the same args return a cached result:

```clojure
(def slow-square
  (memoize (fn [x] (Thread/sleep 1000) (* x x))))

(slow-square 4)   ;; ~1s the first time
(slow-square 4)   ;; instant — cached
```

Only memoize *pure* functions, and beware unbounded cache growth for functions called with many distinct args (use a real cache library like `core.cache` for eviction). Closures (functions that capture surrounding bindings, §6) are the building block for stateful-looking behavior without global mutable state.
