---
name: Alloy
contributors:
    - ["Lenar Khannanov (Assisted-by: LLM)"]
---

Alloy is a lightweight, declarative specification language based on first-order
logic, designed for modeling complex structural relationships. Used with the
Alloy Analyzer to automatically check properties via SAT solving.

```alloy
// Single-line comment starts with //
/*
   Multi-line
   comment
*/

// ———————————————————————————————
// SIGNATURES (Data Types & Structures)
// ———————————————————————————————

// Abstract signature: no known instances (only known via relations)
sig Person {}

// Concrete signature: *may* have instances (but not required)
sig Student extends Person {}

// Concrete with fields (relations)
sig Task {
    // Fields are unary relations: Task → One (meaning: each task maps to exactly one value)
    name: one Name,
    due: one Date,
    done: lone Boolean  // `lone` = 0 or 1 instance (i.e., optional)
}

// Abstract base for names (only used via extensions)
sig Name {}

// Built-in types
sig Bool {}
one sig True in Bool {}
one sig False in Bool {}

// ———————————————————————————————
// RELATIONS (Binary Relations = Edges in Graphs)
// ———————————————————————————————

// A binary relation: assigns each person a set of tasks they own
sig Person {
    owned: set Task  // `set` = 0 or more tasks
}

// Other multiplicities:
// one    — exactly one
// lone   — at most one
// set    — zero or more
// some   — one or more

// Multiplicity is declared in the *field* signature, not the field *usage*

// Recursive structure (e.g., tree)
sig Node {
    children: set Node,
    parent: lone Node  // at most one parent
}

// ———————————————————————————————
// FACTS (Constraints that always hold)
// ———————————————————————————————

fact tree_properties {
    // No cycles
    all disj n1, n2: Node | n1 in n2.^(children) => n2 !in n1.^(children)
    
    // Every node (except root) has exactly one parent
    all n: Node | n != root => one n.parent
}

// ———————————————————————————————
// FUNCTIONS (Short-hand logic)
// ———————————————————————————————

// Recursive function to compute ancestors
fun ancestors[n: Node] : set Node {
    n.parent + ancestors[n.parent]
}

// Recursive function (must be acyclic — enforced by analyzer)
fun descendants[n: Node] : set Node {
    n.children + descendants[n.children]
}

// ———————————————————————————————
// PREDICATES (Named formulas)
// ———————————————————————————————

pred isRoot[n: Node] {
    no n.parent
}

pred allTasksDone[t: Task] {
    t.done = true
}

// ———————————————————————————————
// ASSERTIONS (Properties to check)
// ———————————————————————————————

// “In every instance, if a person owns a task, then they are a person.”
assert owned_implies_person {
    all p: Person | all t: p.owned | p in Person  // trivial, but illustrative
}

// “If all tasks are done, then the system is healthy.”
pred healthy {
    all t: Task | t.done = true
}

assert health_implies_no_pending {
    healthy => no (t: Task | t.done = false)
}

// ———————————————————————————————
// RUN & CHECK COMMANDS
// ———————————————————————————————

// Generate a sample instance (scope 3 = up to 3 atoms per signature)
run { some Person and some Task } for 3

// Check if assertion *can* be violated
check health_implies_no_pending for 4

// Look for a counterexample to "every person has at most one parent"
check { all p: Person | one p.parent } for 5

// “Generate all healthy instances”
run { healthy } for 3 but 1 Person, 2 Task

// Scope syntax:
// for N  — same bound for all sigs
// for N but Person:3, Task:5  — override per signature
// but 1 Bool  — only one atom per Bool (True/False merged)
// but exactly N X — X must have *exactly* N atoms

// ———————————————————————————————
// LATTICE & SET OPERATORS
// ———————————————————————————————

// Boolean logic: `and`, `or`, `not`, `implies`, `iff`
// Set operations:
//    r + s    — union
//    r & s    — intersection
//    r - s    — set difference
//    r - s    — same
//    r - s    — same again (no subtraction in all contexts? No! It *is* `r - s`)
//    r - s    — Yes, `r - s` is difference

// Closure operators:
//    r.*        — reflexive transitive closure (including identity)
//    r.^        — transitive closure (no self-loop)
//    r.* = id + r + r.r + r.r.r + ...

// Example: reachability
pred reachable[n1, n2: Node] {
    n2 in n1.*children
}

// ———————————————————————————————
// SCOPING & BOUNDS (Crucial!)
// ———————————————————————————————

// Every signature is interpreted as a *finite* set of atoms (≤ scope)
// Alloy *does not* support infinite models directly.

// Example: ordering with limited scope (you’ll see holes!)
sig Ord {
    next: lone Ord
}

fact {
    all disj o1, o2: Ord | o1 != o2 implies o1.next != o2.next  // injective next
    no o: Ord | o in o.*next  // acyclic
}

// With scope 3, this gives only a chain of 3 distinct elements.
// With scope 4, you can see a longer chain — but no *infinite* model.

// ———————————————————————————————
// METRICS & UTILS (for analysis tuning)
// ———————————————————————————————

// `pred` / `fun` / `fact` can include `implies`, quantifiers, etc.
// But *not* recursion in `fact` (use `fun` instead).

// Utility: `disj` — ensures distinctness in quantifiers
all disj a, b: Person | a.name != b.name  // a ≠ b ∧ a.name ≠ b.name

// `some`, `no`, `one`, `lone` — cardinality constraints
some x: Person | x.done = true  // “at least one person has done=true”
no t: Task | t.done = false     // “no pending tasks”
one p: Person | p in owner    // “exactly one owner”

// ———————————————————————————————
// EXAMPLE: Simple Library Model
// ———————————————————————————————

sig Book { title: one Title }
sig Title {} // abstract

sig Library {
    books: set Book,
    patrons: set Patron
}

sig Patron {
    borrowed: set Book
}

fact {
    // A book can be borrowed by at most one patron at a time
    all b: Book | one b.(borrowed.~)
}

pred checkout[b: Book, p: Patron] {
    // Precondition: book is in library, not currently borrowed
    b in books && no p'.borrowed[b]
    // Postcondition: p now borrows b
    borrowed' = borrowed + (p -> b)
}

// To check:
// check { all b, p, l: Library | checkout[b, p] in l => b.(borrowed') = p } for 2

// ———————————————————————————————
// KEY POINTERS
// ———————————————————————————————

// ✅ Alloy models are *relational* (all structure is edges in a graph).
// ✅ No procedures, no execution — only *declarative* constraints.
// ✅ Scope bound is mandatory — but you can refine per signature.
// ✅ `run` → find instances; `check` → find counterexamples.
// ✅ Alloy is *sound* and *complete* for its finite scopes.

// 🔧 Tips:
// • Use `one`, `lone`, `set`, `some` — they’re your best friends.
// • Prefer `fact` over `assert` for system-wide invariants.
// • Use `run` early to *see* the model; debug with visualization (click “Show/Hide View”).
// • Use `util/seq` for sequences: `open util/seq[Node]` gives `Seq[Node]`, etc.

// 📦 Built-in modules (import with `open`):
// open util/ordering[Person]  — total ordering of Person atoms
// open util/boolean          — Boolean utilities
// open util/relation         — advanced relation utils
// open util/arity            — for higher-arity relations

// 🎯 Try:
// - Modeling a database schema
// - A state machine (use `next: set State` + `init` + `step` preds)
// - A web session (users, pages, sessions)
// - A file system (dirs, files, links)
```
