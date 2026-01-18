---
description: Architectural reference for Scope
---

# Scope

**Package:** com.hypixel.hytale.server.npc.util.expression
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface Scope {
```

## Architecture & Concepts
The Scope interface defines a contract for a runtime environment within which expressions are evaluated. It is a fundamental component of the server-side expression language, likely used for NPC behaviors, quest triggers, and other dynamic game logic.

Architecturally, Scope acts as a bridge between the abstract expression evaluation engine and the concrete game state. It provides a standardized mechanism for the evaluator to query the values of variables and invoke functions without having any direct knowledge of game objects like NPCs, players, or world properties.

This decouples the expression language from the game engine. The core principle is lazy evaluation, enforced by the use of functional interfaces like Supplier, DoubleSupplier, and BooleanSupplier. This ensures that when an expression like *npc.health > 50* is evaluated, the value of *npc.health* is fetched from the game world at the precise moment of evaluation, not when the expression is parsed or compiled.

## Lifecycle & Ownership
As an interface, Scope itself has no lifecycle. The following pertains to its concrete implementations.

- **Creation:** A Scope implementation is created on-demand by a system that needs to execute a script or expression. For example, an NPC Behavior Tree system would construct a Scope populated with references to that specific NPC's state just before evaluating a logic node.
- **Scope:** Transient and short-lived. A Scope object typically exists only for the duration of a single, complete expression evaluation. It is considered ephemeral context.
- **Destruction:** The Scope object is eligible for garbage collection as soon as the expression evaluation completes and all references to it are released. There is no manual destruction or cleanup method defined in the contract.

## Internal State & Concurrency
- **State:** The interface is stateless. However, any concrete implementation is inherently stateful, as it holds references or keys to access live, mutable game world data. The use of Suppliers means the Scope does not cache values; it provides a live view into the game state.
- **Thread Safety:** **Not thread-safe.** Implementations of Scope are expected to be created, used, and discarded within a single thread, almost certainly the main server tick thread. Accessing a Scope from a worker thread would lead to severe concurrency issues, as it reads live game data without any synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStringSupplier(String name) | Supplier<String> | O(1) | Returns a supplier for a string variable. Throws if the variable is not found or is of a different type. |
| getNumberSupplier(String name) | DoubleSupplier | O(1) | Returns a supplier for a numeric variable. Throws if the variable is not found or is of a different type. |
| getBooleanSupplier(String name) | BooleanSupplier | O(1) | Returns a supplier for a boolean variable. Throws if the variable is not found or is of a different type. |
| getFunction(String name) | Scope.Function | O(1) | Retrieves a callable function from the environment by its name. |
| isConstant(String name) | boolean | O(1) | Returns true if the variable's value is guaranteed not to change during the expression's evaluation. Used by the compiler for optimization. |
| getType(String name) | ValueType | O(1) | Returns the type of the variable or null if it does not exist. Used for type checking. |
| encodeFunctionName(name, types) | static String | O(N) | A static utility to create a mangled function signature, embedding parameter types into the name. |

## Integration Patterns

### Standard Usage
A Scope implementation is typically provided to an Expression or Script object just before execution. The evaluator then uses the Scope to resolve all symbols found in the expression tree.

```java
// A system (e.g., NPC AI) creates a scope for a specific context
Scope npcScope = new NpcRuntimeScope(npc);

// The expression is evaluated using this scope
Expression healthCheck = expressionCache.get("npc.health > 50");
boolean isHealthy = healthCheck.evaluateBoolean(npcScope);
```

### Anti-Patterns (Do NOT do this)
- **Caching Supplier Results:** Do not call a supplier once and store its result for later use. The fundamental design of Scope relies on suppliers to fetch live data at the moment of need. Caching the value defeats this purpose and will lead to bugs based on stale data.
- **Cross-Thread Access:** Never pass a Scope object to another thread. The underlying game state it accesses is not thread-safe. All evaluations must occur on the primary game thread.
- **Long-Lived Scopes:** Do not hold references to Scope objects beyond the immediate execution of an expression. They are designed to be lightweight, temporary objects. Retaining them can inadvertently prevent game objects from being garbage collected.

## Data Pipeline
The Scope is a critical component in the data flow from the game engine to the expression evaluator.

> Flow:
> Expression Evaluator requests variable "npc.health" -> **Scope** implementation receives request -> Implementation accesses live NPC object field -> A DoubleSupplier is returned to the Evaluator -> Evaluator invokes getAsDouble() -> Live health value is returned

