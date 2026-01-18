---
description: Architectural reference for StdScope
---

# StdScope

**Package:** com.hypixel.hytale.server.npc.util.expression
**Type:** Stateful Component

## Definition
```java
// Signature
public class StdScope implements Scope {
```

## Architecture & Concepts
StdScope is the standard implementation of the Scope interface, forming the contextual backbone of the Hytale expression evaluation engine. It functions as a symbol table that maps string identifiers to typed values, functions, constants, and variables.

The core architectural pattern is a **hierarchical, chained scope model**. Each StdScope instance can be linked to a parent Scope. When a symbol lookup is requested, the current scope is checked first. If the symbol is not found, the request is delegated up the chain to the parent scope. This enables the creation of nested execution contexts, such as a global server scope, a per-world scope, and a temporary per-NPC or per-expression scope, where inner scopes can access and override symbols from outer scopes.

A critical design choice is the use of functional suppliers (e.g., DoubleSupplier, Supplier<String>) to represent symbol values. This decouples the expression engine from the concrete state of game objects. Instead of storing a static value like an NPC's health, the Scope stores a *supplier* that, when invoked, retrieves the current health. This facilitates **lazy evaluation** and ensures that expressions always operate on the most up-to-date game state without requiring manual state synchronization.

For performance, StdScope pre-allocates and reuses static instances for common, immutable values like true, false, null, and empty strings/arrays. This flyweight pattern reduces object churn during intensive expression evaluation.

## Lifecycle & Ownership
- **Creation:** A StdScope is created directly via its constructor, typically by the system responsible for initiating an expression evaluation, such as a script runner or an AI behavior tree. A parent scope can be provided during construction to establish the scope chain. Static factory methods like copyOf are also provided for cloning existing scopes.

- **Scope:** The lifetime of a StdScope instance is explicitly tied to the context it represents. A root-level scope may persist for the entire server session, while a local scope created for a single function evaluation is short-lived and should be discarded after use.

- **Destruction:** The object is managed by the Java garbage collector. There are no explicit cleanup or disposal methods. Ownership is determined by standard Java reference rules; once a scope is no longer referenced (e.g., after an expression is evaluated), it becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** StdScope is a highly mutable, stateful component. Its primary internal state is the symbolTable, a HashMap that stores the symbols defined within its local scope. Methods like addVar and changeValue directly modify this internal map.

- **Thread Safety:** **This class is not thread-safe.** The internal symbolTable is a non-synchronized HashMap. Concurrent access from multiple threads, especially write operations, will lead to unpredictable behavior, data corruption, or ConcurrentModificationExceptions.

    **WARNING:** A single StdScope instance must be confined to a single thread of execution. If state needs to be shared across threads, it must be done through external synchronization mechanisms or by creating thread-local copies of the scope.

## API Surface
The public API is designed for populating the scope and retrieving value suppliers for the expression evaluator.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addConst(name, value) | void | O(1) | Adds an immutable constant to the symbol table. Throws IllegalStateException if the symbol already exists. |
| addVar(name, value) | void | O(1) | Adds a mutable variable to the symbol table. Throws IllegalStateException if the symbol already exists. |
| addSupplier(name, supplier) | void | O(1) | Binds a symbol to a supplier for dynamic, lazy-evaluated values. |
| changeValue(name, value) | void | O(1) | Updates the value of an existing variable. Throws IllegalStateException if the symbol is a constant or does not exist. |
| get...Supplier(name) | Supplier | O(d) | Retrieves the value supplier for a symbol. Traverses the parent chain, where *d* is the depth of the scope. |
| merge(other) | StdScope | O(n) | Merges symbols from another scope into this one. *n* is the number of symbols in the other scope. |
| isConstant(name) | boolean | O(d) | Checks if a symbol is defined as a constant. Traverses the parent chain. |

## Integration Patterns

### Standard Usage
The primary pattern involves creating a hierarchy of scopes. A base scope is configured with global functions and constants. Child scopes are then created for specific contexts, populated with local variables, and passed to the expression evaluator.

```java
// 1. Create a global scope with server-wide constants
StdScope globalScope = new StdScope(null);
globalScope.addConst("MAX_PLAYERS", 100.0);

// 2. For a specific NPC, create a child scope
StdScope npcScope = new StdScope(globalScope);

// 3. Populate with dynamic, state-driven variables using suppliers
npcScope.addSupplier("self.health", () -> npc.getHealth());
npcScope.addSupplier("target.distance", () -> npc.getDistanceToTarget());

// 4. The expression engine uses this fully-formed scope for evaluation
Expression expr = Expression.compile("self.health > 50 && target.distance < 10");
boolean result = expr.evaluateBoolean(npcScope);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not share and modify a single StdScope instance across multiple threads without external locking. This will corrupt its internal state.
- **State Leakage:** Do not reuse a temporary, mutable scope for multiple, unrelated expression evaluations. A previous evaluation might have changed a variable's value, causing unexpected behavior in the next. Always create a fresh, clean scope for each distinct execution context.
- **Modifying Constants:** Do not attempt to use changeValue on a symbol declared with addConst. This will result in an IllegalStateException.
- **Shadowing without Intent:** Be aware that adding a variable to a child scope with the same name as a variable in a parent scope will "shadow" the parent's variable. This is a powerful feature but can cause bugs if done unintentionally.

## Data Pipeline
StdScope acts as a bridge, allowing the expression engine to query game state without being directly coupled to it.

> Flow:
> Game State (e.g., NPC Health) -> `addSupplier` -> **StdScope Symbol Table** -> `get...Supplier` -> Expression Node Evaluation -> Result

