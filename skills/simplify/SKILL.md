---
name: simplify
description: Review recently changed code for unnecessary complexity, duplication, and quality issues, then fix them. Goes beyond surface formatting to examine structure, naming, and responsibility.
when_to_use: Invoke explicitly with /simplify to review recently changed code for unnecessary complexity and fix it, e.g. after completing a task or accepting a batch of AI-generated code.
allowed-tools: Read Edit Bash Glob Grep
---

# Simplify

Review changed files and fix what you find. Report before fixing. List the issues first, then apply the changes.

## What to examine and fix

### Names (Fowler: *Mysterious Name*)
A name should answer three questions: why this thing exists, what it does, and how it is used. If any of those require a comment to answer, the name has failed. Fix the name, not the comment. Full names always. No single-letter variables outside of index counters, no abbreviations in comprehensions, no names that describe implementation rather than intent.

If the domain has a precise term for something, use it. A name like `user` where the domain uses `care_recipient` is a signal the name was chosen lazily rather than accurately.

### Single responsibility
A function that does two things should be two functions. Split anything that reads *and* writes, validates *and* transforms, or handles *and* logs. Name each piece after the single thing it does. If you can't name it without "and", it needs to be split.

A subtler tell: mixed levels of abstraction within a single function. A function that calls `render_page()` alongside `.append("\n")` is working at two different altitudes. High-level intent and low-level mechanics should be separate layers. The stepdown rule: a function's body should only contain calls one level of abstraction below its name, so the code reads top-to-bottom like a narrative rather than a mixed-altitude tangle.

### Dead code
Remove unused variables, imports, commented-out blocks, feature flags for features that shipped, and TODO comments for finished work. Dead code is a liability: it creates noise, misleads future readers, and accrues maintenance cost without benefit.

### Speculative abstraction (Fowler: *Speculative Generality*)
Remove abstractions added for hypothetical future uses. Three similar lines are better than a premature abstraction. If an abstraction has exactly one call site, ask whether the abstraction adds clarity or just indirection. Prefer directness.

### Duplication vs abstraction (Fowler: *Duplicated Code*)
Duplication is a problem when the same logic must change in multiple places. Extract it when that condition is met, not before. Don't extract early because the code looks similar; extract when the semantics are provably the same and the coupling is desired.

### Command-query separation
A function should either change state or return information, not both. A function that sets a value and returns a success flag forces the caller to use it as an expression in a conditional, making it ambiguous whether the word in the `if` is a verb or an adjective. Separate the command from the query. If you find yourself writing `if set("key", value):`, split it into `set("key", value)` followed by a separate existence check.

### Argument smells
**Flag arguments:** passing a boolean into a function is a declaration that the function does two things: one when the flag is true, another when it's false. Split it into two functions. `render(full=True)` should be `render_full()` and `render_single()`.

**Output arguments:** arguments carry information *into* a function; return values carry it *out*. A function that mutates its argument instead of returning a result inverts this expectation and forces a double-take at every call site. In object-oriented code, if the function needs to change state, it should change the state of its owning object, not a parameter passed in.

### Comments
Delete comments that describe *what* the code does. The code already does that. Keep only comments that explain *why* something non-obvious was done, or document a constraint, invariant, or known limitation that a future reader would otherwise have to rediscover. If a comment is the only thing making the code understandable, improve the code first.

### Error handling scope
Error handling belongs at the layer with enough context to do something meaningful with the error. A low-level function that swallows exceptions silently is wrong. A high-level layer that catches every error and logs generically is wrong. Each layer should let errors propagate unless it can genuinely handle them.

### Unnecessary defensiveness
Don't add null checks for values that can't be null. Don't validate inputs that have already been validated at the system boundary. Don't add fallbacks for code paths that are already guaranteed by framework or language invariants. Trust your tools.

### Structural smells (Fowler)
Name the smell when you spot it. Invoking the term sharpens both the diagnosis and the fix. Each reads *what it is* → *how to fix*; all are judgement calls, and a documented repo standard or an existing convention overrides any of them.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move it onto the data it envies.
- **Data Clumps**: the same few fields or parameters keep travelling together. → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept. → give the concept its own small type.
- **Repeated Switches**: the same switch/if-cascade on the same type recurs. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files. → gather what changes together into one module.
- **Divergent Change**: one module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

## Process

The disposition is the Boy Scout Rule: leave every file slightly cleaner than you found it. A better name, a smaller function, one fewer duplicate, a comment replaced by expressive code. Small improvements applied consistently compound; don't wait for a dedicated cleanup pass.

1. Read each changed file in full
2. List every issue found: one line each, file and approximate location
3. Apply fixes directly to the files
4. After editing: verify each file is internally consistent and follows the surrounding code style

Edit the issue list for concision before showing it. The report should read like the code it asks for: terse, concrete, no filler.
