# Tidy Code Heuristics

Small, safe structural improvements that make behaviour changes easier.
Always commit structural changes separately from behaviour changes.

---

## Code-Level Heuristics

### Guard Clauses

Replace nested conditions with early returns.

```go
// Before
func process(user *User) {
    if user != nil {
        if user.Active {
            // ...main logic...
        }
    }
}

// After
func process(user *User) {
    if user == nil {
        return
    }
    if !user.Active {
        return
    }
    // ...main logic...
}
```

- Only apply when the `if` wraps *all* remaining code in the function.
- Don't overdo it — seven guard clauses is not cleaner.

---

### Dead Code

Delete it. Version control keeps it if you ever need it back.

- If unsure whether code is called, add logging first and wait.
- Delete one logical unit at a time (one clause, one function, one file).

---

### Normalise Symmetries

When the same problem is solved multiple ways, pick one and convert.

```go
// Multiple variants doing the same thing — pick one:
if foo == nil {
    foo = defaultFoo()
}
// vs
foo = cmp.Or(foo, defaultFoo())
```

Difference in code implies difference in intent. Unnecessary variation confuses readers.

---

### New Interface, Old Implementation

When an existing interface is awkward, create the interface you *wish* existed and implement it by delegating to the old code.

```go
// New clean wrapper — delegates to messy internals
func ProcessItem(id string, value int) error {
    return legacyProcess(map[string]any{"id": id, "value": value})
}
```

Migrate callers over time, then inline or remove the old implementation.

---

### Reading Order

Reorder elements in a file so a reader encounters them in the order most helpful for understanding.

- You just read it — you know what order would have helped.
- Don't mix reordering with other changes in the same commit.
- In Go, exported symbols first is usually the right order.

---

### Cohesion Order

Move coupled elements next to each other before changing them.

- Applies to: functions in a file, files in a package, packages in a module.
- Not the same as eliminating coupling — just reduces the *cost* of the coupling.

---

### Move Declaration and Initialisation Together

Declare a variable as close as possible to where it is first used and initialised.

```go
// Before
var result []string
rows, err := db.Query(q)
// ...unrelated setup...
result = transform(rows)

// After
rows, err := db.Query(q)
// ...unrelated setup...
result := transform(rows)
```

Respect data dependencies — if `b` depends on `a`, `a` must be initialised first.

---

### Explaining Variables

Extract a complex sub-expression into a well-named variable.

```go
// Before
return image.Point{X: r.Min.X + (r.Max.X-r.Min.X)/2, Y: r.Min.Y + (r.Max.Y-r.Min.Y)/2}

// After
cx := r.Min.X + (r.Max.X-r.Min.X)/2
cy := r.Min.Y + (r.Max.Y-r.Min.Y)/2
return image.Point{X: cx, Y: cy}
```

Separates sub-expressions so each is easier to change independently.

---

### Explaining Constants

Replace magic literals with named constants.

```go
// Before
if resp.StatusCode == 429 {

// After
const statusTooManyRequests = 429
if resp.StatusCode == statusTooManyRequests {
```

The same literal can appear twice with different meanings — don't merge them blindly.

---

### Explicit Parameters

Make implicit inputs (globals, environment variables, mutable maps) into explicit function arguments.

```go
// Before
func process(params map[string]any) {
    // buried: params["timeout"], params["retries"]
}

// After
func process(params map[string]any) {
    processWithConfig(params["timeout"].(int), params["retries"].(int))
}

func processWithConfig(timeout, retries int) {
    // ...
}
```

Makes code easier to read, test, and reason about.

---

### Chunk Statements

Put a blank line between logically distinct blocks of code.

```go
rows, err := db.Query(q)
if err != nil {
    return err
}

users := make([]User, 0, len(rows))
for _, r := range rows {
    users = append(users, toUser(r))
}

return users, nil
```

The simplest heuristic here. Often the first step that reveals Extract Helper or comment opportunities.

---

### Extract Helper

Extract a block of code with a clear, limited purpose into a named function. Name it after the *purpose*, not the mechanism.

