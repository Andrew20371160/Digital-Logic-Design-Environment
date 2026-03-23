## Overview

The **Digital Logic Design Environment** is an interactive C++ application that simulates a virtual digital logic circuit board. It allows users to create, configure, and evaluate complex digital circuits composed of various logic gates (NOT, BUFFER, NAND, AND, NOR, OR, XNOR, XOR) with support for hierarchical gate structures, wiring systems, and persistent storage.

---

# Table of Contents

1. [Overview](#overview)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
   - 3.1 [Gate Class](#1-gate-class)
     - 3.1.1 [Key Attributes](#key-attributes)
     - 3.1.2 [Gate Types Enum](#gate-types-enum)
     - 3.1.3 [Key Gate Methods](#key-gate-methods)
   - 3.2 [Graph Class](#2-graph-class)
     - 3.2.1 [Key Attributes](#key-attributes-1)
     - 3.2.2 [Path Notation System](#path-notation-system)
     - 3.2.3 [Core Graph Methods](#core-graph-methods)
4. [Logic Gate Implementations](#logic-gate-implementations)
   - 4.1 [AND / NAND Gates](#and--nand-gates)
   - 4.2 [OR / NOR Gates](#or--nor-gates)
   - 4.3 [XOR / XNOR Gates](#xor--xnor-gates)
   - 4.4 [NOT / BUFFER Gates](#not--buffer-gates)
5. [Data Structure Organization](#data-structure-organization)
   - 5.1 [Gate Tree Structure](#gate-tree-structure)
   - 5.2 [Wiring System](#wiring-system)
6. [File Format Specification](#file-format-specification)
   - 6.1 [Circuit File Format](#circuit-file-format)
   - 6.2 [Wiring File Format](#wiring-file-format)
7. [User Interface](#user-interface)
   - 7.1 [Main Menu](#main-menu)
   - 7.2 [Gate Navigation Controls](#gate-navigation-controls)
   - 7.3 [Visualization](#visualization)
8. [Helper Functions](#helper-functions)
   - 8.1 [Input Handling](#input-handling)
   - 8.2 [File Operations](#file-operations)
   - 8.3 [Gate Utilities](#gate-utilities)
9. [Algorithm Highlights](#algorithm-highlights)
   - 9.1 [Circuit Evaluation (Post-Order Traversal)](#circuit-evaluation-post-order-traversal)
   - 9.2 [Path Finding](#path-finding)
   - 9.3 [SaveLoad Mechanism](#saveload-mechanism)
10. [Example Usage: Building a Half Adder](#example-usage-building-a-half-adder)
11. [Key Design Patterns](#key-design-patterns)
    - 11.1 [Tree with Linked List Siblings](#1-tree-with-linked-list-siblings)
    - 11.2 [Dual Input Sources](#2-dual-input-sources)
    - 11.3 [Path-Based Automation](#3-path-based-automation)
    - 11.4 [Post-Order Traversal for Evaluation](#4-post-order-traversal-for-evaluation)
12. [Special Constraints and Rules](#special-constraints--rules)
    - 12.1 [NOT and BUFFER Gates](#not-and-buffer-gates)
    - 12.2 [Gate Insertion](#gate-insertion)
    - 12.3 [Input Assignment](#input-assignment)
13. [Error Handling](#error-handling)
14. [Memory Management](#memory-management)
15. [Main Application Loop](#main-application-loop)
# Digital Logic Design Environment - Complete Documentation

## Project Structure

The project consists of two main files:
- **logic_gates.h** - Header file with class definitions and function declarations
- **logic_gates.cpp** - Implementation file with complete logic gate and graph functionality

---

## Core Components

### 1. Gate Class

The `gate` class represents an individual logic gate in the circuit. It is a foundational building block that handles:
- Gate type and output logic value
- Input management (direct input pins or child gates)
- Hierarchical relationships (parent-child connections)
- Wiring system for connecting gates together

#### Key Attributes:

```cpp
uint16_t gate_type;              // Enum value: NOT, BUFFER, NAND, AND, NOR, OR, XNOR, XOR
bool output;                     // Single output value
bool *input;                     // Array of input pins for leaf gates
uint32_t input_size;            // Number of input pins

gate *parent;                    // Parent gate pointer
gate *next, *prev;              // Sibling gate pointers
gate *children;                 // First child gate pointer
uint32_t children_count;        // Count of child gates

list<gate*> wire_input;         // Gates providing input via wiring
list<gate*> wire_output;        // Gates receiving output via wiring
```

#### Gate Types Enum:
```cpp
enum {
    NOT = 0,    // NOT gate (1 input)
    BUFFER,     // BUFFER gate (1 input, passes through)
    NAND,       // NAND gate (2+ inputs)
    AND,        // AND gate (2+ inputs)
    NOR,        // NOR gate (2+ inputs)
    OR,         // OR gate (2+ inputs)
    XNOR,       // XNOR gate (2+ inputs)
    XOR         // XOR gate (2+ inputs)
};
```

#### Key Gate Methods:

**Memory Management:**
- `gate* get_gate()` - Allocates memory for a new gate with specified type and input size
- `void remove_all(gate*&)` - Deallocates entire gate tree using BFS

**Structural Operations:**
- `bool append_right()` - Adds a sibling gate to the right
- `bool append_child()` - Adds a child gate (converts leaf to parent)
- `bool resize_input()` - Changes input size, with special handling for NOT/BUFFER gates
- `bool buffer_not_condition()` - Validates NOT/BUFFER constraint (only one input source)

**Wiring System:**
- `bool connect_wire()` - Connects another gate's output to this gate's input
- `bool disconnect_wire()` - Removes a wired connection
- `bool disconnect_wires_in()` - Disconnects all input wires
- `bool disconnect_wires_out()` - Disconnects all output wires
- `gate* first_no_input()` - Finds first gate with no input source
- `list<gate*> all_no_input()` - Returns all gates with no input source
- `gate* get_to()` - Navigates to gate using path string

**Printing & Visualization:**
- `void print_input_logic()` - Displays input values
- `void print_input_sticks()` - Shows input structure
- `void print_input_gates()` - Displays input gate names
- `void print()` - Comprehensive gate visualization

---

### 2. Graph Class

The `graph` class represents the entire digital logic circuit board. It manages the overall structure and provides user interface operations.

#### Key Attributes:

```cpp
gate *root;              // Root gate(s) of the circuit tree
mutable gate *traverser; // Current selected gate (for navigation)
```

#### Path Notation System:

The graph uses a path notation system for automation:
- **'r'** = Move to right sibling (next)
- **'c'** = Move to child (down)
- **Empty string** = Root gate
- Example: `"crrc"` means: down → right → right → down

#### Core Graph Methods:

**Initialization & Cleanup:**
- `graph()` - Default constructor
- `~graph()` - Destructor
- `void remove_graph()` - Removes all gates

**Navigation & Selection:**
- `bool move()` - Interactive gate traversal using WASD controls
  - **w**: Go to parent
  - **s**: Go to child
  - **a**: Go to previous sibling
  - **d**: Go to next sibling
  - **r**: Return to root
  - **enter**: Select gate
  - **q**: Quit navigation

**Gate Management:**
- `void insert()` - Add new gate to circuit
- `bool remove()` - Remove selected gate and its children
- `void remove_links()` - Update tree pointers after removal
- `void remove_gate()` - Comprehensive gate deletion with wire cleanup

**Input & Logic Evaluation:**
- `void set_input()` - Set input values from string (0s and 1s)
- `void evaluate()` - Post-order traversal to compute gate outputs
- `void view_logic()` - Navigate and view logic values

**Logic Gate Evaluation Functions:**
- `void evaluate_and_nand()` - Evaluates AND/NAND logic
- `void evaluate_or_nor()` - Evaluates OR/NOR logic
- `void evaluate_xor_xnor()` - Evaluates XOR/XNOR logic
- `void evaluate_buffer_not()` - Evaluates BUFFER/NOT logic

**Wiring Operations:**
- `bool connect()` - Interactive wire connection between gates
- `bool disconnect()` - Interactive wire disconnection
- `void disconnect_all_wires()` - Removes all wiring in circuit

**File I/O & Persistence:**
- `void save()` - Saves circuit to files (indexed by tree)
- `void load()` - Loads circuit from files and auto-wires
- `string get_path()` - Returns path string to gate
- `string get_wiring()` - Returns all wiring connections
- `void wire_system()` - Applies wiring from loaded files

**Component Management:**
- `uint32_t count_list()` - Counts gates in a list
- `bool assign_parent()` - Sets parent pointer for gate list
- `bool append_child()` - Appends component as children
- `bool append_right()` - Appends component as siblings

**Editing:**
- `void edit()` - Modify selected gate's properties
- `void resize_all_input()` - Resize input for all leaf gates

---

## Logic Gate Implementations

### AND / NAND Gates

**AND Logic:**
- Output = 1 if ALL inputs are 1
- Output = 0 if ANY input is 0

**NAND Logic:**
- Inverted AND (NOT AND)
- Output = 0 if ALL inputs are 1
- Output = 1 if ANY input is 0

**Inputs:**
- Direct pins (boolean array)
- Child gate outputs
- Wired gate outputs

### OR / NOR Gates

**OR Logic:**
- Output = 1 if ANY input is 1
- Output = 0 if ALL inputs are 0

**NOR Logic:**
- Inverted OR (NOT OR)
- Output = 0 if ANY input is 1
- Output = 1 if ALL inputs are 0

### XOR / XNOR Gates

**XOR Logic:**
- Output = 1 if odd number of inputs are 1
- Uses bitwise AND with 1 to check parity

**XNOR Logic:**
- Inverted XOR
- Output = 1 if even number of inputs are 1

### NOT / BUFFER Gates

**NOT Logic:**
- Output = inverted input
- Restricted to exactly 1 input source
- Cannot have mixed input types

**BUFFER Logic:**
- Output = input (no change)
- Restricted to exactly 1 input source
- Pass-through gate

**Special Constraint:**
```
NOT and BUFFER gates can only have:
- ONE input pin, OR
- ONE child gate, OR
- ONE wired connection

But NOT a combination of multiple sources
```

---

## Data Structure Organization

### Gate Tree Structure

The circuit is organized as a **n-ary tree** where:
- Each gate can have multiple children
- Children are stored as a linked list
- Siblings are connected via `next` and `prev` pointers
- Each child has a parent pointer

```
Example Circuit:
      root1          root2
       |              |
      AND            XOR
      /|\             |
     / | \           NOT
   OR NAND BUFFER     |
```

### Wiring System

Gates can be connected via a wiring system:
- `wire_input`: List of gates providing input via wires
- `wire_output`: List of gates receiving output via wires
- Forms additional connections beyond the tree hierarchy

```
Example:
  OR --wire--> AND
  (output)     (input)
```

---

## File Format Specification

### Circuit File Format (component0.txt, component1.txt, etc.)

Each line represents a gate: `GATE_TYPE:IN_SIZE_OR_CH_COUNT:LEAF_STATUS`

```
Format: [gate_enum]:[size]:[leaf_status]\n

Where:
- gate_enum: 0-7 representing gate type (NOT=0, BUFFER=1, etc.)
- size: Input size (for leaf) or children count (for non-leaf)
- leaf_status: 0 = not_leaf (has children), 1 = leaf (has input pins)

Example (Half Adder):
3:2:1    (AND gate with 2 inputs, is leaf)
2:2:1    (NAND gate with 2 inputs, is leaf)
5:2:0    (OR gate with 2 children, is not leaf)
```

### Wiring File Format (componentwiring.txt)

```
Format: [from_path]:[to_path]\n

Where:
- from_path: Path string to source gate's output
- to_path: Path string to destination gate's input

Example:
c:cc     (Root's child output wired to root's child's first child's input)
cr:      (Root's child's sibling output wired to root)
```

---

## User Interface

### Main Menu

```
1  - Insert gate
2  - Connect (wire gates)
3  - Disconnect (remove wire)
4  - Edit gate properties
5  - Remove all wiring
6  - Resize input pins for all leaf gates
7  - Test logic (set inputs and evaluate)
8  - Remove selected gate
9  - Remove entire circuit
10 - Save circuit to files
11 - Load circuit from files
12 - Navigate circuit (move command)
13 - Quit
```

### Gate Navigation Controls (move command)

```
w - Move up to parent gate
s - Move down to first child gate
a - Move left to previous sibling
d - Move right to next sibling
r - Jump to root gate
enter - Select current gate
q - Exit navigation
```

### Visualization

When navigating, gates display:
```
[output_value]
        |
   [gate_name]
   
[input_logic]
     |  |  |
[gates/inputs]
```

---

## Helper Functions

### Input Handling

```cpp
template<typename DataType>
void get_input(DataType& input)
```
- Handles user input with error checking
- Clears invalid input from stream
- Loops until valid input received

### File Operations

```cpp
bool openFileForWriting(const std::string& filePath)
bool openFileForReading(const std::string& filePath)
```
- Validates file open operations
- Returns success/failure status

### Gate Utilities

```cpp
bool is_valid_gate(const uint16_t& g_type, const uint32_t& in_size)
void print_gates(uint16_t start)
```
- Validates gate type and input size
- Prints available gate options

---

## Algorithm Highlights

### Circuit Evaluation (Post-Order Traversal)

1. Visit all children recursively
2. Evaluate children first
3. Evaluate current gate using:
   - Input pins (if leaf)
   - Children outputs (if parent)
   - Wired inputs (if connected)

### Path Finding

Uses iterative approach to find path from root to any gate:
1. Move up to determine tree level
2. Count right movements at each level
3. Return path string of 'r' (right) and 'c' (child) characters

### Save/Load Mechanism

**Save:**
1. BFS traversal of each tree
2. Save gate type, input/children count, and leaf status
3. Separate wiring file with connection mappings

**Load:**
1. Read gate info from file
2. Reconstruct tree structure
3. Apply wiring from separate file
4. Prompt user for placement in existing circuit

---

## Example Usage: Building a Half Adder

```cpp
// Create a new circuit
graph circuit;

// Insert first XOR gate (sum)
circuit.insert();  // Choose XOR, 2 inputs

// Insert AND gate (carry)
circuit.insert();  // Choose AND, 2 inputs

// Set inputs to "11"
circuit.set_input("11");

// Evaluate logic
circuit.evaluate();

// View results
circuit.view_logic();

// Save for reuse
circuit.save();  // Save as "half_adder"
```

---

## Key Design Patterns

### 1. Tree with Linked List Siblings
- Combines hierarchical tree with sibling connections
- Allows both parent-child and sibling relationships

### 2. Dual Input Sources
- Gates can receive input from:
  - Direct input pins (for leaf nodes)
  - Child gate outputs (for parent nodes)
  - Wired connections (from other gates)

### 3. Path-Based Automation
- Uses string paths for saving/loading wiring
- Enables serialization of complex circuits

### 4. Post-Order Traversal for Evaluation
- Ensures children evaluated before parents
- Maintains logical correctness

---

## Special Constraints & Rules

### NOT and BUFFER Gates

```
✗ Cannot mix input sources
✓ Exactly ONE of:
  - Input pins
  - Child gate
  - Wired connection
```

### Gate Insertion

```
If root is NULL:
  → Create root gate

If root exists:
  → User navigates to position
  → Choose append direction:
     (1) As sibling to the right
     (2) As child
```

### Input Assignment

Input values are assigned left-to-right to leaf gates:
- Read string of '0' and '1' characters
- Skip non-binary characters
- Fill remaining inputs with 0s if insufficient

---

## Error Handling

- **Invalid gate type**: Rejected by `is_valid_gate()`
- **File not found**: Graceful return from file operations
- **Invalid input**: Re-prompts user
- **Constraint violations**: Operations return false on failure

---

## Memory Management

- Uses `new`/`delete` for gate allocation
- BFS-based cleanup in `remove_all()` and `remove_gate()`
- Proper pointer cleanup when converting leaf to parent gates
- Dynamic array resizing for input pins

---

## Main Application Loop

```cpp
int main() {
    graph board;
    
    while(true) {
        // Display menu
        // Get user choice
        // Execute corresponding operation
        // Continue until quit (choice 13)
    }
}
```

The application provides an interactive environment for designing and testing digital logic circuits with a persistent storage system.
