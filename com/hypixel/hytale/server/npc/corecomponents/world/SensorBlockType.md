---
description: Architectural reference for SensorBlockType
---

# SensorBlockType

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Component / Decorator

## Definition
```java
// Signature
public class SensorBlockType extends SensorBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts
The SensorBlockType is a conditional component within the server-side NPC AI framework. It operates as a **Decorator** that wraps another Sensor component, adding a specific world-state check to its logic. Its primary function is to verify if the block at a position provided by the wrapped sensor belongs to a predefined BlockSet.

This class acts as a filter or a gate within an NPC's behavior tree or state machine. For example, an NPC might use a generic PositionSensor to find a point of interest, and a SensorBlockType would be layered on top to ensure that point of interest is on a specific type of ground, like grass or stone, before proceeding with an action.

It is a critical bridge between the abstract logic of the NPC AI system and the concrete, mutable state of the game world. All lifecycle events are delegated directly to the wrapped Sensor, meaning SensorBlockType is transparent for lifecycle management but active during the logical evaluation phase of an AI tick.

## Lifecycle & Ownership
- **Creation:** SensorBlockType is not intended for direct instantiation. It is constructed exclusively by the NPC asset loading pipeline, specifically via a BuilderSensorBlockType, when parsing an NPC's behavior definition from its configuration files.
- **Scope:** The object's lifetime is bound to the lifetime of its parent Role component, which represents the active behavior set for a single NPC entity. It persists as long as the NPC is active and using that Role.
- **Destruction:** The component is eligible for garbage collection when its parent Role is destroyed. This typically occurs when an NPC is unloaded from the world or its behavior is fundamentally changed. The delegated `unloaded` and `removed` methods signal this teardown process.

## Internal State & Concurrency
- **State:** The component's internal state consists of two final fields: the wrapped `sensor` and the integer `blockSet`. This configuration is immutable after construction. However, its `matches` method depends on external, highly mutable state: the World's block data and the state of the wrapped sensor's InfoProvider.
- **Thread Safety:** **This class is not thread-safe.** Its core `matches` method accesses live world data (WorldChunk) which is not designed for concurrent access. All interactions with a SensorBlockType instance must be performed on the main server thread to prevent chunk access violations, race conditions, and data corruption.

## API Surface
The public contract is dominated by the `matches` method, which performs the primary logical check.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the wrapped sensor and then checks the block type at the resulting position. Returns false if the chunk is not in memory. |
| getSensorInfo() | InfoProvider | O(1) | Returns the InfoProvider from the wrapped sensor, exposing its state. |

## Integration Patterns

### Standard Usage
This component is not invoked directly by developers. It is configured declaratively in NPC asset files and managed by the AI system. The system invokes the `matches` method during an NPC's update tick as part of a larger behavior evaluation.

A conceptual invocation by the AI engine would look like this:
```java
// Executed by a higher-level AI system (e.g., a Behavior Tree Node)
boolean conditionMet = sensorBlockType.matches(entityRef, currentRole, deltaTime, entityStore);

if (conditionMet) {
    // Proceed with action, e.g., pathfind to the location
    // provided by sensorBlockType.getSensorInfo().
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SensorBlockType()`. The component is deeply integrated with the asset loading pipeline and requires a builder for correct initialization. Manual creation will result in a non-functional component.
- **Cross-thread Access:** Calling `matches` from any thread other than the main server thread is strictly forbidden and will lead to server instability or crashes.
- **State Caching:** Do not cache the boolean result of `matches` across ticks. The world state can change at any moment, and the check must be re-evaluated with fresh data on every AI tick.

## Data Pipeline
The `matches` method follows a clear data-flow sequence to arrive at its boolean result. The process is designed to fail fast, aborting as soon as a condition is not met.

> Flow:
> AI Tick -> **SensorBlockType.matches()** -> Wrapped Sensor.matches() -> IPositionProvider -> World -> WorldChunk Lookup -> Block ID Retrieval -> BlockSetModule.blockInSet() -> Boolean Result

