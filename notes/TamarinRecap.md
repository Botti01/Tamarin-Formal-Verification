# Tamarin Prover Notes

## 1. Core idea

Tamarin is a symbolic tool for modeling and analyzing security protocols. It works by describing protocol behavior as a set of **multiset rewriting rules**, the attacker’s capabilities, and the properties that should hold. The protocol state is represented symbolically, together with network messages, fresh values, and adversary knowledge.

Tamarin supports both automatic and interactive proof search. When automation is not enough, the GUI helps inspect proof states, attack graphs, and intermediate constraints.

---

## 2. Cryptographic primitives & Built-ins

```tamarin
builtins: hashing, asymmetric-encryption, symmetric-encryption, signing, diffie-hellman
```

This declares the built-in cryptographic primitives used by the model. Tamarin automatically handles the equational theories for these:
* **Hashing:** A unary function `h(x)`.
* **Asymmetric encryption:** `aenc(m, pk)`, `adec(c, sk)`, and `pk(sk)` for the public key.
* **Symmetric encryption:** `senc(m, k)` and `sdec(c, k)`.
* **Signing:** `sign(m, sk)` and `verify(sig, m, pk(sk))`. The function `true` is often used to represent successful verification.
* **Diffie-Hellman:** Enables symbolic exponentiation (e.g., `g^~x`), which is fundamental for key exchange protocols like TLS or IKE.

A key equation of the built-in asymmetric encryption theory is that decrypting a ciphertext with the correct private key returns the original plaintext.

---

## 3. Modeling a Public Key Infrastructure

In Tamarin, the protocol and its environment are modeled with rules over facts. For example:
* `Out(m)` means that message `m` is sent on the public channel.
* `In(m)` means that message `m` is received from the public channel.

A typical PKI rule is:

```tamarin
rule Register_pk:
  // Generate a fresh long-term private key
  [ Fr(~ltk) ]
-->
  // Add persistent facts to the state
  [ !Ltk($A, ~ltk), !Pk($A, pk(~ltk)) ]
```

This means that when a fresh long-term private key `~ltk` is generated, Tamarin adds the persistent facts `!Ltk($A, ~ltk)` and `!Pk($A, pk(~ltk))` to the state. Persistent facts remain available for future rule applications.

---

## 4. Notation & Message Syntax

* `~x` denotes `x:fresh`
* `$x` denotes `x:pub`
* `%x` denotes `x:nat`
* `#i` denotes `i:temporal`
* `m` denotes `m:msg`

**Message Tuples:**
When you need to group multiple pieces of data into a single message (e.g., sending an ID, a nonce, and a ciphertext), use angle brackets to create a tuple: `<A, ~n, aenc(m, pkB)>`.

**Useful intuition:**
* `~x` is a fresh value, usually created by `Fr(~x)`.
* `$x` is a public value.
* `%x` is a natural number.
* `#i` is a timepoint variable.
* `m` is a general message term.

**Also:**
* `!` before a fact means that the fact is persistent.
* `@ #i` means that a fact or action holds at trace position `#i`.
* `#i < #j` means that event `i` happens before event `j`.

---

## 5. Sorts and constants

Tamarin uses a top sort `msg`, with subsorts such as `fresh`, `pub`, and `nat`. A constant like `'c'` denotes a public name. Timepoint variables belong to sort `temporal` and are used to reason about trace ordering.

---

## 6. Facts, terms, substitutions, matching, unification, rewriting

### Definitions

* **Fact**: A fact is like a "state container" or a symbolic predicate. It is used to store information during protocol execution: for example, who has a key, what was sent, what was received, or what conditions hold at a given moment. Facts can be linear (consumable) or persistent.
* **Term**: A term is a symbolic message built with constants, variables, and functions. For example, `h(k)`, `pk(~ltk)`, and `aenc(m, pk(sk))` are terms. In Tamarin, messages are not concrete numbers or strings, but symbolic structures.
* **Substitution**: A replacement of variables with terms. For instance, if `x ↦ aenc(m, pk(sk))`, then every occurrence of `x` is replaced with that term. In Tamarin, substitutions are used to instantiate rules, facts, and constraints.
* **Matching**: The process of checking whether a pattern can be made equal to a term via an appropriate substitution, without freely changing both sides. Essentially, it means adapting a pattern to fit a term.
* **Unification**: More general than matching: it seeks a substitution that makes two terms equal. If matching is "pattern versus term," unification is "term versus term" and looks for a common solution. If multiple variables exist, the solution might not be unique.
* **Rewriting**: The transformation of a term or state using a rule. In Tamarin, multiset rewriting rules describe how the state evolves: if the facts on the left side of the rule are present, the rule can fire and produce the facts on the right side.
* **Multiset rewriting**: Tamarin's core mechanism: the state is a multiset of facts, not an ordered set. Applying a rule means consuming some facts and producing others, while respecting the rule's conditions.

