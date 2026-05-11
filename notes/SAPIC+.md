# SAPIC+ Notes

## 1. Core idea

SAPIC+ (Stateful Applied PI-Calculus plus) is the **process language** integrated in Tamarin.

Instead of writing a protocol only as a set of multiset rewriting rules, you can describe it as a **process** with sequential control flow, parallel composition, state updates, inputs/outputs, conditionals, and events. Tamarin then **translates the process into rules**.

This is useful when you want a specification that looks closer to a program or an applied pi-calculus model.

**Mental model:**
- **Tamarin rules** = low-level operational model
- **SAPIC+ processes** = higher-level protocol description
- **Tamarin prover** = analyzes the translated model

---

## 2. Why use SAPIC+

SAPIC+ is convenient when the protocol has:
- many sequential steps
- local control flow
- input/output on channels
- shared state
- branching on conditions
- explicit events for lemmas

It is often easier to read than a large set of separate rules.

---

## 3. Basic syntax

A SAPIC+ process is built from constructs such as:

```sapic
new n; P
out(t); P
out(c, t); P
in(x); P
in(c, x); P
if cond then P else Q
let t1 = t2 in P else Q
P | Q
!P
0
```

### Meaning of the main constructs

- `new n; P`  
  Generates a fresh value `n` and continues with `P`.

- `out(t); P`  
  Sends `t` on the public channel.

- `out(c, t); P`  
  Sends `t` on channel `c`.

- `in(x); P`  
  Receives a message from the public channel and binds it to `x`.

- `in(c, x); P`  
  Receives from channel `c` and binds the received term to `x`.

- `if cond then P else Q`  
  Branches depending on a condition.

- `let t1 = t2 in P else Q`  
  Pattern matching / local binding. This is especially important for destructors.

- `P | Q`  
  Parallel composition.

- `!P`  
  Replication: infinitely many copies of `P` may run.

- `0`  
  Null process: terminate this branch.

---

## 4. Fresh values and `new`

In Tamarin rules you usually write:

```tamarin
[ Fr(~n) ]
-->
[ ... ]
```

In SAPIC+ the same idea is written as:

```sapic
new n; P
```

Example:

```sapic
new n; out(n); 0
```

This means: generate a fresh nonce `n`, send it, then stop.

### Intuition
- `Fr(~n)` is the rule-level view
- `new n` is the process-level view

---

## 5. Communication

### Public channel

If you write:

```sapic
out(t); P
in(x); P
```

SAPIC+ uses the public channel implicitly.

Example:

```sapic
out(<A, n>); in(x); 0
```

This is similar to Tamarin facts:

```tamarin
Out(<A, n>)
In(x)
```

### Explicit channel

You can also write:

```sapic
out(c, t); P
in(c, x); P
```

This means communication happens on channel `c`.

---

## 6. Conditionals

SAPIC+ has explicit branching:

```sapic
if t1 = t2 then P else Q
```

This is the process-level way to test equality.

Example:

```sapic
if m = expected then out(ok); 0 else out(error); 0
```

Use this when the protocol should continue only if two terms match.

---

## 7. Let bindings and destructors

This is one of the most important parts of SAPIC+.

```sapic
let t1 = t2 in P else Q
```

The `let` can do **pattern matching** and can also evaluate **destructors**.

### Example: pair projection

```sapic
let <x, y> = m in out(x); 0 else out(fail); 0
```

- If `m` is a pair, the binding succeeds.
- If not, the binding fails and the process continues with `else`.

### Example: decryption

```sapic
let m = adec(c, sk) in out(m); 0 else out(fail); 0
```

If decryption cannot be performed, the destructor fails and the `else` branch is taken.

### Important difference

In SAPIC+ the failure of a destructor is part of the process semantics.  
This is often clearer than encoding the same behavior only with rules.

---

## 8. Parallel composition and replication

### Parallel composition

```sapic
P | Q
```

Both processes run concurrently.

### Replication

```sapic
!P
```

This means an unbounded number of sessions of `P` may exist.

Example:

```sapic
! ( new n; out(n); 0 )
```

This models many independent sessions generating fresh values.

---

## 9. Global state

SAPIC+ supports explicit state manipulation.  
This is one of the biggest differences from a plain applied pi-calculus model.

### Insert

```sapic
insert k, v; P
```

Store value `v` under key `k`.

### Lookup

```sapic
lookup k as x in P else Q
```

If the key exists, bind its value to `x` and continue with `P`.  
Otherwise continue with `Q`.

### Delete

```sapic
delete k; P
```

Remove the entry associated with `k`.

### Example

```sapic
new s;
insert <'session', A>, s;
lookup <'session', A> as x in out(x); 0 else out(none); 0
```

This is useful for databases, revocation lists, counters, or stored session data.

---

## 10. Locking

SAPIC+ also has synchronization primitives:

```sapic
lock t; P
unlock t; P
```

These are used when multiple parallel processes access shared state and you want mutual exclusion.

**Typical intuition:**
- `lock` = enter critical section
- `unlock` = leave critical section

---

## 11. Events

You can emit events inside a process:

```sapic
event F; P
```

Events are crucial for lemmas, authentication properties, and trace reasoning.

Example:

```sapic
event Send(A, m); out(m); 0
```

