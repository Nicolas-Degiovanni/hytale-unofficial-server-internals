---
description: Architectural reference for LogicCondition
---

# LogicCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient Data Model

## Definition
```java
// Signature
public class LogicCondition extends Condition {
```

## Architecture & Concepts
The LogicCondition class is a fundamental component of the server-side conditional logic system, which governs everything from entity behavior to status effect application. It embodies the **Composite Design Pattern**, allowing for the construction of complex, tree-like logical expressions from simpler parts.

A LogicCondition is a specialized type of Condition that does not perform a direct check itself. Instead, its sole purpose is to aggregate an array of other Condition objects and evaluate them based on a specified logical operator, either AND or OR. This recursive structure enables developers and content creators to define sophisticated rule sets in a declarative way.

This class is designed to be data-driven. Its state is almost exclusively loaded from external game asset files (e.g., JSON definitions for items, abilities, or NPC behaviors) via the Hytale **Codec** system. The static CODEC field is the serialization contract that maps asset data to an in-memory LogicCondition instance, making it a critical bridge between game data and executable server logic.

Internally, the evaluation is delegated to the nested Operator enum, which implements a **Strategy Pattern** to provide distinct evaluation logic for AND and OR operations.

## Lifecycle & Ownership
-   **Creation:** Instances are overwhelmingly created by the Hytale Codec framework during server startup or asset loading. The framework reads a structured data file, finds a section corresponding to a LogicCondition, and uses the static CODEC to construct the object and its children recursively. Manual instantiation via its constructor is rare and generally reserved for unit testing or dynamic, in-memory logic generation.

-   **Scope:** A LogicCondition is a stateless configuration object. Its lifetime is bound to the containing asset that defines it. For example, a LogicCondition defined within a specific monster's AI package will exist in memory as long as that monster definition is loaded. It does not persist across sessions or hold any mutable game state.

-   **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup once the root asset object that references them is unloaded from memory. There are no explicit `destroy` or `close` methods.

## Internal State & Concurrency
-   **State:** **Effectively Immutable**. The internal fields, `operator` and `conditions`, are populated once during deserialization by the codec. While not declared as `final`, they are not intended to be modified after construction. The class acts as a static blueprint for a logical evaluation.

-   **Thread Safety:** **Thread-safe for evaluation**. The primary method, `eval`, is a pure function with no side effects. It operates solely on its immutable internal state and the provided arguments. Multiple threads can safely call `eval` on a shared LogicCondition instance without locks or synchronization, provided the arguments (such as the ComponentAccessor) are handled correctly by the calling context.

## API Surface
The public API is inherited from the parent Condition class. The core logic is implemented in the protected `eval0` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(accessor, ref, time) | boolean | O(N) | Evaluates the nested conditions. N is the number of child conditions. The operation short-circuits, resulting in O(1) complexity in the best case (e.g., the first condition of an OR is true). |

## Integration Patterns

### Standard Usage
A developer or system will almost never interact with LogicCondition directly. Instead, it will be part of a larger structure retrieved from an asset definition. The calling code interacts with the abstract Condition interface, unaware of the specific composite implementation.

```java
// Assume 'someAsset' has a 'condition' field defined in its data file
// The system loads the asset, and the codec populates the field.
Condition condition = someAsset.getCondition();

// The system evaluates the condition without needing to know its concrete type.
// If 'condition' is a LogicCondition, this call will trigger its composite evaluation.
boolean result = condition.eval(entityAccessor, entityRef, Instant.now());

if (result) {
    // Apply game logic
}
```

### Anti-Patterns (Do NOT do this)
-   **State Modification:** Do not attempt to get and modify the internal `conditions` array after the object has been constructed. This violates its intended immutability and can lead to unpredictable behavior across the server.
-   **Deep Nesting Performance:** While the iterative evaluation prevents `StackOverflowError`, creating excessively deep trees of LogicConditions (e.g., hundreds of levels) can negatively impact performance during evaluation. Favor broader, flatter structures where possible.
-   **Direct Instantiation:** Avoid using `new LogicCondition(...)` in general game logic. Conditions should be defined as data in asset files. This decouples game logic from game data and allows for content to be changed without recompiling code.

## Data Pipeline
The primary flow for LogicCondition is from a static data definition into an executable in-memory object that returns a boolean result.

> Flow:
> Game Asset File (e.g., JSON) -> Hytale Codec System -> **LogicCondition Instance** -> Game System (e.g., EntityStatsManager) -> `condition.eval()` -> Boolean Result

