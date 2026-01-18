---
description: Architectural reference for BuilderRelativeWaypointDefinition
---

# BuilderRelativeWaypointDefinition

**Package:** com.hypixel.hytale.server.npc.path.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderRelativeWaypointDefinition extends BuilderBase<RelativeWaypointDefinition> {
```

## Architecture & Concepts
The BuilderRelativeWaypointDefinition is a component within the server-side NPC Asset Loading Framework. Its sole responsibility is to translate a specific JSON configuration structure into a concrete, in-memory RelativeWaypointDefinition object.

This class embodies the **Builder** design pattern. It is a specialized factory that decouples the high-level asset loading system from the specific data format and validation logic required for relative waypoints. When the server loads NPC behaviors from configuration files, a central registry identifies the appropriate builder for a given JSON object type and delegates the parsing and instantiation task to it.

This component acts as a data marshalling and validation layer. It ensures that waypoint definitions loaded from disk are well-formed and conform to engine constraints (e.g., valid rotation ranges, positive distances) before they are integrated into the live pathfinding system.

### Lifecycle & Ownership
- **Creation:** Instances are created dynamically and on-demand by the core NPC asset building system (e.g., a BuilderRegistry or factory). A new instance is typically created for each unique JSON object it needs to process.
- **Scope:** The lifecycle of a BuilderRelativeWaypointDefinition instance is extremely short. It is created, configured via the readConfig method, used once to produce a result via the build method, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is destroyed by the garbage collector shortly after the build method completes. The created RelativeWaypointDefinition object is what persists, not the builder itself.

## Internal State & Concurrency
- **State:** This class is fundamentally **mutable and stateful**. The readConfig method directly modifies the internal rotation and distance fields. This state is temporary and exists only to accumulate configuration before the final object is constructed.

- **Thread Safety:** This class is **not thread-safe**. Its design assumes it will be used in a single-threaded context per-instance. If multiple threads were to access the same instance and call readConfig, a race condition would occur, corrupting the internal state. The asset loading framework guarantees safety by creating a new builder for each parsing task, effectively confining its use to a single operational context.

## API Surface
The primary contract is defined by the BuilderBase superclass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | Builder | O(1) | Parses the input JSON, validates fields, and mutates internal state. Converts rotation from degrees to radians. |
| build(BuilderSupport) | RelativeWaypointDefinition | O(1) | Constructs and returns a new RelativeWaypointDefinition using the internal state. |
| category() | Class | O(1) | Returns the type token for the object this builder creates, used for registry mapping. |

## Integration Patterns

### Standard Usage
This builder is not intended for direct invocation by developers. It is used implicitly by the asset loading system. A developer interacts with it by defining the corresponding data structure in a JSON asset file.

The system will automatically select this builder when it encounters a JSON object configured for this waypoint type.

**Example NPC Behavior JSON:**
```json
{
  "type": "Hytale.RelativeWaypoint",
  "config": {
    "Rotation": 90.0,
    "Distance": 5.0
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BuilderRelativeWaypointDefinition()`. The asset framework is responsible for managing builder lifecycles. Manually creating an instance bypasses the registry and intended workflow.
- **Instance Reuse:** Never reuse a builder instance to parse multiple JSON objects. The internal state from the first `readConfig` call will persist and corrupt the result of the second.
- **State Manipulation:** Do not call `readConfig` and then attempt to manually modify the builder's state before calling `build`. The builder's internal logic and validation should be considered a single, atomic operation.

## Data Pipeline
This builder is a critical step in the pipeline that transforms static configuration files into active game logic for NPC pathfinding.

> Flow:
> NPC Behavior JSON File -> Server Asset Loader -> JSON Parser -> **BuilderRelativeWaypointDefinition** -> `RelativeWaypointDefinition` Object -> NPC Pathing Component -> Live Path Execution

