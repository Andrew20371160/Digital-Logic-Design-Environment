# Digital-Logic-Design-Environment

# 0) Concepts & data model (what the code implements)

## 0.1 Gate graph structure (tree + sibling lists)
A circuit is represented using `gate` nodes linked as:

- **Sibling list** via `gate::next` and `gate::prev`
- **Parent/children** via `gate::parent` and `gate::children` (where `children` is the head of a sibling list)

At the top level, `graph::root` itself can be a sibling list: multiple “major trees” can exist at the same level (independent outputs).

## 0.2 Input sources to a gate
A `gate` can receive input from:
1) A **leaf input array** (`bool* input`, size `input_size`)
2) **Children outputs** (each child gate has `output`)
3) **Wired inputs** (`wire_input`, a list of pointers to other gates)

The header comment states an important invariant:
- if a gate gets children or wires as an input source, the leaf boolean array is deleted via `resize_input(0)`.

## 0.3 Wiring system
Wiring is stored as two adjacency lists:
- If gate A drives gate B (A → B):
  - A contains B in `wire_output`
  - B contains A in `wire_input`

---

# 1) Common helpers & metadata (I/O, validation, names)

These functions are not “graph editing” or “evaluation” but are used across those features.

## 1.1 `template<typename DataType> void get_input(DataType& input)`
**Purpose:** Repeatedly read a value from `std::cin` until it succeeds.

**How it works:**
- Loops while input is invalid.
- Reads with `std::cin >> input`.
- If `cin.fail()`:
  - clears state (`cin.clear()`)
  - discards rest of current line (`cin.ignore(max, '\n')`)
  - repeats.

**Used by:** menu selection, numeric inputs, reading strings for logic/test/save/load.

---

## 1.2 `bool is_valid_gate(const uint16_t& g_type, const uint32_t& in_size)`
**Purpose:** Validate a gate type and its requested input size.

**Behavior in code:**
```cpp
return (g_type >= NOT && g_type <= XOR && in_size >= 0);
```
Since `in_size` is unsigned (`uint32_t`), the `in_size >= 0` test is always true. Practically it checks only that `g_type` is within enum range.

**Used by:** `gate::get_gate`, `gate::append_right`, `gate::append_child`, `graph::insert`, `graph::edit`.

---

## 1.3 File open helpers (cpp-local)

### `bool openFileForWriting(const std::string& filePath)`
**Purpose:** Quick check whether a file can be opened for writing.

**Behavior:**
- Creates `std::ofstream file(filePath);`
- Returns `false` if the stream is not good, else `true`.

**Used by:** `graph::save()`.

### `bool openFileForReading(const std::string& filePath)`
**Purpose:** Quick check whether a file can be opened for reading.

**Behavior:**
- Creates `std::ifstream file(filePath);`
- Returns `false` if the stream is not good, else `true`.

**Used by:** `graph::load()`.

---

## 1.4 Gate names and listing helpers (cpp-local)

### `const uint16_t gate_count = 8;`
**Purpose:** Number of gate types.

### `const string gates[8] = {"NOT","BUFFER","NAND","AND","NOR","OR","XNOR","XOR"};`
**Purpose:** Map gate enum value → printable string.

### `void print_gates(uint16_t start = 0)`
**Purpose:** Print a list of selectable gate types as `(index:name)` lines.

**Behavior:**
- Loops from `start` to `gate_count - 1`
- prints `\n(i:NAME)` using `gates[i]`.

**Used by:** `graph::insert()`, `graph::edit()`.

---

# 2) `gate` class — allocation, constraints, and structure (linkage)

`gate` is the internal node type. The header says “no objects are to be created from this class”; allocation is done via a private allocator.

## 2.1 Allocation / initialization helper

### `gate* gate::get_gate(const uint16_t& g_type, const uint32_t& in_size)` *(private)*
**Purpose:** Allocate a new `gate` and initialize all fields.

**Preconditions:**
- `is_valid_gate(g_type, in_size)` must be true for allocation to occur.

**Effects:**
- `new gate`
- Initializes:
  - `gate_type = g_type`
  - `output = 0`
  - `input = NULL`, `input_size = 0`
  - calls `resize_input(in_size)` (allocates leaf array if `in_size > 0`)
  - `parent = next = prev = children = NULL`
  - `children_count = 0`
  - `wire_input` and `wire_output` are empty lists

