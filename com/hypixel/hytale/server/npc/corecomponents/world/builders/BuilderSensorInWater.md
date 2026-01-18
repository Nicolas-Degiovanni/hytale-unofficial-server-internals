---
description: Architectural reference for BuilderSensorInWater
---

# BuilderSensorInWater

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorInWater extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorInWater class is a stateless factory component within the server-side NPC Behavior system. Its sole responsibility is to instantiate a concrete SensorInWater object, which is the runtime component that actively checks if an NPC entity is submerged in water.

In Hytale's NPC architecture, behavior trees are defined in data assets (e.g., JSON files). The engine parses these assets and uses a registry of "Builder" classes to construct the live behavior tree in memory. BuilderSensorInWater serves as the bridge between the static data definition of an "IsInWater" sensor and its live, executable counterpart. This pattern decouples behavior definition from implementation, allowing designers to compose complex AI without writing code.

This class represents a leaf node in the AI *configuration* graph, responsible for producing a leaf node in the AI *runtime* graph.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorInWater is typically created by the NPC asset loading system when it encounters the corresponding identifier in an NPC's behavior definition file. It is not intended for manual instantiation.
- **Scope:** This object is transient and stateless. Its lifecycle is expected to be short. It may be created on-demand by the asset system, used to build the SensorInWater instance, and then immediately become eligible for garbage collection.
- **Destruction:** Managed by the JVM garbage collector. As a stateless factory, there are no explicit cleanup or teardown requirements.

## Internal State & Concurrency
- **State:** **Immutable and Stateless.** This class contains no instance fields and its behavior is consistent across all calls. It does not cache any data.
- **Thread Safety:** **Fully Thread-Safe.** Due to its stateless nature, a single instance can be safely shared and used across multiple threads without any synchronization mechanisms. The primary `build` method produces a new, distinct object on each invocation, preventing any shared state issues.

## API Surface
The public API is minimal, focusing exclusively on metadata and object creation for the NPC framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Factory method. Instantiates and returns a new SensorInWater object. |
| getShortDescription() | String | O(1) | Provides a human-readable summary for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides a detailed human-readable description. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the development status of this component, indicating it is stable. |

## Integration Patterns

### Standard Usage
This class is not designed for direct use by gameplay programmers. It is invoked internally by the NPC behavior tree factory during the NPC initialization process. The framework locates the appropriate builder from a registry and uses it to construct the runtime sensor.

```java
// Conceptual example of framework-level usage
// Do not replicate this pattern in gameplay code.

// 1. Framework retrieves the builder from a registry
BuilderSensorBase builder = npcBehaviorRegistry.getBuilderFor("SensorInWater");

// 2. Framework invokes the build method with context
Sensor runtimeSensor = builder.build(builderSupportContext);

// 3. The resulting sensor is attached to the NPC's behavior tree
npc.getBehaviorTree().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderSensorInWater()` in game logic. NPC behaviors should be defined entirely within data assets. Manually constructing AI components breaks the data-driven design and creates brittle, hard-to-debug code.
- **Caching Instances:** Do not store and reuse an instance of BuilderSensorInWater. It is a lightweight, transient object designed to be created and discarded.
- **Extending the Class:** This is a concrete, final implementation. Subclassing it is not a supported extension point and will likely break the assumptions of the NPC asset pipeline.

## Data Pipeline
BuilderSensorInWater acts as a transformation step during the NPC asset-to-runtime pipeline. It translates a declarative configuration token into an active game object.

> Flow:
> NPC Asset File (e.g., `mob.json`) -> Hytale Asset Parser -> **BuilderSensorInWater** -> `build()` Invocation -> `SensorInWater` Instance -> Live NPC Behavior Tree

