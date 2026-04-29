# Tamarin Prover: GUI and Graph Interpretation Guide

When working with Tamarin, the terminal output is often not enough. Tamarin provides a powerful web-based Graphical User Interface (GUI) to interactively explore proofs and visualize attack graphs. This guide explains how to navigate the interface and interpret the visual diagrams.

## 1. Starting the Interactive Server

To launch the GUI, open your terminal in the folder containing your `.spthy` files and run:
```bash
tamarin-prover interactive .
```
Then, open your browser and navigate to `http://localhost:3001` (or click the link provided in the terminal).

## 2. Navigating the Interface

### The Left Sidebar (Theory List)
- When you open the UI, the left pane shows all `.spthy` files in your current directory.
- Clicking on a file loads its **Theory**.

### The Main View (Theory Details)
Once a theory is selected, the left sidebar expands to show its components:
1. **Message Theory**: The cryptographic functions and equations you defined (e.g., `senc`, `h1`).
2. **Rules**: The multiset rewriting rules of your protocol (e.g., `Alex_nonce`, `Blake_ack`).
3. **Lemmas**: The properties you want to prove (e.g., `BothDone`, `Key_Secrecy`).

---

## 3. The Proof Tree

When you click on a lemma, Tamarin attempts to prove it. The central pane will display a **Proof Tree**.

### Color Coding
- **🟢 Green**: The lemma is **Verified** (or the specific branch is resolved).
- **🔴 Red**: The lemma is **Falsified**. This means Tamarin found a counterexample (an attack!).
- **🟠 Orange / Gray**: The proof is **Incomplete** (open goals remain). Tamarin is stuck or waiting for you to make a choice.

### Autoproving vs. Manual Proving
- **Autoprove**: Right-click on an orange node and select `autoprove`. Tamarin will try all heuristics to close the branch automatically.
- **Manual Proving**: If Tamarin loops indefinitely, you can click on individual open goals (the numbered lines in the center pane) to guide the prover step-by-step.

---

## 4. How to Read the Diagrams (Attack Graphs)

When a lemma is falsified (Red), or an `exists-trace` lemma is verified (Green), Tamarin generates a visual graph. Clicking on the final node of the proof tree will display this graph on the right pane.

### Understanding the Nodes (Boxes)
Every box in the graph represents a **Rule** that was executed.
- The **top half** of the box shows the **Action Facts** (`--[ ]->` part of your rule).
- The **bottom half** shows the rule name and the exact variable instantiations for that specific run.

### Understanding the Edges (Arrows)
Arrows represent the flow of **Facts** (data and state) from one rule to another.
- **Solid Black Arrows**: Linear facts being consumed. Once consumed, they are gone. (e.g., State facts like `State_A1`).
- **Dotted / Dashed Arrows**: Persistent facts. These can be consumed infinitely without disappearing. (e.g., Long-term keys `!Ltk`, or Public Keys `!Pk`).

### Special Built-in Rules in the Graph
You will often see rules in the graph that you didn't write. These are the attacker and system rules:

1. **`Fr( ~x )`**: The Fresh rule. It simply generates a new, unique random number (like a nonce or a key) and provides it to your rule.
2. **`K( x )` or `!KU( x )`**: The **Knowledge** rules. This is the Attacker! 
   - When you see an arrow pointing *from* your rule (`Out`) *to* a `!KU` node, it means the attacker has intercepted your message.
   - When you see an arrow pointing *from* a `!KU` node *to* your rule (`In`), it means the attacker is injecting a message into the network for your agent to receive.
3. **`c_...` rules (e.g., `c_senc`, `c_h1`)**: These show the attacker performing cryptographic operations. If the attacker knows `A` and `B`, the graph will show a `c_kdf` node where the attacker computes `kdf(A, B)`.

---

## 5. Pro Tip: Following the Attacker's Logic

If you have a failing `Key_Secrecy` lemma, look at the graph and locate the `K( ~SK )` node at the very bottom (this means the attacker learned the session key).
1. Trace the arrows backward from the `K` node.
2. Ask yourself: *"How did the attacker compute this?"*
3. You will see the attacker combining intercepted `Out` messages using `c_` rules.
4. This visual trace tells you exactly which missing protection allowed the attacker to reconstruct the key (e.g., nonces sent in clear text).
