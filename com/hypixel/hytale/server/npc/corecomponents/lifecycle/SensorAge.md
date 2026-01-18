---
description: Architectural reference for SensorAge
---

# SensorAge

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorAge extends SensorBase {
```

## Architecture & Concepts
SensorAge is a specific implementation of the Sensor contract within the Hytale NPC AI framework. In this system, a Sensor is a component responsible for evaluating a specific condition about the game world, the NPC itself, or its environment. The result of this evaluation, a simple boolean, is used by higher-level AI constructs like Behavior Trees or State Machines to make decisions.

This class serves as a temporal predicate, answering the question: "Does the current in-game time fall within a specific, predefined window?" It is used to gate NPC behaviors, making them active only during certain in-game events, seasons, or time-of-day cycles.

SensorAge decouples the NPC's core logic from the global time system. Instead of directly querying a time service, an AI behavior simply relies on the abstract `matches` contract of this sensor. The sensor, in turn, depends on the `WorldTimeResource` as the single source of truth for the current game time, ensuring consistency across the server.

## Lifecycle & Ownership
- **Creation:** An instance of SensorAge is not created directly by developers. It is instantiated by the server's asset loading pipeline when an NPC definition is parsed. A corresponding builder, `BuilderSensorAge`, reads configuration from an asset file (e.g., JSON) and uses the `BuilderSupport` context to construct the final, immutable SensorAge object.
- **Scope:** The object's lifetime is bound to the NPC asset definition it belongs to. It is created once when the asset is loaded and is shared by all NPC instances of that type. It is effectively a stateless, reusable configuration object.
- **Destruction:** The object is marked for garbage collection when its parent NPC asset definition is unloaded from memory. This typically occurs during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** Immutable. The core state consists of `minAgeInstant` and `maxAgeInstant`, which are final fields set exclusively during construction. The object does not change after it has been created.
- **Thread Safety:** This class is inherently thread-safe. Its immutable state guarantees that concurrent reads from multiple NPC processing threads are safe. The `matches` method is a pure function with respect to the object's state, operating only on its immutable fields and the parameters passed to it.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | The primary evaluation method. Retrieves the current game time from the WorldTimeResource and returns true if it falls inclusively between the min and max age instants defined at construction. |
| getSensorInfo() | InfoProvider | O(1) | Returns an object for debugging and introspection. This implementation currently returns null. |

## Integration Patterns

### Standard Usage
This component is not intended for direct use in gameplay code. It is configured declaratively within an NPC's asset definition file and invoked automatically by the AI runtime.

A designer would specify a time range in an asset file, which the system uses to create and integrate the sensor into an NPC's behavior tree.

```json
// Example pseudo-asset configuration
"sensors": [
  {
    "type": "age",
    "minAge": "event.winter_festival.start",
    "maxAge": "event.winter_festival.end"
  }
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorAge()`. The object must be constructed via the asset pipeline to ensure its time boundaries are correctly resolved from game constants and its dependencies are properly injected.
- **Subclassing for State:** Avoid extending SensorAge to add mutable state. Sensors are designed to be stateless predicates. Any per-NPC instance state should be managed in a separate component, such as an AI Memory store.

## Data Pipeline
The data flow for this component occurs in two distinct phases: configuration and execution.

**Configuration Flow:**
> NPC Asset File -> Asset Loading Service -> BuilderSensorAge -> **SensorAge Instance** (with resolved Instants) -> NPC Definition Registry

**Execution Flow (per AI tick):**
> AI Runtime -> `matches()` call -> WorldTimeResource lookup -> Current Game Time -> **SensorAge** (Comparison) -> Boolean Result -> Behavior Tree Node