**Returns:** pointer to allocated gate or `NULL`.

---

## 2.2 NOT/BUFFER single-input-source constraint

### `bool gate::buffer_not_condition(const gate* ptr)` *(private)*
**Purpose:** Enforce the special constraint: `NOT` and `BUFFER` can only have **one** source of input.

**Behavior:**
- If `ptr` is type `NOT` or `BUFFER`, it returns true only if:
  - `ptr->children == NULL`
  - and `ptr->wire_input.size() == 0`
- For other gate types returns true.
- If `ptr == NULL`, returns false.

**Used by:**
- `gate::append_child()` (checks `this`)
- `gate::append_right()` (checks `parent`)
- `gate::connect_wire()` (checks destination gate)

---

## 2.3 Tree/linkage insertion

### `bool gate::append_right(const uint16_t& g_type, const uint32_t& in_size)` *(public)*
**Purpose:** Insert a newly created gate as the **next sibling** of the current gate.

**Preconditions / constraints:**
- `is_valid_gate(g_type, in_size)` must hold.
- Must satisfy either:
  - `parent == NULL`, or
  - `buffer_not_condition(parent)` is true (prevents adding additional inputs under NOT/BUFFER parents)

**Effects on structure:**
- Allocates `new_gate = get_gate(g_type, in_size)`
- Splices it into the sibling doubly-linked list after `this`
- Sets `new_gate->parent = this->parent`
- If parent exists: `parent->children_count++`

**Returns:** true on successful insertion, false otherwise.

---

### `bool gate::append_child(const uint16_t& g_type, const uint32_t& in_size)` *(public)*
**Purpose:** Add a child gate under the current gate.

**Preconditions / constraints:**
- `is_valid_gate(g_type, in_size)` must hold.
- `buffer_not_condition(this)` must be true.

**Effects:**
- If `children == NULL`:
  - sets `children = get_gate(...)`
  - `children->parent = this`
  - calls `resize_input(0)` on the parent gate (deletes leaf input array)
  - increments `children_count`
- Else appends to the end of existing children list by walking to last child and calling `append_right(...)`.

**Returns:** true on success path, false otherwise.

---

## 2.4 Leaf input allocation / resizing

### `bool gate::resize_input(const uint32_t new_in)` *(public)*
**Purpose:** Allocate or deallocate the leaf input array (`bool* input`) and update `input_size`.

**Behavior:**
- Deletes existing array if present, sets `input = NULL` and `input_size = 0`.
- If gate type is `NOT` or `BUFFER`:
  - if `new_in > 0`: forces `input_size = 1` and allocates one bool set to 0.
- Otherwise:
  - if `new_in > 0`: allocates `new_in` bools, sets all to 0, sets `input_size = new_in`.
- If `new_in == 0`: no array is allocated.

**Returns:** true in the current code path (unsigned comparison `new_in >= 0` is always true).

---

### `bool gate::change_type(const uint16_t& g_type)` *(public, declared only)*
**Purpose (by name/comment):** Change the gate type.

**Status in these files:** Declared in `logic_gates.h`, but **no definition exists in `logic_gates.cpp`**.

---

# 3) Wiring system (gate-level + graph-level)

## 3.1 Gate-level wiring primitives

### `bool gate::connect_wire(gate*& src_input)` *(private)*
**Purpose:** Wire `src_input` output into the caller gate’s input sources.

**Constraints:**
- `src_input` must be non-null.
- Destination (`this`) must satisfy `buffer_not_condition(this)`.

**Effects:**
- Calls `resize_input(0)` on destination (removes leaf input array if it exists).
- Adds bidirectional bookkeeping:
  - destination: `wire_input.push_back(src_input)`
  - source: `src_input->wire_output.push_back(this)`

**Returns:** true if connected, else false.

---

### `bool gate::disconnect_wire(gate*& src_input)` *(private)*
**Purpose:** Remove the wire from `src_input` to the caller.

**Effects:**
- Removes `src_input` from `wire_input`.
- Removes `this` from `src_input->wire_output`.

