---
description: Architectural reference for BuilderSensorIsBackingAway
---

# BuilderSensorIsBackingAway

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorIsBackingAway extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorIsBackingAway class is a factory component within the server-side NPC AI asset system. Its sole responsibility is to construct an instance of SensorIsBackingAway, which is a runtime condition used by an NPC's behavior tree to determine if it is currently moving away from a target.

This class acts as a bridge between the static data definition of an NPC's behavior (likely defined in a JSON or similar asset file) and the live, in-game AI object. The engine's asset loader identifies the need for a "IsBackingAway" sensor, instantiates this builder, and invokes the `build` method to get the concrete sensor object. This pattern decouples the asset format from the runtime implementation of AI components, allowing for flexible and data-driven AI design.

## Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the NPC asset loading framework, specifically when parsing a behavior tree that references this sensor type. The framework supplies a BuilderSupport context object required for the build process.
-   **Scope:** Extremely short-lived and transient. An instance of this builder typically exists only for the duration of the `build` method call. It is not retained or referenced after the SensorIsBackingAway object has been created.
-   **Destruction:** The builder object is eligible for garbage collection immediately after the `build` method completes. The ownership of the created SensorIsBackingAway is transferred to the NPC's behavior tree.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no mutable fields and its behavior is entirely determined by its inherited configuration and the arguments passed to its methods.
-   **Thread Safety:** This class is inherently **thread-safe**. As a stateless factory, it can be safely used across multiple threads without synchronization. However, the typical usage pattern involves creating a new instance for each build operation, obviating most concurrency concerns.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorIsBackingAway | O(1) | Constructs and returns a new SensorIsBackingAway instance. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description for tooling. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this component for tooling. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay code. It is invoked automatically by the engine's asset systems. A game designer would specify the sensor in a data file, and the engine would use this builder behind the scenes.

The conceptual engine-level usage would resemble the following:

```java
// Hypothetical engine code for loading an NPC behavior
// This is NOT user-facing code.
BuilderSensorBase builder = assetRegistry.getBuilderFor("SensorIsBackingAway");
SensorIsBackingAway sensor = ((BuilderSensorIsBackingAway) builder).build(builderSupport);
npcBehaviorTree.addCondition(sensor);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderSensorIsBackingAway()`. The class is designed to be managed and invoked by the asset pipeline, which provides critical context via the BuilderSupport object.
-   **Caching Instances:** Do not cache and reuse instances of this builder. It is a lightweight, transient object designed to be created and discarded on demand.

## Data Pipeline
This builder is a single transformation step in the NPC asset instantiation pipeline. It converts a data-driven request for a sensor into a live game object.

> Flow:
> NPC Behavior Asset -> Asset Deserializer -> **BuilderSensorIsBackingAway** -> SensorIsBackingAway Instance -> NPC Behavior Tree