```go
// Before
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // ...auth check...
    token := r.Header.Get("Authorization")
    if token == "" {
        http.Error(w, "unauthorised", http.StatusUnauthorized)
        return
    }
    // ...main handler logic...
}

// After
func isAuthorised(r *http.Request) bool {
    return r.Header.Get("Authorization") != ""
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    if !isAuthorised(r) {
        http.Error(w, "unauthorised", http.StatusUnauthorized)
        return
    }
    // ...main handler logic...
}
```

Also use to encode temporal coupling (`a()` must be called before `b()`):

```go
func openAndMigrate(cfg Config) (*sql.DB, error) {
    db, err := open(cfg)
    if err != nil {
        return nil, err
    }
    return db, migrate(db)
}
```

---

### One Pile

When over-fragmented code is harder to understand than a single block, inline it all first, then re-extract with clearer structure.

Symptoms that suggest inlining:
- Long, repeated argument lists across helpers
- Repeated conditionals scattered across functions
- Poorly named helpers that obscure what's happening
- Shared mutable state passed between many small functions

---

### Explaining Comments

Write down what wasn't obvious from the code — the *why*, not the *what*.

```go
// Batch inserts here to avoid N+1 round-trips; do not refactor into a loop.
```

- Write to someone unfamiliar with this domain area.
- Comment immediately when you spot a non-obvious constraint or coupling.

---

### Delete Redundant Comments

Remove any comment that merely restates what the code already says.

```go
// BAD
// GetUser returns the user
func GetUser(id string) (*User, error) {
```

When a refactor makes a comment's context self-evident, delete the comment too. The goal is to make all comments non-redundant, not to have no comments.

---

## Managing Structural Changes

### Separate Structure from Behaviour

Keep structural commits and behaviour-change commits separate.

- Structural PRs should contain only structural changes.
- Behaviour PRs should contain only behaviour changes.
- This makes review faster, reverts safer, and intent clearer.

---

### Heuristics Chain

Structural improvements unlock further improvements. Common chains:

| Done | Next |
|---|---|
| Guard clause | Explaining variable or extract helper |
| Dead code removed | Reading order or cohesion order |
| Normalise symmetries | Reading order (parallels are now visible) |
| Chunk statements | Explaining comment or extract helper |
| Extract helper | Guard clause, explaining constants, delete redundant comments |
| One pile | Chunk statements → explaining comments → extract helpers |
| Explaining variable | Extract helper; delete redundant comments |
| Explaining comment | Move the info into the code (variable / constant / helper) |

---

### Batch Sizes

Prefer small batches of structural changes before integrating.

| Too many per batch | Too few per batch |
|---|---|
| More merge collisions | Higher fixed review cost |
| Higher chance of accidental behaviour change | (acceptable if review is cheap) |
| Encourages speculative refactoring | |

---

### When to Tidy

**First** — when it makes the upcoming change immediately easier and you know what to do.

**After** — when you couldn't see the improvement until after the change, and you'll be back in this area soon.

**Later** — when there's eventual payoff but no immediate need; keep a list and chip away when you have slack.

**Never** — when the code will never change again.

---

## Theory

### Coupling

> Two elements are **coupled** w.r.t. a change if changing one requires changing the other.

- Coupling is what drives the cost of software.
- Analyse coupling with respect to *specific changes*, not in the abstract.
- Two danger properties: **1–N** (one change fans out to many) and **cascading** (changes trigger further changes).
- Files that frequently appear together in commits are coupled.

### Cohesion

> Coupled elements should live inside the same containing element.

- Group things that change together; separate things that don't.
- Extract a cohesive helper or subpackage for tightly coupled pieces.
- Move uncoupled elements out to where they *are* coupled.
- Make one move at a time — you're working with incomplete information.

### The Core Equation

```
cost(software) ≈ cost(change) ≈ coupling
```

Reduce coupling → reduce the cost of change → reduce the overall cost of software.

---

## References

- Kent Beck — *Tidy First?*
