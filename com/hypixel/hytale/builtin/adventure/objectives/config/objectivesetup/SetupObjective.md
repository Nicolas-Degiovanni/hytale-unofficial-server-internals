---
description: Architectural reference for SetupObjective
---

# SetupObjective

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.objectivesetup
**Type:** Transient

## Definition
```java
// Signature
public class SetupObjective extends ObjectiveTypeSetup {
```

## Architecture & Concepts
The SetupObjective class is a concrete implementation of the ObjectiveTypeSetup contract. It functions as a data-driven command object within the Adventure Mode framework. Its sole responsibility is to encapsulate the configuration required to initiate a new objective for a set of players.

This class is not a service or a long-lived component. Instead, it is a deserialized representation of an action defined in an adventure asset file. The Hytale engine's codec system parses game data and instantiates SetupObjective to represent a "start new objective" event. When this event is triggered, the instance's *setup* method is invoked, which then delegates the core logic of objective creation to the central ObjectivePlugin.

Its primary role is to decouple the game's event and trigger systems from the concrete implementation of the objective management system.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during the deserialization of adventure mode assets. The static CODEC field defines the schema for parsing the configuration data into a valid SetupObjective object. Manual instantiation is an anti-pattern.
- **Scope:** The lifecycle of a SetupObjective instance is extremely short and bound to a specific event. It is created, its *setup* method is invoked once, and it is then immediately eligible for garbage collection. It holds no persistent state beyond its initial configuration.
- **Destruction:** Managed by the Java Garbage Collector. There are no native resources or explicit cleanup procedures required.

## Internal State & Concurrency
- **State:** The internal state consists of a single field, objectiveId, which is populated during deserialization. The object is effectively immutable after construction, as there are no public methods to alter its state.
- **Thread Safety:** The object itself is thread-safe due to its immutable nature. However, the methods it exposes, particularly *setup*, are not. The *setup* method interacts with the global ObjectivePlugin, which modifies core game state. **WARNING:** Invoking *setup* from any thread other than the main server game thread will lead to race conditions, data corruption, and server instability. All interactions must be synchronized with the main game loop.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getObjectiveIdToStart() | String | O(1) | Returns the configured identifier of the objective to be started. |
| setup(playerUUIDs, worldUUID, markerUUID, store) | Objective | O(N) | Delegates to the ObjectivePlugin to create and start a new objective instance for N players. Returns the newly created Objective or null on failure. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked by higher-level adventure mode systems that process game events. The following example is a conceptual representation of how an event processor might use a deserialized ObjectiveTypeSetup instance.

```java
// Conceptual example: A trigger system executing a configured objective setup
ObjectiveTypeSetup objectiveAction = trigger.getOnActivateAction(); // Deserialized from an asset

if (objectiveAction instanceof SetupObjective) {
    // The system invokes the setup method, passing in the current context
    objectiveAction.setup(playersInZone, world.getUUID(), trigger.getMarkerUUID(), entityStore);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SetupObjective()`. The object must be created via the codec system from a valid data source to ensure all validators and field initializers are correctly run.
- **Asynchronous Execution:** Do not call the *setup* method from a separate thread. Objective creation is a stateful operation that must be synchronized with the server's main tick loop.

## Data Pipeline
The SetupObjective class is a key component in the data flow that translates static configuration into active game state.

> Flow:
> Adventure Asset File (JSON/HOCON) -> Hytale Codec System -> **SetupObjective Instance** -> Game Event System (e.g., TriggerVolume) -> ObjectivePlugin.startObjective -> New Objective added to World State

