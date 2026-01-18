---
description: Architectural reference for the Condition abstract base class.
---

# Condition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Polymorphic Strategy Component

## Definition
```java
// Signature
public abstract class Condition {
```

## Architecture & Concepts

The Condition class is an abstract base class that forms the core of a data-driven logic evaluation system within the server's entity statistics module. It represents a single, stateless, evaluatable rule that can be combined with other rules to control complex game behaviors, such as health regeneration, status effect application, or AI decision-making.

This system is built on two fundamental design patterns:

1.  **Strategy Pattern:** Condition defines a common interface for a family of algorithms. Each concrete subclass (e.g., a hypothetical IsDaytimeCondition or IsOnFireCondition) provides a specific implementation for the evaluation logic. This allows game logic to operate on the abstract Condition type without needing to know the specifics of each rule.

2.  **Template Method Pattern:** The public `eval` method provides the skeleton of the evaluation algorithm. It implements the shared inversion logic (checking the *inverse* field) and then defers the actual condition-specific evaluation to the abstract `eval0` method, which must be implemented by subclasses.

The most critical architectural feature is its integration with the Hytale Codec system. The static `CODEC` field, a CodecMapCodec, acts as a central registry. This allows the game engine to dynamically deserialize entire arrays of different, concrete Condition types directly from asset files (e.g., JSON or HOCON). This is the cornerstone of Hytale's data-driven design, enabling designers to create and chain complex logical rules without writing any new Java code.

### Lifecycle & Ownership

-   **Creation:** Condition instances are almost never created directly in code using the `new` keyword. They are instantiated by the Hytale Codec framework during the server's asset loading phase. The engine reads a definition from a data file, uses the `CODEC` registry to find the appropriate subclass, and constructs the object.
-   **Scope:** The lifetime of a Condition instance is tied to its owning asset. For example, a set of conditions defined in a monster's asset file will remain in memory as long as that monster definition is loaded. They are effectively immutable configuration objects.
-   **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup when the asset that defined them is unloaded from memory. There is no manual destruction or cleanup required.

## Internal State & Concurrency

-   **State:** Instances of Condition and its subclasses must be considered **immutable** after deserialization. The `inverse` field is set at creation time and should not change. The `eval` method is purely computational and stateless; its result depends solely on the arguments provided at the time of the call.

-   **Thread Safety:** This class is inherently **thread-safe**. Its immutable nature and lack of side effects in the evaluation methods allow a single Condition instance to be safely evaluated by multiple threads concurrently. This is essential in a multi-threaded server environment where different world regions or entities may be processed in parallel.

    **WARNING:** Subclass implementations must uphold this contract. A concrete Condition that introduces mutable state or side effects will break thread safety and lead to unpredictable behavior.

## API Surface

The primary public contract is for evaluating collections of conditions. Direct interaction with single instances is rare.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(accessor, ref, time) | boolean | O(1) | Public entry point for evaluation. Implements inversion logic and delegates to the concrete `eval0` implementation. |
| allConditionsMet(...) | static boolean | O(N) | **Primary Usage.** Iterates an array of Conditions, returning true only if all evaluate to true. Short-circuits on the first failure. |

## Integration Patterns

### Standard Usage

Developers should not interact with individual Condition instances. The system is designed to be consumed through the static `allConditionsMet` helper method, typically by a higher-level game system that holds a reference to a pre-loaded array of conditions.

```java
// Example from a hypothetical stat regeneration system
public void processRegeneration(ComponentAccessor<EntityStore> accessor, Ref<EntityStore> entityRef, Instant time, EntityStatType.Regenerating statInfo) {
    // The correct pattern is to use the static helper on the full condition set
    boolean canRegenerate = Condition.allConditionsMet(
        accessor,
        entityRef,
        time,
        statInfo.getConditions()
    );

    if (canRegenerate) {
        // Proceed with stat regeneration logic
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new MyCustomCondition()`. This bypasses the data-driven asset pipeline and creates tightly-coupled logic that is difficult to maintain. All conditions should be defined in data files.
-   **Mutable Subclasses:** Implementing a subclass of Condition with mutable internal state is a severe violation of the architectural contract. It breaks thread safety and can cause difficult-to-diagnose bugs related to race conditions.
-   **Side Effects in eval0:** The `eval0` implementation must be a pure function. It should not modify the entity, the world, or its own internal state. Its only purpose is to return a boolean result based on the provided inputs.

## Data Pipeline

The Condition class is a key component in the data flow from configuration assets to live game logic.

> Flow:
> Game Asset File (e.g., monster.json) -> Hytale Codec Deserializer -> In-Memory `Condition[]` Array (held by an asset object) -> Game System (e.g., EntityStatSystem) -> **Condition.allConditionsMet** -> Boolean Result -> Game Logic Execution

