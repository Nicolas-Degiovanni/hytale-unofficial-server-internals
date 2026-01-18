---
description: Architectural reference for SensorLight
---

# SensorLight

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Component

## Definition
```java
// Signature
public class SensorLight extends SensorBase {
```

## Architecture & Concepts
The SensorLight is a specialized component within the Hytale NPC AI framework, designed to evaluate ambient light levels at a specific world position. As a subclass of SensorBase, it functions as a conditional node within an NPC's Behavior Tree. Its primary role is to answer the question: "Does the light level at a target location meet a specific set of criteria?"

This component is a key enabler for creating environmentally aware AI behaviors. For example, it can be used to make creatures of the night retreat from sunlight, or to ensure a merchant NPC only operates in a well-lit area.

The sensor's logic is data-driven, configured entirely through NPC asset files. During asset loading, a BuilderSensorLight object is used to construct and configure an instance of SensorLight. This pattern decouples the sensor's logic from its configuration, allowing designers to tweak AI behavior without changing source code.

Internally, it delegates the complex task of sampling and comparing light values to a dedicated LightRangePredicate utility. This follows the principle of single responsibility, keeping the SensorLight focused on its role within the AI system—identifying the target and retrieving world state—while the predicate handles the low-level light value mathematics.

## Lifecycle & Ownership
- **Creation:** SensorLight is instantiated exclusively by the server's NPC asset loading pipeline. A corresponding BuilderSensorLight, parsed from an NPC's definition file, provides the necessary configuration during the construction of an NPC's behavior tree.
- **Scope:** An instance of SensorLight is owned by a single NPC's behavior tree. Its lifetime is directly tied to the lifetime of that NPC entity in the world.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is unloaded or destroyed. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state of SensorLight is effectively **immutable** after construction. The target slot index and the LightRangePredicate, which contains all light level criteria, are marked as final and are configured only once in the constructor. This design ensures that a sensor's behavior is consistent and predictable throughout its lifetime.
- **Thread Safety:** This class is not thread-safe by design. The matches method reads from the World and EntityStore, which are not safe for concurrent modification and access. It is architecturally guaranteed to be called only from the main server thread responsible for ticking the world in which the parent NPC resides.

**WARNING:** Calling the matches method from any thread other than the owning world's main tick thread will lead to severe concurrency issues, including data corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | The core evaluation method. Returns true if the light level at the target's position satisfies the configured predicate. |
| getSensorInfo() | InfoProvider | O(1) | Intended for debugging and visualization tools. The current implementation returns null. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in code. Instead, it is defined declaratively within an NPC's asset file (e.g., a JSON file). The behavior tree executor then invokes the sensor automatically.

A conceptual asset definition might look like this:

```json
// Example NPC Behavior Node
{
  "type": "Condition",
  "sensor": {
    "type": "SensorLight",
    "sunlightRange": [0, 2], // Condition passes if sunlight is very low
    "useTargetSlot": -1 // Use the NPC's own position
  },
  "child": {
    // Behavior to execute if in shadow, e.g., "Flee"
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new SensorLight(). The component is deeply integrated with the asset loading pipeline and cannot be constructed manually without a valid BuilderSensorLight and BuilderSupport context.
- **External State Modification:** Do not attempt to retrieve the LightRangePredicate and modify its state via reflection. The sensor's configuration is intended to be immutable after asset loading.

## Data Pipeline
The SensorLight acts as a gate in the data flow of an NPC's decision-making process. It transforms a world-state query into a simple boolean that influences the behavior tree's execution path.

> Flow:
> NPC Asset File -> Asset Loader -> BuilderSensorLight -> **SensorLight Instance** -> Behavior Tree Tick -> `matches()` -> World Light Query -> Boolean Result -> Behavior Tree Path Selection

