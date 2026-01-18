---
description: Architectural reference for the Sensor interface, a core component of the Hytale NPC AI system.
---

# Sensor

**Package:** `com.hypixel.hytale.server.npc.instructions`
**Type:** Behavioral Contract (Interface)

## Definition
```java
// Signature
public interface Sensor extends RoleStateChange, IAnnotatedComponent, IComponentExecutionControl {
```

## Architecture & Concepts

The Sensor interface is a fundamental building block of the server-side NPC artificial intelligence system. It represents the "Sense" phase in a classic "Sense-Think-Act" AI loop. A Sensor's sole responsibility is to act as a predicate—a conditional check against the state of the world—to determine if an associated AI behavior should be considered for activation.

Architecturally, a Sensor is a stateless component that encapsulates a single, specific query about the game world from the perspective of an NPC. Examples of such queries include:
- Is the NPC's health below 50%?
- Is there a hostile entity within a 10-meter radius?
- Is the current time of day "night"?
- Is the NPC currently standing on a "grass" block?

Sensors are not executed in isolation. They are composed within higher-level AI constructs, such as an `Instruction` or a state transition rule within a `Role`. The AI execution engine iterates over these constructs, invoking the `matches` method on each Sensor. A return value of true signals to the engine that the conditions for a specific action or state change have been met.

The interface extends several other key contracts, indicating its deep integration into the NPC component model:
- **RoleStateChange:** Allows the Sensor to react to changes in the NPC's active `Role`, such as being activated or deactivated.
- **IAnnotatedComponent:** Provides a mechanism for attaching metadata, likely used by developer tools, debuggers, or for dynamic configuration.
- **IComponentExecutionControl:** Implies that the Sensor's lifecycle and execution are managed by a higher-level control system, which can start, stop, or tick the component.

The static `Sensor.NULL` field is an implementation of the Null Object Pattern, providing a safe, inert default that always evaluates to false. This is critical for preventing null pointer exceptions in unconfigured or dynamically assembled AI behaviors.

### Lifecycle & Ownership
- **Creation:** Implementations of Sensor are instantiated during the loading of an NPC's behavior configuration. They are typically defined in data files (e.g., JSON or HOCON) and deserialized into concrete objects alongside the `Actions` they gate. They are owned by the containing `Instruction` or `Role`.
- **Scope:** A Sensor instance is a long-lived configuration object. Its lifetime is tied to the lifetime of the NPC's behavior definition, which generally persists as long as the NPC entity exists.
- **Destruction:** The object is eligible for garbage collection when the NPC entity is destroyed and its associated behavior configuration is unloaded from memory.

## Internal State & Concurrency
- **State:** The Sensor contract is designed for stateless implementations. Any configurable parameters (e.g., a detection radius) should be treated as immutable state set at creation time. A Sensor's purpose is to evaluate external world state, not to maintain its own mutable state across ticks.
- **Thread Safety:** Implementations of `matches` must be thread-safe. While the AI for a single NPC is typically ticked on a single thread, the arguments passed to `matches`, such as the `EntityStore`, provide a view into a potentially shared world state. Implementations must not cause side effects or modify shared state. All operations should be read-only.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Ref, Role, double, Store) | boolean | Variable | The core predicate method. Returns true if world conditions meet the Sensor's criteria. Complexity is implementation-dependent, from O(1) for simple state checks to O(N) or worse for complex spatial queries. |
| done() | void | O(1) | A lifecycle callback, invoked when the Sensor is no longer being actively considered by the AI engine. Used for internal state cleanup if necessary. |
| getSensorInfo() | InfoProvider | O(1) | Returns an object providing metadata about the Sensor's current state or configuration, primarily for debugging and visualization tools. |

## Integration Patterns

### Standard Usage
A Sensor is never invoked directly by game logic. It is configured as part of an NPC's behavior tree or state machine and is evaluated automatically by the AI execution engine each tick. The developer's primary interaction is to implement the interface to create new, custom world-state queries.

```java
// Example: A Sensor implementation is defined and attached to an Instruction.
// The AI system later invokes `matches` on this sensor.

// This code is conceptual and does not represent a direct engine API call.
Instruction followPlayerInstruction = new Instruction();
Sensor playerNearbySensor = new PlayerDistanceSensor(15.0); // Custom implementation

// The Sensor is registered as a condition for the instruction.
followPlayerInstruction.addSensor(playerNearbySensor);

// The game's AI engine will later evaluate it:
// boolean canFollow = playerNearbySensor.matches(worldRef, npcRole, ...);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store mutable state within a Sensor implementation that changes between calls to `matches`. This violates the stateless predicate pattern and can lead to highly unpredictable and difficult-to-debug AI behavior.
- **Expensive Computations:** The `matches` method may be called many times per second for every active NPC. Avoid computationally expensive operations like complex pathfinding or full-world data scans. Offload such tasks to asynchronous systems and have the Sensor check the result.
- **World Modification:** A Sensor must never modify the world state. Its role is strictly read-only. Modifying entities or world data from within `matches` will cause severe concurrency issues and state corruption.

## Data Pipeline
The Sensor acts as a gate in the NPC data processing pipeline. It consumes world state and produces a boolean signal that controls the flow of execution.

> Flow:
> Server Tick -> AI System Update -> **Sensor.matches(World State)** -> [true] -> Instruction/Action Execution -> World State Mutation
>
> `[false]` -> Execution path is terminated.

---

