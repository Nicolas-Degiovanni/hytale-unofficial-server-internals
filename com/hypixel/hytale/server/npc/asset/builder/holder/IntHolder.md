---
description: Architectural reference for IntHolder
---

# IntHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Component

## Definition
```java
// Signature
public class IntHolder extends ValueHolder {
```

## Architecture & Concepts
The IntHolder is a specialized component within the NPC asset building framework, designed to represent an integer value that can be either static or dynamically evaluated at runtime. It serves as a crucial abstraction layer, decoupling the definition of an NPC's attributes from their concrete values during gameplay.

During the server's asset loading phase, NPC definitions are parsed from JSON files. For every integer-based property, such as health, damage, or count, an IntHolder instance is created. This allows game designers to define attributes as simple constants (e.g., `100`) or as complex, context-sensitive formulas (e.g., `world.difficulty * 10`).

The core of its architecture is a two-stage validation system:
1.  **Intrinsic Validation:** An IntValidator is used to check the value against simple, self-contained rules (e.g., must be positive, must be within a specific range). This validation is applied as soon as the value is resolved.
2.  **Relational Validation:** A list of relation validators allows for complex, cross-field consistency checks. These validators are injected by the parent asset builder and are executed with the full ExecutionContext, enabling rules like "the number of minions must not exceed the leader's level".

This component is fundamental to creating data-driven and flexible NPC behaviors without requiring code changes.

## Lifecycle & Ownership
-   **Creation:** An IntHolder is never instantiated directly. It is created and configured exclusively by a higher-level asset builder during the parsing of a JSON configuration file via the readJSON method. This method acts as the primary initialization entry point.
-   **Scope:** The lifecycle of an IntHolder is tightly bound to the lifecycle of the parent NPC asset it belongs to. It persists in memory for as long as the NPC asset definition is loaded by the server.
-   **Destruction:** The object is marked for garbage collection when its parent NPC asset is unloaded or the server shuts down. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The IntHolder is a stateful component. Its internal state, including the expression to be evaluated and its list of validators, is configured once during the asset loading phase. After this initialization, its configuration is considered frozen and should not be modified. The value returned by the get method is transient and depends on the ExecutionContext provided at runtime.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be configured on a single thread during server bootstrap. Subsequent reads via the get method are safe to call from the main game loop thread, provided no concurrent modifications are attempted.

    **WARNING:** Modifying an IntHolder by calling methods like addRelationValidator after the initial asset loading phase is complete will introduce severe race conditions and system instability. All configuration must occur synchronously during the server's asset parsing stage.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(N) | Initializes the holder from a JSON element. N is the complexity of the expression. This is the primary configuration entry point. |
| get(ExecutionContext) | int | O(E) | Resolves the expression, runs all validators, and returns the final integer. E is the evaluation complexity of the underlying expression. |
| rawGet(ExecutionContext) | int | O(E) | Resolves the expression and runs only the intrinsic validator. Skips relational checks. Use with extreme caution. |
| addRelationValidator(...) | void | O(1) | Appends a cross-field validator. **WARNING:** Must only be called during the initial asset build process. |

## Integration Patterns

### Standard Usage
The IntHolder is intended to be used by asset builders to construct a complete, validated NPC template. Game logic then retrieves values from the holder at runtime using a specific context.

```java
// During asset loading (Builder's responsibility)
IntHolder health = new IntHolder();
IntValidator healthValidator = IntValidator.range(1, 10000);
health.readJSON(npcJson.get("health"), 100, healthValidator, "health", builderParams);

// During gameplay (Game Logic's responsibility)
ExecutionContext context = createNpcContext(npc);
int currentMaxHealth = health.get(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new IntHolder()` for any purpose other than immediate initialization via `readJSON`. An uninitialized holder will throw NullPointerExceptions.
-   **Late-Stage Modification:** Never call `addRelationValidator` on an IntHolder that is part of a fully loaded and active NPC asset. This is a critical concurrency violation.
-   **Ignoring the Context:** The `get` method requires an `ExecutionContext`. Passing a null or incomplete context may lead to incorrect calculations for any dynamic expressions.
-   **Overuse of rawGet:** The `rawGet` method bypasses important relational validation. Using it can lead to logically inconsistent state within an NPC. It should only be used in performance-critical code where the relational constraints are guaranteed to be met by other means.

## Data Pipeline
The IntHolder processes data from a static configuration file into a context-aware value used by the live game engine.

> Flow:
> JSON Asset File -> Gson Parser -> **IntHolder.readJSON()** -> Expression & Validators configured -> Game Logic provides ExecutionContext -> **IntHolder.get()** -> Expression Evaluated -> Intrinsic & Relational Validation -> Final Integer Value