### Short version to remember
* **facts** = state
* **terms** = symbolic messages
* **substitutions** = variables replaced by terms
* **matching** = fitting a pattern to a term
* **unification** = finding a substitution that makes two expressions equal
* **rewriting** = applying a rule to transform the state

---

## 7. Rules: State Facts vs. Action Facts

Rules describe state transitions and are written as:

```tamarin
rule RuleName:
  [ Premises ] // State Facts
--[ Labels ]->   // Action Facts
  [ Conclusions ] // State Facts
```

**The fundamental difference:**
* **State Facts (Premises/Conclusions):** They are placed inside square brackets `[ ]`. They exist in the state and are consumed (if linear) or remain (if persistent). *You cannot use them directly in lemmas.*
* **Action Facts (Labels):** They are placed inside the dashes `--[ ]->`. They are not part of the state, but are recorded in the "trace" (the history of events). *Lemmas can ONLY read these facts.*

```tamarin
rule Send_Message:
  [ Fr(~n) ]                    // State Fact (consumed)
--[ Send($A, ~n) ]->            // Action Fact (logged in the trace)
  [ Out(< $A, ~n >), State(~n) ] // State Facts (produced)
```

---

## 8. Restrictions

Rules define what *can* happen, restrictions define what is *allowed* to exist in a valid trace. They are fundamental for filtering out undesired behaviors that make no logical sense in the protocol.
If a trace violates a restriction, Tamarin simply discards it.

**Classic example (Equality check):**
Often used to verify signatures, MACs, or strict pattern matching.

```tamarin
restriction Equality:
  "All x y #i. Eq(x,y) @i ==> x = y"
```
*(This restriction requires that there is a rule with the Action Fact `--[ Eq(a, b) ]->`)*

---

## 9. Lemmas and Induction

A lemma states a property that Tamarin should prove or refute. Lemmas are trace properties: they talk about what must or must not happen in any execution trace.

**Typical logical symbols:**
* `Ex` = exists
* `All` = for all
* `&` = and
* `|` = or
* `not(...)` = negation
* `==>` = implication

**Induction:**
When you write lemmas on persistent facts or on protocols with infinite loops (e.g., continuously updated keys, or repeated sessions), the basic prover often goes into an infinite loop.
By adding `[use_induction]`, Tamarin will try to prove the lemma by induction.

```tamarin
lemma My_Security_Property [use_induction]:
  "All x #i. Action(x) @ #i ==> (Ex #j. OtherAction(x) @ #j & #j < #i)"
```

---

## 10. Adversary model

By default, Tamarin assumes a Dolev–Yao adversary: the attacker controls the network and can intercept, delete, modify, and inject messages, while still being limited by the symbolic cryptographic equations of the model.

---

## 11. GUI arrows and colors

In the GUI, Tamarin uses different arrow styles and colors to visualize the proof graph.

### Solid arrows
* **Black or gray solid arrows:** origins of protocol facts, for linear or persistent facts.
* **Solid red/orange arrows:** steps where the adversary extracts values from a received message.

### Dashed arrows
Dashed arrows represent ordering constraints between actions:
* **Black dashed:** constraint from formulas, for example from a lemma or a restriction.
* **Dark blue dashed:** constraint caused by a fresh value.
* **Red dashed:** constraint from adversary composition steps.
* **Dark orange dashed:** constraint implied by Tamarin’s normal-form conditions.
* **Purple dashed:** constraint originating from an injective fact instance.

> *Note: An arrow can have multiple colors if several constraints apply at the same time.*

### Dotted arrows
* **Dotted green arrows:** incomplete adversary deduction steps during proof search.

### Graph simplification
Tamarin can simplify graphs by hiding some rules/arrows. The GUI offers different simplification levels, and the “Options” menu lets you change how much detail is shown.

---

## 12. Useful workflow

1. Define cryptographic primitives (`builtins`).
2. Model state and protocol behavior with rules.
3. Define `restrictions` to filter invalid states.
4. State desired properties as `lemmas`.
5. Run the automatic prover:
   ```bash
   tamarin-prover Protocol.spthy --prove
   ```
6. Inspect the GUI if the proof gets stuck or if an attack is found.

---

## 13. Tips & Best Practices (Extra)

* **Sanity Checks (Existential Lemmas):** Before proving that the protocol is secure, prove that it *can work*. Write a lemma with `exists-trace`. If Tamarin cannot find a trace, it means you have modeled rules that block each other.
  ```tamarin
  lemma executable:
    exists-trace
    "Ex A B m #i #j. Send(A, m) @ #i & Receive(B, m) @ #j & #i < #j"
  ```
