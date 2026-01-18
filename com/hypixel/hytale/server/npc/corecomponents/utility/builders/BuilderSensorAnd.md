---
description: Architectural reference for BuilderSensorAnd
---

# BuilderSensorAnd

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient / Factory

## Definition
```java
// Signature
public class BuilderSensorAnd extends BuilderSensorMany {
```

## Architecture & Concepts
The BuilderSensorAnd class is a factory component within the server-side NPC AI framework. Its sole responsibility is to construct an instance of a SensorAnd composite object from a declarative definition, typically loaded from an asset file.

In Hytale's AI system, a Sensor is a component that allows an NPC to perceive its environment and evaluate conditions. The BuilderSensorAnd facilitates the creation of a complex logical condition: it builds a SensorAnd that only signals *true* if **all** of its contained child Sensors also signal true.

This class acts as a bridge between the static data representation of an NPC's behavior (e.g., a JSON file) and the live, executable object graph that constitutes the NPC's "brain" at runtime. It is a key part of the asset hydration pipeline for AI behaviors.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorAnd is created by the NPC asset loading system when it parses a block defining a logical AND sensor group. The asset loader populates this builder with other builder objects corresponding to the child sensors.
- **Scope:** The lifecycle of a BuilderSensorAnd instance is extremely short and transient. It exists only during the NPC instantiation and configuration phase.
- **Destruction:** The object is eligible for garbage collection immediately after the `build` method is called and the resulting SensorAnd object is integrated into the parent AI component. It is not managed, pooled, or reused.

## Internal State & Concurrency
- **State:** BuilderSensorAnd is stateful. It internally maintains a list of child Sensor builders via its parent class, BuilderSensorMany. This state is populated once during asset parsing and is read during the `build` process.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed to be used exclusively within the single-threaded context of the server's asset loading and NPC initialization pipeline. Concurrent modification of its internal builder list would result in a corrupted SensorAnd object.

## API Surface
The public API is primarily for internal framework use and providing metadata to development tools.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for tooling. |
| getLongDescription() | String | O(1) | Provides a detailed explanation of the component's function for tooling. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns a stability flag (e.g., Stable) for asset editors. |
| build(BuilderSupport) | SensorAnd | O(N) | Constructs the final SensorAnd object. N is the number of child sensors. Returns null if no child sensors were defined, effectively pruning this condition. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked by the NPC asset pipeline. The conceptual flow is as follows:

```java
// Conceptual example of framework usage
// The framework would have previously populated the builder's internal list.
BuilderSensorAnd builder = new BuilderSensorAnd();
// ... framework configures builder with child sensor builders ...

// The framework provides a support context and invokes the build method.
SensorAnd finalSensor = builder.build(npcAssetSupportContext);

// The finalSensor is then attached to the NPC's AI controller.
npc.getAI().addSensor(finalSensor);
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not manually instantiate this class in game logic. NPC behaviors should be defined in data files and processed by the asset pipeline. Bypassing this pattern breaks the declarative nature of the AI system.
- **State Reuse:** Do not hold a reference to a BuilderSensorAnd instance after calling `build`. It is a one-shot factory and its internal state should not be reused or modified post-build.

## Data Pipeline
The BuilderSensorAnd is a transformation step in the data flow from asset definition to a runtime AI component.

> Flow:
> NPC Definition Asset (JSON) -> Asset Deserializer -> **BuilderSensorAnd Instance** -> `build()` Invocation -> `SensorAnd` Runtime Instance -> NPC Behavior Tree