**Returns:** true if `src_input` non-null, else false.

---

### `bool gate::disconnect_wires_in(void)` *(public)*
**Purpose:** Remove all *incoming* wires to this gate.

**Effects (if any exist):**
- For each source gate in `wire_input`, removes `this` from that source’s `wire_output`.
- Clears `wire_input`.

**Returns:** true if there were incoming wires, else false.

---

### `bool gate::disconnect_wires_out(void)` *(public)*
**Purpose:** Remove all *outgoing* wires from this gate.

**Effects (if any exist):**
- For each destination gate in `wire_output`, removes `this` from that destination’s `wire_input`.
- Clears `wire_output`.

**Returns:** true if there were outgoing wires, else false.

---

## 3.2 Graph-level wiring operations (interactive)

### `bool graph::connect(void)` *(public)*
**Purpose:** Interactive wire creation: connect output of one gate to input of another.

**Flow:**
1) sets `traverser = root`
2) prompts “Connect Output of:” and calls `move()` to select source (`from`)
3) prompts “To Input of:” and calls `move()` to select destination
4) calls `destination->connect_wire(from)`

**Returns:** true if both selections succeeded, else false.

---

### `bool graph::disconnect(void)` *(public)*
**Purpose:** Interactive wire removal.

**Flow:**
1) sets `traverser = root`
2) prompts for “from” (output source) via `move()`
3) prompts for “to” (input destination) via `move()`
4) calls `destination->disconnect_wire(from)`

**Returns:** true if both selections succeeded, else false.

---

### `void graph::disconnect_all_wires(void)` *(public)*
**Purpose:** Remove all wires in the entire graph.

**Behavior:**
- For each major tree root:
  - BFS through nodes:
    - calls `disconnect_wires_in()`
    - calls `disconnect_wires_out()`
    - enqueues children

---

# 4) Navigation & visualization (graph traversal + gate printing)

## 4.1 Graph navigation

### `bool graph::move(void)` *(public)*
**Purpose:** Interactive traversal (“remote control”) through the tree/sibling structure.

**Behavior:**
- If `root` is null: returns false.
- Starts at `traverser = root`.
- Repeatedly prints current gate (`traverser->print()`), reads a key via `getch()`, and updates position:
  - `w`: go to `parent` if exists
  - `s`: go to `children` if exists
  - `a`: go to `prev` if exists
  - `d`: go to `next` if exists
  - `r`: jump to `root`
  - `Enter` (`'\r'`): accept, return true
  - `q`: quit, reset `traverser = root`, return false

**Note from code/comments:** Wired connections are not traversed by `move()`; only `children` are used for “down”.

---

## 4.2 Gate printing

### `void gate::print_input_logic(void)` *(public)*
**Purpose:** Print the values feeding this gate.

**Order:**
1) leaf input array values (if `input` exists)
2) each child’s `output` (if children exist)
3) each wired source’s `output` (if `wire_input` non-empty)

---

### `void gate::print_input_sticks(void)` *(public)*
**Purpose:** Print `"| "` once per input source position.

Prints `"| "` for:
- each leaf pin
- each child
- each wire input

---

### `void gate::print_input_gates(void)` *(public)*
**Purpose:** Print gate names corresponding to non-leaf inputs.

Prints:
- spaces for leaf pins
- then each child gate name
- then each wire-input source gate name

---

### `void gate::print(void)` *(public)*
**Purpose:** Print a compact representation of:
- output
- gate name
- current inputs and their origin types/names

It prints:
1) output value
2) `"|"`
3) gate name
4) input logic line (via `print_input_logic`)
5) input gate names line (via `print_input_gates`)

---

# 5) Logic input assignment (setting leaf pins)

## `void graph::set_input(const string& input_logic)` *(public)*
**Purpose:** Assign a string of `'0'`/`'1'` values to leaf input pins, left-to-right.

**Behavior:**
- Initializes `logic_counter = 0`.
- For each major tree root, calls the recursive helper:
  - `set_input(input_logic, logic_counter, root_of_tree)`.

---

## `void graph::set_input(const string& input_logic, uint32_t& logic_counter, gate* ptr)` *(private)*
**Purpose:** Recursive worker for assigning input pins.

