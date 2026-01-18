---
description: Architectural reference for BuilderBodyMotionFind
---

# BuilderBodyMotionFind

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionFind extends BuilderBodyMotionFindWithTarget {
```

## Architecture & Concepts

The BuilderBodyMotionFind class is a key component of the server's NPC Behavior Asset Pipeline. It functions as a deserializer and factory for the `BodyMotionFind` runtime component, which governs how an NPC pursues a target using pathfinding and steering.

This class embodies a **Configuration-as-Data** architecture. Instead of hard-coding NPC movement parameters in Java, they are defined in external JSON files. This builder is responsible for parsing a specific JSON object and translating it into a configurable, ready-to-use Java object.

Its core architectural pattern is **Deferred Resolution**. The builder does not store final, concrete values like `double` or `boolean`. Instead, it uses specialized `...Holder` objects (e.g., DoubleHolder, BooleanHolder). These Holders store the configuration as defined in the JSON, which may include static values, references to game variables, or other dynamic expressions. The final values are resolved at runtime by the `BodyMotionFind` instance, using an `ExecutionContext` provided via the BuilderSupport object. This decouples the behavior *definition* from its *execution*, allowing for highly dynamic and flexible NPC designs.

This builder extends BuilderBodyMotionFindWithTarget, inheriting the fundamental logic for identifying and managing a target. Its specific responsibility is to add parameters related to pathfinding, approach distances, and the transition between long-range pathing and short-range steering.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderBodyMotionFind is created exclusively by the server's internal asset loading system. This occurs when an NPC's behavior tree is being parsed from a JSON configuration file and a component of this type is encountered.
-   **Scope:** The object is **transient and short-lived**. Its entire lifecycle is confined to the asset parsing and NPC instantiation process. It exists only to deserialize the JSON data and produce a single `BodyMotionFind` instance.
-   **Destruction:** The builder object is eligible for garbage collection immediately after the `build` method has been called and the resulting `BodyMotionFind` component has been integrated into the NPC's runtime behavior set. It holds no persistent state and is not referenced after instantiation is complete.

## Internal State & Concurrency

-   **State:** The internal state is **mutable** during the configuration phase, which is exclusively managed by the `readConfig` method. The state consists of several `...Holder` fields that store the unresolved configuration data. After `readConfig` completes, the object's state should be considered immutable.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to operate within a single-threaded asset loading pipeline. Concurrent calls to `readConfig` will result in a corrupted state. All interactions with a given instance should be confined to the thread that created it.

## API Surface

The public API is designed for internal use by the asset pipeline and the resulting `BodyMotionFind` object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionFind | O(1) | Factory method. Constructs and returns a new `BodyMotionFind` runtime instance, passing a reference to itself for deferred parameter resolution. |
| readConfig(JsonElement) | BuilderBodyMotionFind | O(N) | Deserializes a JSON object, populating the internal Holder fields. N is the number of keys in the JSON. Throws exceptions on validation failure. |
| getStopDistance(BuilderSupport) | double | O(1) | Resolves and returns the configured stop distance. Intended to be called by the created `BodyMotionFind` instance at runtime. |
| getAbortDistance(BuilderSupport) | double | O(1) | Resolves and returns the distance at which the pathfinding behavior should be aborted. Intended for internal consumption by the runtime component. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use by developers. Its primary user is the server's asset loading system. A game designer defines the behavior in a JSON file, which the system then uses to configure and instantiate this builder.

**Conceptual JSON Configuration:**
```json
{
  "type": "BodyMotionFind",
  "target": { ... },
  "StopDistance": 5.0,
  "SlowDownDistance": 10.0,
  "AbortDistance": 100.0,
  "SwitchToSteeringDistance": 15.0
}
```

**Internal System Flow (Simplified):**
```java
// This code is executed by the asset loading system, not a game logic developer.
JsonElement motionConfig = parseNpcBehaviorFile("path/to/behavior.json");

// The system identifies the "type" and instantiates the correct builder.
BuilderBodyMotionFind builder = new BuilderBodyMotionFind();
builder.readConfig(motionConfig);

// The builder is used to create the runtime component for an NPC instance.
BodyMotionFind runtimeComponent = builder.build(npc.getBuilderSupport());
npc.addMovementComponent(runtimeComponent);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not manually create an instance using `new BuilderBodyMotionFind()` in game logic code. The configuration and lifecycle are managed entirely by the asset pipeline.
-   **Post-Config Modification:** Do not attempt to modify the builder's state after `readConfig` has been called. The object is not designed for this, and behavior would be unpredictable.
-   **External State Resolution:** Do not call the `get...` methods from outside the `BodyMotionFind` instance that this builder creates. These methods require a valid runtime context (BuilderSupport) that is only guaranteed to be correct for the associated NPC.

## Data Pipeline

The flow of configuration data through this component is linear and unidirectional, transforming declarative JSON into an active runtime object.

> Flow:
> NPC Behavior JSON File -> Server Asset Deserializer -> **BuilderBodyMotionFind.readConfig()** -> Configured Builder Instance -> **build()** -> BodyMotionFind Runtime Object -> NPC Behavior System

