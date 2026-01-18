---
description: Architectural reference for BuilderSensorFlockCombatDamage
---

# BuilderSensorFlockCombatDamage

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorFlockCombatDamage extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorFlockCombatDamage class is a factory component within the server-side NPC Artificial Intelligence framework. Its specific role is to deserialize a JSON configuration snippet and construct an instance of SensorFlockCombatDamage. This class is a key part of the engine's asset-to-object pipeline for AI behaviors.

In the Hytale AI system, a **Sensor** is a component that perceives the game world, providing conditional checks for a behavior tree. This builder creates a sensor that specifically tests if any member of an NPC's flock (or just the leader) has recently received combat damage.

It acts as a bridge between the declarative JSON asset format used by designers and the concrete, executable Java objects used by the server's AI runtime. By extending BuilderSensorBase, it adheres to a standardized contract for creating all AI sensor types, allowing the asset loading system to handle them polymorphically.

### Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the server's NPC asset loading system when it encounters a corresponding sensor definition in an NPC's behavior configuration file. It is not intended for manual instantiation during gameplay logic.
-   **Scope:** The builder object is extremely short-lived. Its scope is confined to the asset deserialization process. It exists only to parse a JsonElement, configure its internal state, and produce a SensorFlockCombatDamage object via the build method.
-   **Destruction:** The builder becomes eligible for garbage collection immediately after the build method returns its result to the asset loader. It does not persist.

## Internal State & Concurrency
-   **State:** The class maintains a single, mutable boolean field: leaderOnly. This state is populated by the readConfig method and is used to configure the final SensorFlockCombatDamage object. The state is transient and exists only for the duration of the build process.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within a single-threaded asset loading context. Accessing a single instance from multiple threads will lead to race conditions on its internal state. This design is intentional, as the asset pipeline is a sequential, single-threaded process.

## API Surface
The public API is designed for use by the automated asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorFlockCombatDamage | O(1) | Constructs and returns a new SensorFlockCombatDamage instance based on the internal state. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the provided JSON data to configure the builder's internal state. This method must be called before build. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling and editor UIs. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description for tooling and editor UIs. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the development status of this component, e.g., Stable. |
| isLeaderOnly() | boolean | O(1) | A simple accessor for the configured leaderOnly state. |

## Integration Patterns

### Standard Usage
This class is not used directly in gameplay code. It is invoked by the engine's asset pipeline. A designer or developer would define the sensor in a JSON file, which the engine then uses to drive the builder.

*Conceptual JSON Definition:*
```json
{
  "type": "SensorFlockCombatDamage",
  "LeaderOnly": true
}
```

The engine would internally locate and use BuilderSensorFlockCombatDamage to process this definition and inject the resulting Sensor into an NPC's behavior tree.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually instantiate and use this class in game logic, such as `new BuilderSensorFlockCombatDamage()`. The AI framework manages its lifecycle entirely.
-   **State Reuse:** Do not cache an instance of this builder for reuse. Each sensor object must be created from a fresh builder instance to ensure state isolation.
-   **Configuration Skipping:** Calling build without first having the engine call readConfig will result in an improperly configured sensor that uses default values, which may lead to unintended AI behavior.

## Data Pipeline
This builder operates within a configuration and asset loading pipeline, not a real-time gameplay data pipeline.

> Flow:
> NPC Behavior JSON Asset -> Asset Manager -> JSON Parser -> **BuilderSensorFlockCombatDamage.readConfig()** -> **BuilderSensorFlockCombatDamage.build()** -> SensorFlockCombatDamage Instance -> NPC Behavior Tree

