# Digital Logic Design Environment

A graph-based digital logic design environment written in **C++** that enables you to **build, simulate, and evaluate** digital circuits using common logic gates, hierarchical gate composition, and an automated wiring + persistence system.

> Demo/usage playlist: https://www.youtube.com/playlist?list=PLbDnyrrfgzH6rByhw83TfR-hEWR1vLolN

---

## Features

- **Logic gate modeling**: NOT, BUFFER, NAND, AND, NOR, OR, XNOR, XOR  
- **Graph/tree-based circuit design**
  - Gates can have **children** (hierarchical composition)
  - Gates can have **siblings** (linked-list style at the same level)
- **Wiring system**
  - Connect the **output of any gate** to the **input of another** (beyond the tree structure)
  - Connect / disconnect wires and clear all wiring
- **Circuit evaluation**
  - Evaluates circuits using **post-order traversal** (children first, then parents)
- **Persistence (save/load)**
  - Save circuits/components to files
  - Reload components and re-apply wiring using a **path notation system**
- **Interactive navigation UI**
  - Move around the circuit using keyboard controls and view logic

---

## Project Structure

This project is primarily implemented in:

- `logic_gates.h` — class definitions + declarations
- `logic_gates.cpp` — full implementation (gate logic, graph operations, save/load, wiring, UI loop)

Additional docs:
- `DOCUMENTATION.MD` — full detailed documentation
- `DEMO.md` — short overview + demo link
- `features.readme` — feature list
- `Updates.md` — development log / series notes

---

## Core Concepts

### Gate types

```cpp
enum {
    NOT = 0,
    BUFFER,
    NAND,
    AND,
    NOR,
    OR,
    XNOR,
    XOR
};
```

### Circuit structure (tree + wiring)

- The main circuit is an **n-ary tree**
  - Each gate may have multiple children
  - Children are stored as a linked list (siblings connected by `next` / `prev`)
- A separate **wire system** provides cross-links:
  - `wire_input`: gates feeding this gate
  - `wire_output`: gates receiving this gate’s output

### Path notation (for save/load + wiring)

Paths are strings used to locate a gate from the root:

- `c` = go to **child**
- `r` = go to **right sibling**
- `""` (empty string) = root

Example: `crrc` = child → right → right → child

---

## User Interface (Main Menu)

The program exposes an interactive menu:

1. Insert gate  
2. Connect (wire gates)  
3. Disconnect (remove wire)  
4. Edit gate properties  
5. Remove all wiring  
6. Resize input pins for all leaf gates  
7. Test logic (set inputs and evaluate)  
8. Remove selected gate  
9. Remove entire circuit  
10. Save circuit to files  
11. Load circuit from files  
12. Navigate circuit (move command)  
13. Quit  

### Navigation controls (move command)

- `w` parent
- `s` child
- `a` previous sibling
- `d` next sibling
- `r` return to root
- `enter` select current gate
- `q` quit navigation

---

## Important Constraints

### NOT and BUFFER gates (single input source rule)

NOT and BUFFER gates must have **exactly one** input source and cannot mix sources:

- ONE input pin **OR**
- ONE child gate **OR**
- ONE wired connection  
…but not a combination.

This prevents ambiguous/invalid gate behavior.

---

## File Format (Persistence)

### Circuit files (`component0.txt`, `component1.txt`, ...)

Each line:

`GATE_TYPE:IN_SIZE_OR_CHILD_COUNT:LEAF_STATUS`

- `gate_enum`: 0–7 (NOT..XOR)
- `size`: input pin count (leaf) or children count (non-leaf)
- `leaf_status`: `1` leaf (uses input pins), `0` non-leaf (uses children)

### Wiring file (`componentwiring.txt`)

Each line:

`from_path:to_path`

Where both sides are path strings (using the `c`/`r` notation).

---

## Documentation

If you want the full deep-dive (classes, algorithms, save/load, wiring, UI details), read:

- `DOCUMENTATION.MD`

---

## Status / Notes

Per `Updates.md`, the project reached a point where building very complex systems through a CLI-only interface becomes difficult without a GUI; however, it remains useful for experimenting with gate networks, building smaller reusable components, and learning how logic simulation, wiring, and persistence can be implemented.

---