This is similar to action facts in Tamarin rules.

---

## 12. Relationship with Tamarin rules

A Tamarin model typically looks like this:

```tamarin
rule Example:
  [ Fr(~n), In(m) ]
--[ Send(A, m) ]->
  [ Out(<A, ~n, m>) ]
```

The SAPIC+ view would be closer to:

```sapic
new n;
in(m);
event Send(A, m);
out(<A, n, m>);
0
```

### Useful intuition
- rules describe one transition at a time
- processes describe a flow of actions
- translation turns the process into rules automatically

---

## 13. Practical equivalence cheat sheet

### Fresh generation

```tamarin
Fr(~n)
```

```sapic
new n
```

### Output

```tamarin
Out(t)
```

```sapic
out(t)
```

### Input

```tamarin
In(x)
```

```sapic
in(x)
```

### Event

```tamarin
--[ F(t) ]->
```

```sapic
event F(t)
```

### Equality check

```tamarin
_restrict(t1 = t2)
```

or lemma/restriction logic

```sapic
if t1 = t2 then P else Q
```

### Pattern match / destructor

```tamarin
let <x,y> = m in ...
```

```sapic
let <x,y> = m in P else Q
```

### Persistent/shared state

```tamarin
!State(k,v)
```

```sapic
insert k, v; ...
lookup k as x in ...
```

---

## 14. A complete toy example

```sapic
process:
!
  new n;
  event Start(n);
  out(n);
  in(m);
  let <x, y> = m in
    event GotPair(x, y);
    out(x)
  else
    event ParseFailed(m);
    out(error)
```

### What it does
1. Creates a fresh nonce `n`.
2. Emits an event `Start(n)`.
3. Sends `n`.
4. Receives a message `m`.
5. If `m` is a pair `<x,y>`, it logs `GotPair(x,y)` and sends `x`.
6. Otherwise it logs `ParseFailed(m)` and sends `error`.

---

## 15. Modeling secrets and authentication

SAPIC+ itself does not magically prove security; it just gives you a cleaner way to specify the protocol.

You still use Tamarin-style lemmas over the translated model.

### Secrecy idea
Emit an event when a secret is created:

```sapic
event Secret(k);
```

Then prove that the attacker never learns it.

### Authentication idea
Use matching events such as:

```sapic
event Begin(A, B, n);
event End(B, A, n);
```

Then prove correspondence between them.

---

## 16. Common modeling tips

- Keep processes small and readable.
- Use `let` for destructors and pattern matching.
- Use `event` at the points you want to reason about in lemmas.
- Use `lookup/insert/delete` only when you really need shared state.
- Use `lock/unlock` if several parallel branches access the same state.
- Use `!P` for protocols with many sessions.
- Prefer explicit branching over overly complex nested patterns.

---

## 17. Typical mistakes

### 1. Confusing rule-level and process-level syntax

In SAPIC+ you do not write a rule like:

```tamarin
[ In(x) ] --> [ Out(x) ]
```

unless you are explicitly writing a Tamarin rule.

In SAPIC+ you write a process:

```sapic
in(x); out(x); 0
```

### 2. Forgetting that destructors can fail

A `let` with a pattern may go to `else`.  
Always think about the failure branch.

### 3. Ignoring shared-state synchronization

If multiple parallel sessions modify the same store, you may need `lock/unlock`.

### 4. Overusing process-level code for things that are clearer as rules

If the model is already very low-level, plain Tamarin rules may still be better.

---

## 18. Workflow for learning SAPIC+

1. Start from a simple protocol with only `new`, `in`, `out`, and `event`.
2. Add `let`-patterns.
3. Add replication `!P`.
4. Add state with `insert/lookup/delete`.
5. Add locks only when necessary.
6. Translate mentally between the process and the generated rules.

---

## 19. Recommended reading order

1. Grammar of SAPIC+ processes
2. Communication and parallelism
3. `let` and destructor failure
4. Global state primitives
5. Events and lemma reasoning
6. Examples from the official Tamarin repository

---

## 20. Short version to remember

- **Tamarin rules** = low-level state transitions
- **SAPIC+** = process language on top of Tamarin
- `new` = freshness
- `out` / `in` = communication
- `let` = binding + pattern matching + destructor failure
- `event` = trace fact for lemmas
- `insert/lookup/delete` = shared state
- `lock/unlock` = synchronization
- `!P` = replication

SAPIC+ is basically a more readable front-end for modeling the same kind of protocol behavior that Tamarin can already analyze.

## 21. Comparing rule-based and process-based modeling

A protocol can be modeled in terms of rules or as a single process. The process is translated into a set of rules that adhere to the semantics of process calculus. It is even possible to mix a process declaration with a set of rules, although this is not recommended, as the interactions between the rules and the process depend on how precisely this translation is defined.

## 22. Grammar details and conditionals

In the process grammar, `n` stands for a fresh name. Generic variables are denoted by `x`. Terms are denoted by `t`, `t1`, or `t2`. The letter `F` represents a fact. The term `cond` denotes a conditional, which is either a comparison `t1=t2` or a custom predicate. Most frequently, it is an equality check of the form `t1=t2`, but it is also possible to define a predicate using Tamarin's security property syntax.