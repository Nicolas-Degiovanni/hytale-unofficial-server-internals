---
description: Architectural reference for SensorAnd
---

# SensorAnd

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient

## Definition
```java
// Signature
public class SensorAnd extends SensorMany {
```

## Architecture & Concepts
The SensorAnd class is a composite conditional component within the server-side NPC AI framework. It functions as a logical **AND** gate, aggregating multiple child Sensor components to form a more complex perception query. This class embodies the Composite design pattern, allowing developers to construct intricate AI behaviors from simple, reusable Sensor primitives.

Its primary role is to evaluate a set of conditions against the current game state. It returns a positive result only if **all** of its contained child Sensors also return a positive result. This evaluation is short-circuited; if any child Sensor fails, the process terminates immediately, providing a performance optimization for complex AI checks.

SensorAnd is a fundamental building block for defining NPC state transitions and action prerequisites. For example, an NPC might only attack if it *sees* a player **AND** it *has* a weapon equipped **AND** its *health* is above a certain threshold. Each of these conditions would be a separate Sensor, composed together by a SensorAnd instance.

## Lifecycle & Ownership
- **Creation:** SensorAnd instances are not created directly via code. They are instantiated by the Hytale asset pipeline, specifically through a corresponding builder class (BuilderSensorAnd), when an NPC's behavior asset file is parsed and loaded by the server.
- **Scope:** An instance of SensorAnd exists for the lifetime of its parent NPC behavior definition. It is a stateless configuration object that is reused across all NPCs that share the same behavior asset.
- **Destruction:** The object is marked for garbage collection when the server unloads the associated NPC asset, typically during a server shutdown or a dynamic asset reload.

## Internal State & Concurrency
- **State:** The component's configuration, specifically its list of child sensors, is immutable after construction. However, the object is internally stateful during the execution of a single *matches* call. It uses a WrappedInfoProvider to temporarily aggregate positional data from its children, but this state is cleared at the start of every call and is not intended to persist.

- **Thread Safety:** **WARNING:** This class is not thread-safe and is designed for single-threaded access. It must only be invoked from the main server thread responsible for ticking the specific NPC entity. Concurrent access to the *matches* method on the same instance will result in race conditions, state corruption, and undefined AI behavior. The engine's NPC update loop guarantees this sequential execution.

## API Surface
The public contract is minimal, focusing entirely on state evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Evaluates all child sensors. Returns true only if all children return true. This method short-circuits, returning false immediately upon the first child failure. N is the number of child sensors. |

## Integration Patterns

### Standard Usage
Direct interaction with this class in Java is incorrect. The standard pattern is to define its behavior declaratively within an NPC asset file (e.g., a JSON or HOCON file). The engine's asset loader is responsible for its construction and integration into the AI system.

A conceptual asset definition would look like this:
```json
// Example NPC Behavior Asset Snippet
{
  "type": "SensorAnd",
  "sensors": [
    {
      "type": "SensorTargetInView",
      "viewAngle": 90
    },
    {
      "type": "SensorHealth",
      "minPercent": 0.25
    },
    {
      "type": "SensorTargetInRange",
      "maxDistance": 10
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorAnd(...)` in game logic. The object's lifecycle is managed entirely by the asset system. Manual creation will bypass critical initialization and result in a non-functional component.
- **State Persistence:** Do not attempt to read state from a SensorAnd instance after a *matches* call. Any internal state, such as aggregated positional data, is transient and only valid for the duration of that single method execution.
- **Modification After Creation:** Do not attempt to modify the internal list of child sensors after the object has been constructed by the asset loader. This will lead to unpredictable behavior across all NPCs using that shared asset.

## Data Pipeline
SensorAnd acts as a conditional gate in the NPC data processing pipeline. It consumes game state and produces a binary decision that influences the NPC's behavior tree.

> Flow:
> NPC Asset File -> Server Asset Loader -> **SensorAnd** (In-Memory Representation) -> NPC Update Tick -> `matches()` call with current Game State -> Boolean Result -> Behavior Tree State Transition