**Behavior:**
- If `ptr` has children:
  - recurses into each child (left-to-right by `next`)
- Else (no children):
  - if `ptr->input` exists:
    - fills `ptr->input[i]` from `input_logic[logic_counter]`
    - accepts only `'0'` and `'1'`
    - if a character is not `'0'` or `'1'`, it decrements `i` to retry the same pin, but still advances `logic_counter`
    - if input string runs out, remaining pins are set to 0
  - if there is no input array, nothing is assigned

---

# 6) Evaluation (logic computation)

Evaluation is implemented as a post-order traversal of each major tree. Each gate’s output is computed from any of these sources:
- leaf `input[]`
- children outputs
- wired source outputs

## 6.1 Evaluation driver

### `void graph::evaluate(gate* ptr = NULL)` *(private)*
**Purpose:** Evaluate outputs in one tree using post-order traversal.

**Behavior:**
- If `ptr` is null, sets `ptr = root`.
- Recursively evaluates all children first.
- Dispatches evaluation based on `ptr->gate_type`:
  - AND/NAND → `evaluate_and_nand`
  - OR/NOR → `evaluate_or_nor`
  - XOR/XNOR → `evaluate_xor_xnor`
  - otherwise (NOT/BUFFER) → `evaluate_buffer_not`

---

### `void graph::view_logic(void)` *(public)*
**Purpose:** Evaluate every major tree and then open the interactive viewer.

**Behavior:**
- For each major root gate:
  - `evaluate(ptr)`
- Then `move()`.

---

### `void graph::evaluate(void)` *(public, declared only)*
**Declared in header:** “evaluate logic for the whole graph”  
**Status in these files:** No definition present in `logic_gates.cpp`.

---

## 6.2 Gate-type evaluation helpers

### `void graph::evaluate_and_nand(gate* ptr)` *(private)*
**Purpose:** Evaluate AND or NAND behavior.

**Behavior:**
- Checks for any `0` across:
  - leaf inputs
  - children outputs
  - wire input outputs
- If gate type is AND:
  - output = NOT(found_zero)
- Else (NAND):
  - output = found_zero

---

### `void graph::evaluate_or_nor(gate* ptr)` *(private)*
**Purpose:** Evaluate OR or NOR behavior.

**Behavior:**
- Checks for any `1` across leaf inputs, children outputs, wire input outputs.
- Then uses:
  ```cpp
  if(ptr->gate_type = OR) output = found_one;
  else output = !found_one;
  ```
  That is exactly what the code contains (assignment in the condition).

---

### `void graph::evaluate_xor_xnor(gate* ptr)` *(private)*
**Purpose:** Evaluate XOR or XNOR behavior.

**Behavior:**
- Counts number of ones across all sources.
- Sets output to parity:
  - XOR: `ones_counter & 1`
  - XNOR: `!(ones_counter & 1)`

---

### `void graph::evaluate_buffer_not(gate* ptr)` *(private)*
**Purpose:** Evaluate BUFFER or NOT behavior using a single source value.

**Behavior (in code order):**
- If leaf input exists: sets output from `input[0]` or its negation.
- If children exist: overwrites output based on the first child’s output.
- If wired input exists: overwrites output based on the first wired source’s output (`front()`).

---

# 7) File persistence & component wiring reconstruction

Persistence is split into:
- Saving/loading the **structure** (gate types and children counts)
- Saving/loading the **wiring** (from-path → to-path mapping)

## 7.1 Path mapping (addressing gates in files)

### `string graph::get_path(gate* wanted)` *(private)*
**Purpose:** Compute a path string from `root` to a gate.

**Encoding used in this implementation:**
- `'r'`: step across siblings (the algorithm uses `prev` to count how far from head)
- `'c'`: step down to children

**Behavior:**
- If wanted is root, returns `""`.
- Otherwise builds a string by iteratively:
  - aligning to a “level”
  - counting sibling steps
  - stepping into children

(Implemented as an infinite `while(1)` loop that returns when it reaches the wanted gate.)

---

### `void graph::get_path(void)` *(public)*
**Purpose:** Interactive helper: select a gate via `move()` and print its path.

---

## 7.2 Wiring serialization

