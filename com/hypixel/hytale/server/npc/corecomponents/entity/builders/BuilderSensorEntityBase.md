---
description: Architectural reference for BuilderSensorEntityBase
---

# BuilderSensorEntityBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient / Abstract Base Class

## Definition
```java
// Signature
public abstract class BuilderSensorEntityBase extends BuilderSensorWithEntityFilters {
```

## Architecture & Concepts
BuilderSensorEntityBase is an abstract base class that serves as the foundational component for constructing entity-aware **Sensors** within the Hytale NPC AI framework. It is not a runtime component; rather, it is a configuration-time object responsible for parsing a JSON definition and preparing the necessary data to instantiate a functional Sensor.

This class acts as the architectural bridge between a designer-authored JSON configuration file and the live, in-game NPC perception system. It standardizes the core properties common to all sensors that detect entities, such as detection range, target locking behavior, and advanced filtering logic.

Key architectural features include:

*   **Configuration-Driven:** The primary entry point is the readConfig method, which consumes a JsonElement. This design decouples NPC behavior logic from compiled Java code, allowing for rapid iteration by designers.
*   **Dynamic Properties:** The class utilizes a Holder pattern (e.g., DoubleHolder, BooleanHolder) for its properties. This allows values like *range* to be defined not just as static numbers, but as dynamic expressions that are evaluated at runtime against the NPC's current state via an ExecutionContext.
*   **Pluggable Logic (Strategy Pattern):** It defines extension points for custom entity selection logic through the ISensorEntityPrioritiser and ISensorEntityCollector interfaces. This allows designers to inject complex behaviors, such as prioritizing the lowest-health target or applying a status effect to all detected entities, without modifying the core sensor implementation.

## Lifecycle & Ownership
The lifecycle of a BuilderSensorEntityBase instance is ephemeral and strictly confined to the server's asset loading phase.

*   **Creation:** An instance of a concrete subclass is created by the server's BuilderManager when it encounters a corresponding sensor definition within an NPC's JSON configuration file.
*   **Scope:** The object exists only for the duration of parsing and validating a single sensor definition. Its internal state is populated by the readConfig method and subsequently used to construct a runtime Sensor object.
*   **Destruction:** The builder object is eligible for garbage collection immediately after the corresponding runtime Sensor has been instantiated. It holds no persistent state and is not referenced by any live game objects.

**WARNING:** These builder objects are not recycled. A new instance is created for every unique sensor definition processed during server startup or asset reload.

## Internal State & Concurrency
*   **State:** The internal state is highly mutable. During the call to readConfig, its fields (range, lockOnTarget, etc.) are populated from the source JSON. This state is intended to be written once during initialization and then read multiple times by the framework to construct the final Sensor.

*   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, configured, and used within a single, synchronous asset-loading thread. There are no internal locks or synchronization mechanisms.

**WARNING:** Concurrent access to a BuilderSensorEntityBase instance will lead to unpredictable behavior and data corruption. The entire NPC asset pipeline must be treated as a single-threaded process.

## API Surface
The public API is primarily for consumption by concrete subclasses and the internal builder framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(data) | Builder<Sensor> | O(N) | Parses the JSON configuration. N is the number of properties and filters. This is the primary initialization method. |
| validate(...) | boolean | O(N) | Recursively validates the parsed configuration and all sub-components, such as filters and prioritisers. |
| getRange(support) | double | O(E) | Resolves and returns the maximum detection range. Complexity depends on the backing expression (E). |
| getPrioritiser(support) | ISensorEntityPrioritiser | O(1) | Constructs and returns the configured entity prioritiser. Returns a default instance if none is specified. |
| getCollector(support) | ISensorEntityCollector | O(1) | Constructs and returns the configured entity collector. Returns a default instance if none is specified. |
| getLockedTargetSlot(support) | int | O(E) | Resolves the string-based target slot name (e.g., "LockedTarget") into a runtime integer ID. |

## Integration Patterns

### Standard Usage
This abstract class is meant to be extended. A concrete implementation defines a specific type of entity sensor (e.g., a Player sensor) and may add its own unique properties. The framework handles the lifecycle.

```java
// Example of a concrete subclass (conceptual)
public class BuilderSensorPlayer extends BuilderSensorEntityBase {
    @Override
    public Builder<Sensor> readConfig(@Nonnull JsonElement data) {
        // 1. Configure all base properties
        super.readConfig(data);

        // 2. Read player-specific properties from JSON
        // this.readPlayerSpecificConfig(data);

        return this;
    }

    // ... methods to build the actual SensorPlayer instance
}
```

### Anti-Patterns (Do NOT do this)
*   **Direct Instantiation:** Never instantiate a subclass of this builder directly. The asset loading framework is responsible for its creation and management.
*   **State Re-use:** Do not attempt to reuse a builder instance to configure a second sensor. The internal state is not designed to be reset and will contain stale data.
*   **Access Before Init:** Calling methods like getRange before readConfig has been successfully executed will result in uninitialized state and likely throw exceptions.

## Data Pipeline
BuilderSensorEntityBase operates within a configuration-time data pipeline, transforming declarative JSON into executable game logic.

> **Flow:**
> NPC JSON File -> Server Asset Parser -> **BuilderSensorEntityBase (Instance)** -> Runtime Sensor Component -> Live NPC Entity

