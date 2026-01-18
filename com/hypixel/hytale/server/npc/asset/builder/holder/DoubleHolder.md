---
description: Architectural reference for DoubleHolder
---

# DoubleHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient Data Holder

## Definition
```java
// Signature
public class DoubleHolder extends DoubleHolderBase {
```

## Architecture & Concepts
The DoubleHolder is a specialized component within the server-side NPC asset processing pipeline. Its primary function is to represent a floating-point number that is not a static literal but is instead derived from a dynamic, context-sensitive expression.

This class acts as a bridge between static NPC asset definitions (e.g., JSON or HOCON files) and the dynamic server runtime. It encapsulates the logic for resolving a value based on the current game state, provided via an ExecutionContext, and subsequently validating that value against a set of predefined relational constraints.

It is a fundamental building block for creating complex and state-aware NPC behaviors, allowing designers to define properties like attack damage or movement speed as expressions (e.g., "base_speed * level_modifier") rather than fixed numbers.

### Lifecycle & Ownership
-   **Creation:** DoubleHolder instances are not meant to be created directly. They are instantiated by a deserialization framework or a factory during the parsing of an NPC asset file. They form a part of the larger NPC definition object graph.
-   **Scope:** The lifetime of a DoubleHolder is strictly tied to its parent NPC asset definition. It persists as long as the template or definition is held in memory.
-   **Destruction:** The object is marked for garbage collection when the parent NPC asset definition is unloaded or discarded. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
-   **State:** A DoubleHolder instance is stateful. The underlying expression and relational constraints are held in the parent DoubleHolderBase and are considered immutable after the initial asset loading and deserialization process. The resolution of the value, however, is volatile and depends entirely on the provided ExecutionContext.
-   **Thread Safety:** **This class is not thread-safe.** The methods are re-entrant, but their safety is wholly dependent on the thread safety of the passed ExecutionContext and the underlying game state it reads from. Concurrent calls with different, isolated contexts are safe. However, if the ExecutionContext or the data it references can be mutated by other threads, external synchronization is mandatory to prevent data races and inconsistent value resolution.

## API Surface
The public API is minimal, focusing exclusively on value retrieval and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(context) | void | O(N) | Triggers the full get-and-validate sequence. Throws an exception if relational constraints are violated. |
| get(executionContext) | double | O(N) | Resolves the underlying expression and validates the result. Returns the final double value. Throws on validation failure. |

**Complexity Note:** The complexity is O(N), where N is the combined cost of evaluating the underlying expression and checking all relational constraints. It is not a constant-time operation.

## Integration Patterns

### Standard Usage
The DoubleHolder should be treated as a read-only component of a fully constructed NPC asset. Game logic systems retrieve the holder and use it to compute a value at the moment it is needed.

```java
// Assume 'npcDefinition' is the loaded asset object
// Assume 'currentContext' is the up-to-date execution context for the NPC

DoubleHolder speedModifier = npcDefinition.getMovement().getSpeedModifier();
double currentSpeed = speedModifier.get(currentContext);

// Use the resolved 'currentSpeed' in the game logic
npc.applySpeed(currentSpeed);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new DoubleHolder()`. The object is invalid unless its internal state is populated by the asset loading system. Always retrieve it from a parent asset definition.
-   **Ignoring Context:** Calling `get` with a stale or incorrect ExecutionContext will produce invalid results that do not reflect the current game state.
-   **Excessive Re-evaluation:** Do not call `get` repeatedly within a tight loop for a value that is not expected to change. If the underlying expression is complex, cache the result in a local variable for the duration of the current tick or operation.

## Data Pipeline
The DoubleHolder functions as a transformation and validation step in the data flow from game state to game logic.

> Flow:
> NPC Asset File -> Deserializer -> **DoubleHolder Instance** -> Game Logic requests value with `ExecutionContext` -> **DoubleHolder** resolves expression and validates -> `double` value returned -> Used by NPC Behavior System