### `string graph::get_wiring(void)` *(public)*
**Purpose:** Serialize every wiring connection in the entire graph.

**Output format:**
- Each line: `<from_path>:<to_path>\n`

**Algorithm:**
- For each major tree:
  - BFS through nodes
  - For each node with any `wire_output` destinations:
    - compute `from_path = get_path(node)`
    - append one line per destination

---

## 7.3 Wiring reconstruction helper (gate-level)

### `void gate::wire_system(string wiring)` *(private)*
**Purpose:** Apply wiring instructions to the currently loaded component.

**Input format:**
- Repeated: `<from_path>:<to_path>\n`

**Behavior:**
- Parses strings into `from` and `to`
- Resolves pointers with `get_to(from)` and `get_to(to)`
- Calls `to_ptr->connect_wire(from_ptr)` when both are valid

---

## 7.4 Save

### `void graph::save(void)` *(public)*
**Purpose:** Save each major tree to indexed `.txt` files and save wiring separately.

**Files written:**
- `<base>0.txt`, `<base>1.txt`, ... (one file per major tree)
- `<base>wiring.txt` (wiring mapping)

**Tree serialization format (each line):**
```
gate_type : (children_count or input_size) : (not_leaf or leaf)
```

**Traversal:** Breadth-first (BFS).

---

## 7.5 Load

### `void graph::load(void)` *(public)*
**Purpose:** Load a multi-tree component and optionally wire it, then append it into the current graph.

**Input expectation:**
- User provides base file path (without index and extension)
- Code attempts `<base>0.txt`, `<base>1.txt`, ... until open fails
- Optional `<base>wiring.txt`

**Reconstruction approach:**
- Reads first line as root of a major tree.
- Uses a queue for BFS reconstruction based on stored `children_count`.
- After building the component:
  - applies wiring via `component->wire_system(wiring)` if wiring file exists
- If current graph empty:
  - sets `root = component`
- Otherwise:
  - user selects insertion location and chooses to append:
    - to the right, or
    - as a child, or
    - cancel (deletes loaded component via `remove_all`)

---

# 8) Graph editing, deletion, and component insertion helpers

## 8.1 Interactive edit features (user-facing)

### `void graph::edit(void)`
**Purpose:** Edit a selected gate.

**Operations:**
- Edit input size (only if no children and no wire inputs)
- Edit gate type (only for gate types > BUFFER; assigns directly)

---

### `void graph::insert(void)`
**Purpose:** Insert a new gate by selecting a target and inserting:
- right sibling, or
- child gate

---

### `bool graph::remove(void)`
**Purpose:** Remove selected gate after confirmation, including:
- its descendants
- all wiring attached to any deleted node

---

## 8.2 Graph deletion helpers

### `void graph::remove_links(gate* component)` *(private)*
**Purpose:** Detach node from:
- root list
- parent child list
- sibling list
and decrement parent `children_count` when applicable.

---

### `void graph::remove_gate(gate* component)` *(private)*
**Purpose:** BFS delete a node and its subtree.

**Also does:**
- disconnect wires in/out on every deleted node
- delete leaf input arrays

---

### `void graph::remove_graph(void)`
**Purpose:** Remove entire board.

---

## 8.3 Component insertion helpers (used by load)

### `uint32_t graph::count_list(gate* head)` *(private)*
Counts nodes in a sibling list via `next`.

### `bool graph::assign_parent(gate* head, gate* parent)` *(private)*
Sets `parent` pointer for each node in a component list.

### `bool graph::append_child(gate* component)` *(private)*
Appends a loaded component list as children under `traverser`, updates `children_count`, and deletes leaf input array of parent (`resize_input(0)`).

### `bool graph::append_right(gate* component)` *(private)*
Splices a loaded component list as siblings to the right of `traverser`, updates parent `children_count` if applicable.

---

# 9) Gate cleanup helper used during load-cancel

### `void gate::remove_all(gate*& root)` *(private)*
**Purpose:** Delete an entire component list (multiple major trees) when canceling load insertion.

**Behavior:**
- While `root` exists:
  - BFS delete the current major tree rooted at `root`
  - advances `root = root->next`

Deletes leaf arrays and nodes.

---
