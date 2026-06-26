# clojure-idiomatic

A Claude Code skill for writing, reviewing, and refactoring idiomatic Clojure — the way experienced Clojurists write it: data-first, immutable by default, built from small pure functions, and developed live at the REPL.

## What it does

This skill activates automatically when you are working in Clojure or ClojureScript. It guides Claude to produce code that follows established Clojure idioms instead of generic functional-programming patterns.

**Triggers on:** `.clj` / `.cljs` / `.cljc` files, `deps.edn`, Leiningen, the REPL, threading macros (`->`, `->>`), atoms/refs/agents, `core.async`, transducers, multimethods, protocols, or any request phrased as "make this more idiomatic."

**Core principles enforced:**

- Data over functions over macros
- Immutability as the default, not an opt-in
- Small pure functions composed at the edges
- REPL-driven development habits
- Simplicity over cleverness

## Included reference files

| File | Covers |
|---|---|
| `references/code-generation.md` | Namespace structure, transducers, error handling, clojure.spec |
| `references/parallelism.md` | `pmap`, `future`, atoms/refs/agents, `core.async`, reducers |
| `references/functions.md` | `defn`, arities, destructuring, multimethods, protocols, recursion |
| `references/immutability.md` | Persistent data structures, structural sharing, update patterns |

Claude reads only the reference files relevant to the current task, keeping token usage low.

## Installation

### Option A — Claude Code Plugin (recommended)

From within Claude Code, first add the marketplace:

```
/plugin marketplace add josephinoo/clojure-idiomatic
```

Then install the plugin:

```
/plugin install clojure-idiomatic@clojure-idiomatic
```

This makes the skill available across all your projects.

### Option B — Install the `.skill` bundle

```bash
# Download and install in your project directory
claude skill install clojure-idiomatic.skill
```

### Option C — Copy files manually

1. Copy `SKILL.md` and the `references/` folder into your project's `.claude/skills/clojure-idiomatic/` directory.
2. Claude Code will pick them up automatically.

## Usage examples

```
"Write a function that groups a list of orders by customer id"
→ Claude produces idiomatic Clojure using group-by, keywords, and threading macros

"Make this more idiomatic"
→ Claude refactors loop/recur into reduce, nested let into ->, etc.

"How do I share state safely between threads?"
→ Claude uses the parallelism reference and recommends atoms/refs/agents as appropriate
```

## Author

**Joseph Avila** — [@josephinoo](https://github.com/josephinoo)

## License

MIT
