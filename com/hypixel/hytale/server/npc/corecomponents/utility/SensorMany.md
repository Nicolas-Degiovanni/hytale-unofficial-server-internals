---
description: Architectural reference for SensorMany
---

# SensorMany

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Composite Component Base

## Definition
```java
// Signature
public abstract class SensorMany extends SensorBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts
The **SensorMany** class is a foundational component within the server-side NPC AI framework. It embodies the **Composite** design pattern, enabling the system to treat a collection of individual **Sensor** components as a single, unified sensor. Its primary architectural role is to act as a non-terminal node in the hierarchical tree of NPC components, grouping multiple sensors to construct more complex perception logic.

Subclasses of **SensorMany**, such as **SensorAnd** or **SensorOr**, use this base to define logical combinations of sensory inputs. For example, an NPC might need to detect a player *and* be near a specific block. This would be implemented by a concrete subclass of **SensorMany** containing two child sensors.

The core responsibility of this class is to propagate lifecycle events and other state changes downwards to its constituent child sensors. When the parent **NPCEntity** is spawned, loaded, or teleported, **SensorMany** ensures that every sensor in its collection receives the corresponding notification, maintaining state consistency across the entire component tree.

## Lifecycle & Ownership
The lifecycle of a **SensorMany** instance is strictly managed by the NPC asset system and is directly tied to the lifecycle of its parent **NPCEntity**.

-   **Creation:** Instances are never created directly with the *new* keyword. They are instantiated by the NPC asset loading pipeline via a corresponding **BuilderSensorMany**. This process occurs when an NPC's definition is parsed from an asset file and its component tree is constructed in memory. The collection of child sensors is injected during this build phase.
-   **Scope:** An instance of **SensorMany** persists for the entire duration that its parent **NPCEntity** exists within the game world. It is an integral part of the NPC's in-memory representation.
-   **Destruction:** The object is marked for garbage collection when the parent **NPCEntity** is removed from the world. The **unloaded** and **removed** lifecycle methods are the final callbacks executed on the instance and its children, allowing for state cleanup before destruction.

## Internal State & Concurrency
-   **State:** **SensorMany** is largely a stateless proxy. Its primary internal state is the **sensors** array, which is a final field assigned during construction. The reference to this array is immutable. However, the child **Sensor** objects within the array are expected to contain their own mutable state related to what they are sensing.
-   **Thread Safety:** This class is **not thread-safe**. All NPC components, including sensors, are designed to be accessed and updated exclusively on the main server thread during the game tick. Unsynchronized access from other threads will lead to race conditions, inconsistent state, and server instability.

**WARNING:** All interactions with **SensorMany** or its subclasses must be performed on the server's main game loop thread.

## API Surface
The public API is primarily for introspection and fulfilling the component collection contract. The main behavior is driven by the inherited lifecycle methods which are not listed here.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSensorInfo() | InfoProvider | O(1) | Returns a provider for aggregated debug and state information from child sensors. |
| componentCount() | int | O(1) | Returns the number of child sensors managed by this instance. |
| getComponent(index) | IAnnotatedComponent | O(1) | Retrieves a specific child sensor by its index in the internal collection. |

## Integration Patterns

### Standard Usage
This abstract class is not used directly. A developer defines a concrete implementation (e.g., a logical AND or OR sensor) within an NPC's asset definition file. The engine then handles its creation and lifecycle management. A developer never writes Java code to interact with this class directly.

*Conceptual NPC Asset Definition (e.g., JSON)*
```json
{
  "name": "patrolling_guard",
  "ai": {
    "sensor": {
      "type": "SensorAnd", // A concrete subclass of SensorMany
      "sensors": [
        { "type": "PlayerInRangeSensor", "radius": 10 },
        { "type": "TimeOfDaySensor", "period": "DAY" }
      ]
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SensorAnd()` or any other subclass of **SensorMany**. The component must be created by the asset pipeline to ensure its context and dependencies are correctly injected.
-   **State Mutation:** Do not attempt to modify the internal collection of sensors after construction. The component hierarchy is assumed to be immutable once the NPC is loaded.
-   **Asynchronous Ticking:** Do not call lifecycle methods like **spawned** or **unloaded** manually or from an external thread. These are managed exclusively by the parent **Role** component.

## Data Pipeline
**SensorMany** does not process a data stream. Instead, it acts as a dispatcher in a control-flow pipeline, propagating events down the component tree.

> **Control Flow:**
> Server Tick -> **NPCEntity** Update -> **Role** Update -> **SensorMany.spawned()** -> Propagates call to **ChildSensor1.spawned()**, **ChildSensor2.spawned()**, etc.

