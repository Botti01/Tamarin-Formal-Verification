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

### Definizioni in italiano

* **Fact**: Un *fact* è come un “contenitore di stato” o un predicato simbolico. Serve per memorizzare informazioni durante l’esecuzione del protocollo: ad esempio chi ha una chiave, cosa è stato inviato, cosa è stato ricevuto, o quali condizioni valgono in un certo momento. I facts possono essere lineari oppure persistenti.
* **Term**: Un *term* è un messaggio simbolico costruito con costanti, variabili e funzioni. Per esempio `h(k)`, `pk(~ltk)` e `aenc(m, pk(sk))` sono terms. In Tamarin, i messaggi non sono numeri o stringhe “concrete”, ma strutture simboliche.
* **Substitution**: Una *substitution* è una sostituzione di variabili con termini. Per esempio, se `x ↦ aenc(m, pk(sk))`, allora ogni occorrenza di `x` viene rimpiazzata con quel termine. In Tamarin le substitution servono per istanziare regole, fatti e vincoli.
* **Matching**: Il *matching* è il processo di verificare se un pattern può essere reso uguale a un termine tramite una sostituzione appropriata, senza dover cambiare liberamente entrambe le parti. In pratica si cerca una sostituzione per far combaciare un lato con l’altro.
* **Unification**: L’*unification* è più generale del matching: cerca una sostituzione che renda uguali due termini. Se il matching è “pattern contro termine”, l’unification è “termine contro termine” e cerca una soluzione comune. Se esistono più variabili, la soluzione può essere non unica.
* **Rewriting**: Il *rewriting* è la trasformazione di un termine o di uno stato usando una regola. In Tamarin, le regole di multiset rewriting descrivono come lo stato evolve: se i facts nel lato sinistro della regola sono presenti, la regola può scattare e produrre i facts del lato destro.
* **Multiset rewriting**: Il *multiset rewriting* è il meccanismo di base di Tamarin: lo stato è un multinsieme di facts, non un insieme ordinato. Applicare una regola significa consumare alcuni facts e produrne altri, rispettando le condizioni della regola.

### Versione breve da ricordare
* **facts** = stato
* **terms** = messaggi simbolici
* **substitutions** = variabili rimpiazzate da termini
* **matching** = adattare un pattern a un termine
* **unification** = trovare una sostituzione che renda uguali due espressioni
* **rewriting** = applicare una regola per trasformare lo stato

---

## 7. Rules: State Facts vs. Action Facts

Rules describe state transitions and are written as:

```tamarin
rule RuleName:
  [ Premises ] // State Facts
--[ Labels ]->   // Action Facts
  [ Conclusions ] // State Facts
```

**La differenza fondamentale:**
* **State Facts (Premesse/Conclusioni):** Stanno dentro le parentesi quadre `[ ]`. Esistono nello stato e vengono consumati (se lineari) o restano (se persistenti). *Non puoi usarli direttamente nei lemmi.*
* **Action Facts (Etichette):** Stanno dentro i trattini `--[ ]->`. Non fanno parte dello stato, ma vengono registrati nella "traccia" (la cronologia degli eventi). *I lemmi possono leggere SOLO questi fatti.*

```tamarin
rule Send_Message:
  [ Fr(~n) ]                    // State Fact (consumed)
--[ Send($A, ~n) ]->            // Action Fact (logged in the trace)
  [ Out(< $A, ~n >), State(~n) ] // State Facts (produced)
```

---

## 8. Restrictions

Le regole definiscono cosa *può* succedere, le restrizioni definiscono cosa è *permesso* che esista in una traccia valida. Sono fondamentali per filtrare comportamenti indesiderati che non hanno senso logico nel protocollo.
Se una traccia viola una restrizione, Tamarin la scarta.

**Esempio classico (Verifica di uguaglianza):**
Spesso usato per verificare firme, MAC o pattern matching rigidi.

```tamarin
restriction Equality:
  "All x y #i. Eq(x,y) @i ==> x = y"
```
*(Questa restrizione richiede che ci sia una regola con l'Action Fact `--[ Eq(a, b) ]->`)*

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
Quando scrivi lemmi su fatti persistenti o su protocolli con cicli infiniti (es. chiavi aggiornate continuamente, o sessioni ripetute), il prover base spesso va in un loop infinito.
Aggiungendo `[use_induction]`, Tamarin proverà a dimostrare il lemma per induzione.

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

* **Sanity Checks (Existential Lemmas):** Prima di provare che il protocollo è sicuro, prova che *può funzionare*. Scrivi un lemma con `exists-trace`. Se Tamarin non trova una traccia, significa che hai modellato regole che si bloccano a vicenda.
  ```tamarin
  lemma executable:
    exists-trace
    "Ex A B m #i #j. Send(A, m) @ #i & Receive(B, m) @ #j & #i < #j"
  ```
* **Injective vs Non-Injective Agreement:** Quando modelli l'autenticazione, ricorda la differenza. *Non-injective* significa che B sa di parlare con A. *Injective* garantisce anche che ad ogni sessione di B corrisponda una e una sola sessione di A (previene i *replay attack*).
* **Chiavi e Freshness:** Usa sempre il prefisso `~` per chiavi di sessione e nonce (es. `~k`). Se usi variabili pubbliche `$k`, l'avversario ne avrà immediatamente il controllo.
* **Evita regole troppo grosse:** Se una regola fa troppe cose contemporaneamente (es. riceve un messaggio, decritta, genera chiavi, esegue controlli complessi e invia la risposta), il prover fatica. Cerca di mantenere i passaggi atomici o logici.
* **Action Facts per i Secret:** Un modo elegante per testare la segretezza è emettere un Action Fact `--[ Secret(x) ]->` quando un agente calcola una chiave, e poi usare un lemma per verificare che l'avversario (rappresentato dal fact `K(x)`) non possa mai conoscerla.
  ```tamarin
  lemma secrecy:
    "All x #i. Secret(x) @ i ==> not (Ex #j. K(x) @ j)"
  ```
