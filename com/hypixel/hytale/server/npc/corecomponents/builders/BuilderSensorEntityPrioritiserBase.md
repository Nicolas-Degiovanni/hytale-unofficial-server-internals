---
description: Architectural reference for BuilderSensorEntityPrioritiserBase
---

# BuilderSensorEntityPrioritiserBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class BuilderSensorEntityPrioritiserBase extends BuilderBase<ISensorEntityPrioritiser> {
```

## Architecture & Concepts
BuilderSensorEntityPrioritiserBase is a foundational component within the server-side NPC asset loading framework. It serves as a mandatory template for all builders that construct an ISensorEntityPrioritiser instance. An ISensorEntityPrioritiser is a runtime component responsible for evaluating a list of entities detected by an NPC's sensors and selecting the most important one based on a specific strategy (e.g., proximity, threat level, health).

This base class is not concerned with the runtime logic of prioritization. Instead, its primary architectural role is to enforce a contract during the **asset validation phase**. It ensures that any concrete prioritiser builder explicitly declares the types of "filters" it supports. This information is then consumed by the NPCLoadTimeValidationHelper to cross-reference against other parts of the NPC configuration, preventing asset misconfigurations before the server even starts.

It acts as a schema and validation bridge between a static NPC definition file and the final, in-memory AI component.

### Lifecycle & Ownership
- **Creation:** Concrete implementations are instantiated by the NPC asset loading system during server startup or when NPC definitions are hot-reloaded. The engine uses reflection or a registry to find the appropriate builder for a given configuration key.
- **Scope:** These builder objects are ephemeral and exist only for the duration of the asset parsing and validation process. They are considered transient, configuration-time objects.
- **Destruction:** Instances are eligible for garbage collection immediately after the corresponding ISensorEntityPrioritiser runtime object has been successfully constructed and the NPC asset is fully loaded. They do not persist into the main game loop.

## Internal State & Concurrency
- **State:** The class holds a single, immutable state field: a Set of strings named providedFilterTypes. This set is populated at construction time via the subclass constructor and cannot be modified thereafter.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, the asset loading pipeline in which it operates is typically a single-threaded, blocking process. Concurrent access is neither expected nor designed for.

## API Surface
The public API is minimal, designed for interaction with the asset loading system, not general game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| category() | Class | O(1) | Returns the ISensorEntityPrioritiser interface. Used by the builder registry for type mapping. |
| isEnabled(context) | boolean | O(1) | Determines if this builder is active. By default, it is always enabled. |
| validate(...) | boolean | O(N) | Injects the declared filter types into the validation helper. Critical for asset integrity checks. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The standard pattern is to **extend** it to create a new type of entity prioritiser. The engine's asset loading system will then automatically discover and utilize the concrete implementation.

```java
// Example of a concrete implementation
public class BuilderPrioritiserByHealth extends BuilderSensorEntityPrioritiserBase {

    // The developer declares the filters this prioritiser provides
    public BuilderPrioritiserByHealth() {
        super(Set.of("LOWEST_HEALTH", "HIGHEST_HEALTH"));
    }

    // The developer implements the logic to build the runtime object
    @Override
    public ISensorEntityPrioritiser build(ExecutionContext context, Scope globalScope) {
        // ... logic to create a new PrioritiserByHealth instance
        return new PrioritiserByHealth(...);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** This class is abstract and cannot be instantiated with `new`. Subclasses should only be instantiated by the engine's asset loading mechanisms.
- **State Mutation:** Do not attempt to modify the provided filter types after construction. This state is intended to be static and defined at compile time.
- **Bypassing Validation:** Calling the `build` method without first running the object through the engine's `validate` pipeline can lead to runtime errors caused by misconfigured NPC assets. The `validate` step is not optional.

## Data Pipeline
This component operates within the NPC asset loading and validation pipeline, not the runtime game loop. Its flow is concerned with configuration data, not real-time game data.

> **Configuration Flow:**
> NPC Definition File (e.g., `mob.json`) -> Asset Deserializer -> **Concrete Builder Instance** -> NPCLoadTimeValidationHelper -> Final `ISensorEntityPrioritiser` Object -> NPC Runtime Asset