* **Injective vs Non-Injective Agreement:** When modeling authentication, remember the difference. *Non-injective* means B knows he is talking to A. *Injective* also guarantees that to each session of B corresponds exactly one session of A (prevents *replay attacks*).
* **Keys and Freshness:** Always use the `~` prefix for session keys and nonces (e.g., `~k`). If you use public variables `$k`, the adversary will immediately have control over them.
* **Avoid overly large rules:** If a rule does too many things simultaneously (e.g., receives a message, decrypts, generates keys, performs complex checks, and sends the response), the prover struggles. Try to keep steps atomic or logical.
* **Action Facts for Secrets:** An elegant way to test secrecy is to emit an Action Fact `--[ Secret(x) ]->` when an agent computes a key, and then use a lemma to verify that the adversary (represented by the fact `K(x)`) can never know it.
  ```tamarin
  lemma secrecy:
    "All x #i. Secret(x) @ i ==> not (Ex #j. K(x) @ j)"
  ```

---

## 14. Let Bindings and Global Macros

When a complex term is repeated multiple times within the same rule, you can use the `let ... in` construct to create local macros. This makes protocol specifications much more readable.

```tamarin
rule MyRuleName:
  let
    foo = h(~x, y)
  in
  [ In(foo) ] --> [ Out(foo) ]
```

If you want to use the same macros across multiple rules, lemmas, or restrictions, you can define them globally using the `macros:` keyword.

---

## 15. Lemma Annotations (`[reuse]` and `[sources]`)

Besides `[use_induction]`, there are other lemma annotations that deeply change how Tamarin executes proofs:
* **`[reuse]`**: A lemma with this annotation will be used in the proofs of all subsequent lemmas. It is a very useful mechanism for proving intermediate lemmas and simplifying the final proof.
* **`[sources]`**: This is essential when Tamarin fails to finish a proof due to infinite loops during resolution (the so-called *partial deconstructions*). *Sources lemmas* are applied automatically to help refine fact origins during precomputation.

---

## 16. Observational Equivalence (Privacy)

Classical lemmas reason over single protocol execution traces. To express privacy or cryptographic indistinguishability properties, Tamarin supports *Observational Equivalence*.
This feature allows proving that an intruder cannot distinguish between two instances of a system. It is modeled using the `diff(x, y)` operator for terms that differ between the two instances and requires starting Tamarin from the terminal with the `--diff` flag.

---

## 17. Other Useful Built-ins

In addition to the basic primitives, the manual provides other built-ins that are essential in advanced modeling:
* **`xor`**: Models the exclusive OR (XOR) operation and automatically includes the related logical cancellation equations.
* **`multiset`**: Introduces the associative-commutative `++` operator to model multisets.
* **`natural-numbers`**: Defines the `%1` constant and the `%+` operator to implement state counters.

---

## 18. Process Calculus (SAPIC+)

Instead of writing fragmented multiset rewriting rules, SAPIC+ (Stateful Applied PI-Calculus) allows you to model a protocol as a sequential flow, similar to classical programming. Tamarin will then automatically compile the process into the underlying multiset rewriting rules.

This is the basic syntax for defining processes (`P`, `Q`):

### Basic Constructs
* `new ~n; P` : Generates a fresh value `~n`.
* `out(t); P` : Sends term `t` on the public channel.
* `in(x); P` : Receives a message from the public channel and binds it to variable `x`.
* `out(c, t); P` : Sends `t` on a private/authenticated channel `c`.
* `in(c, x); P` : Receives from a channel `c` and binds to `x`.
* `if cond then P else Q` : Conditional branch (e.g., to check if two values match).
* `let x = t in P else Q` : Pattern matching or assignment.
* `P | Q` : Parallel execution (the two processes proceed simultaneously).
* `!P` : Replication (generates infinite concurrent instances of `P`, essential for modeling multiple sessions).
* `0` : Null process (terminates the branch).

### Global State Management (Stateful)
Unlike traditional Pi-Calculus, SAPIC+ allows managing shared key-value databases between processes, which is fundamental for modeling complex logic like certificate revocations or counters:
* `insert k, v; P` : Inserts or updates the value `v` associated with the key `k`.
* `lookup k as x in P else Q` : Looks up the key `k`. If it exists, binds its value to `x` and continues in `P`, otherwise executes `Q`.
* `delete k; P` : Removes the key `k` from the global state.

### Events and Synchronization
* `event F; P` : Emits an Action Fact `F` exactly when the process reaches that point (necessary for lemmas and history tracking).
* `lock t; P` and `unlock t; P` : Mutual exclusion mechanism to avoid race conditions when multiple processes attempt to access the same global state in parallel.

### Practical Example

```tamarin
process:
!(
  // Start a new session with a fresh key
  new ~k;
  
  // Emit an event for security lemmas
  event SessionStarted(~k);
  
  // Store the key in the global database with a public identifier
  insert <'session_key', $A>, ~k;
  
  // Send the key encrypted with B's public key
  out(aenc(~k, pkB));
  
  // Terminate this process instance
  0
)
```

