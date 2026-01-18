---
description: Architectural reference for BuilderSensorWithEntityFilters
---

# BuilderSensorWithEntityFilters

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Abstract Builder

## Definition
```java
// Signature
public abstract class BuilderSensorWithEntityFilters extends BuilderSensorBase {
```

## Architecture & Concepts
BuilderSensorWithEntityFilters is an abstract base class that serves as a foundational component within the server-side NPC asset compilation pipeline. It is not a runtime component but rather a part of the asset loading and validation system.

Its primary role is to provide a standardized mechanism for any NPC *Sensor Builder* that needs to process and construct lists of entity filters. In the Hytale AI framework, a Sensor is a component that grants an NPC the ability to perceive its environment—for example, to detect nearby players, enemies, or specific block types. Filters are declarative rules that refine a sensor's perception, allowing for complex logic such as "find all players on my team but ignore those who are currently invisible."

This class acts as the bridge between the declarative NPC configuration files (e.g., JSON or HOCON) and the imperative, runtime `IEntityFilter` objects. It encapsulates the complex logic for parsing, validating, and ultimately building the filter chains that will be executed by the live Sensor component in the game world.

## Lifecycle & Ownership
-   **Creation:** An instance of a concrete subclass (e.g., `BuilderSensorNearbyEntities`) is created by the NPC asset loading system when it parses a sensor definition within an NPC's configuration file. This abstract class is never instantiated directly.
-   **Scope:** Transient. The lifetime of a BuilderSensorWithEntityFilters instance is strictly confined to the asset parsing and validation phase. It holds temporary state parsed from configuration and is discarded once the final runtime component is built. It does not exist during the main game loop.
-   **Destruction:** After the `build` method of a concrete subclass is called (which in turn calls `getFilters`), the resulting runtime `IEntityFilter` array is attached to the final Sensor component. The builder object has then served its purpose and is eligible for garbage collection.

## Internal State & Concurrency
-   **State:** The internal state is mutable and designed for a single, linear build process. The core state is held within the `filters` field, a `BuilderObjectListHelper` which accumulates the definitions of each filter from the source asset. This state is progressively built during parsing and is consumed during the final build phase.

-   **Thread Safety:** This class is **not thread-safe**. The entire NPC asset loading pipeline is designed to be a single-threaded process. Accessing a builder instance from multiple threads will result in race conditions, corrupted state, and unpredictable validation or build failures. All interactions must be confined to the main asset loading thread.

## API Surface
The public API is intended for consumption by concrete subclasses and the asset building framework, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(...) | boolean | O(N) | Recursively validates this sensor and its associated list of N filters. This is a critical step in the asset loading process to provide early feedback on configuration errors. |
| getFilters(...) | IEntityFilter[] | O(N) | Constructs and returns an array of concrete, runtime `IEntityFilter` instances from the parsed configuration. This is the terminal operation for the filter-building part of the lifecycle. |

## Integration Patterns

### Standard Usage
A developer never interacts with this class directly. Instead, they extend it to create a new type of sensor builder that requires entity filtering capabilities.

```java
// Example of a concrete subclass
public class BuilderSensorNearbyPlayers extends BuilderSensorWithEntityFilters {

    // ... other sensor-specific fields and logic ...

    @Override
    public ISensor build(BuilderSupport support) {
        ComponentContext context = createComponentContext();
        
        // Use the inherited method to build the filter chain
        IEntityFilter[] builtFilters = getFilters(support, null, context);
        
        // Pass the filters to the runtime sensor component
        return new SensorNearbyPlayers(this.radius, builtFilters);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Runtime Reference:** Never attempt to hold a reference to this builder or its subclasses in a runtime game component. These are build-time objects only. Storing them will cause memory leaks and prevent assets from being properly unloaded.
-   **State Reuse:** Do not call `getFilters` multiple times on the same builder instance. The builder's internal state is designed for a single, idempotent build operation. Reusing it can lead to unexpected or duplicated filter instances.
-   **Manual Validation:** Do not call `validate` manually. The asset loading framework is responsible for orchestrating the validation lifecycle at the correct time and with the correct context.

## Data Pipeline
This class is a key transformation step in the data pipeline that converts static configuration into live game objects.

> Flow:
> NPC JSON Definition -> Asset Parser -> **BuilderSensorWithEntityFilters (subclass)** -> `getFilters()` -> `IEntityFilter[]` Array -> Runtime Sensor Component -> Live NPC Entity

