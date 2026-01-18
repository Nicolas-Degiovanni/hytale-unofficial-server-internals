---
description: Architectural reference for StatsConditionBaseInteraction
---

# StatsConditionBaseInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class StatsConditionBaseInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts

The StatsConditionBaseInteraction class is an abstract foundation within the server's **Interaction System**. It serves as a data-driven conditional check, designed to gate gameplay interactions based on an entity's statistical resources, such as health, mana, or stamina. This class embodies the "Condition" phase of a standard Trigger-Condition-Action (TCA) gameplay pattern.

Its primary architectural role is to decouple the logic of stat-checking from the specific effects of an interaction. This is achieved through two key mechanisms:

1.  **Data-Driven Configuration:** The class is designed to be instantiated and configured via the Hytale **Codec** system. All properties, such as the required resource costs (`Costs`), the comparison logic (`LessThan`), and value interpretation (`ValueType`), are defined in external asset files (e.g., JSON). This allows designers to create complex stat-based interactions without writing new code.

2.  **Template Method Pattern:** It provides a skeletal algorithm in the `firstRun` method, which orchestrates the validation process. The crucial step of this algorithm, the actual resource check, is deferred to subclasses through the abstract `canAfford` method. This forces concrete implementations to provide specific logic for querying entity stats while the base class handles the generic flow control and failure state management.

Upon deserialization, it performs a critical optimization by resolving string-based stat names from the configuration (`rawCosts`) into a more performant integer-keyed map (`costs`) for rapid lookups during gameplay.

## Lifecycle & Ownership

-   **Creation:** Instances of StatsConditionBaseInteraction subclasses are not created directly using the *new* keyword. They are materialized by the Hytale **Codec** system during the server's asset loading phase. The static `CODEC` field defines the deserialization contract, which is invoked by a higher-level service like an AssetManager or InteractionModule when parsing game configuration files.

-   **Scope:** An instance's lifetime is tied to the loaded game assets. It is effectively a static data object once loaded, persisting for the entire server session or until a hot-reload of assets occurs.

-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server shuts down or its associated asset configuration is unloaded. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency

-   **State:** The internal state is configured entirely at load time and is **effectively immutable** during gameplay. Fields like `rawCosts`, `lessThan`, and `lenient` are populated by the Codec and are not intended to be modified at runtime. The `costs` field is a derivative cache of `rawCosts`, populated once in the `afterDecode` lifecycle hook for performance.

-   **Thread Safety:** This class is **conditionally thread-safe**. Because its internal state is immutable after initialization, it is safe to read its configuration from any thread. However, its primary methods, `firstRun` and `canAfford`, operate on mutable, thread-hostile game state objects like `InteractionContext` and `EntityStore`.

    **WARNING:** All method invocations that interact with the game world must be synchronized by the calling context, which is typically the main server game loop or a dedicated entity processing thread. Calling these methods from an uncontrolled, asynchronous context will lead to race conditions and world corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | The primary entry point for the interaction check. It orchestrates the call to `canAfford` and sets the interaction state to `Failed` if the condition is not met. N is the number of stats to check. |
| canAfford(entityRef, componentAccessor) | abstract boolean | O(N) | Abstract method that must be implemented by subclasses. Contains the core logic for comparing the entity's current stats against the configured costs. |

## Integration Patterns

### Standard Usage

This class is not used directly but is extended to create concrete interaction conditions. A game designer would then reference the concrete implementation in an asset file. The engine invokes it automatically.

A concrete implementation would look like this:

```java
// 1. A concrete subclass provides the canAfford logic.
public class MyStatCheckInteraction extends StatsConditionBaseInteraction {
    // The CODEC for this class would also be defined here.

    @Override
    protected boolean canAfford(@Nonnull Ref<EntityStore> entity, @Nonnull ComponentAccessor<EntityStore> accessor) {
        // Implementation to get stats from the entity and compare
        // against the 'costs' map from the base class.
        EntityStatsModule stats = accessor.getModule(entity, EntityStatsModule.class);
        // ... logic to check stats ...
        return true; // or false
    }
}

// 2. The system invokes the interaction during gameplay.
// This code is internal to the InteractionModule.
InteractionContext context = ...;
StatsConditionBaseInteraction interaction = context.getInteraction(); // Loaded from assets

// The engine calls firstRun, which triggers the canAfford check.
interaction.firstRun(type, context, cooldowns);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never instantiate a subclass of StatsConditionBaseInteraction programmatically with `new`. Doing so bypasses the entire data-driven configuration system, leaving critical fields like `costs` uninitialized and resulting in NullPointerExceptions. All instances must be created via the Codec system.

-   **Runtime State Mutation:** Do not attempt to modify the configuration fields (`costs`, `lenient`, `lessThan`) after the object has been loaded. This violates its design as an immutable configuration object and will cause unpredictable and inconsistent behavior across the server.

-   **Asynchronous Execution:** Do not call `firstRun` or `canAfford` from a separate thread without ensuring exclusive access to the `EntityStore` and `CommandBuffer` being passed in the context. This will cause severe concurrency issues.

## Data Pipeline

The flow of data from configuration to in-game effect follows a clear, one-way path.

> Flow:
> Game Asset File (JSON/HOCON) -> Hytale Codec System -> **Instance of StatsConditionBaseInteraction Subclass** -> InteractionModule Execution -> `firstRun()` -> `canAfford()` Check -> InteractionState is set to `Failed` OR Interaction proceeds to the next step.

