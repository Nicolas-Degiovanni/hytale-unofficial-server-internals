---
description: Architectural reference for BuilderBodyMotionFindBase
---

# BuilderBodyMotionFindBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public abstract class BuilderBodyMotionFindBase extends BuilderBodyMotionBase implements Builder<BodyMotion> {
```

## Architecture & Concepts
The BuilderBodyMotionFindBase is an abstract configuration class that serves as the foundation for all NPC pathfinding and movement behaviors. It operates as a deserializer and container for parameters defined in an NPC's JSON behavior files. This class does not implement the movement logic itself; rather, it prepares a configuration blueprint that is used by a concrete `BodyMotion` instance at runtime.

Its primary architectural role is to translate static JSON data into a flexible, context-aware configuration object. This is achieved through a system of `Holder` objects (e.g., IntHolder, DoubleHolder). Instead of storing raw values, it stores these holders, which can resolve their final values at runtime using a provided `ExecutionContext`. This powerful pattern allows NPC behavior, such as movement speed or pathfinding aggression, to be dynamic and responsive to the current game state (e.g., NPC health, time of day, or proximity to a target).

This class is designed to be extended by more specific builders, such as one for finding a player or another for moving to a static coordinate. The concrete subclass is responsible for implementing the `build` method, which consumes the configured parameters from this base class to construct a fully operational `BodyMotion` object.

### Lifecycle & Ownership
-   **Creation:** An instance of a concrete subclass is created by the server's asset loading system when it parses an NPC's behavior graph from a JSON file. It is never instantiated directly in game logic.
-   **Scope:** The object is extremely short-lived. Its existence is confined to the asset parsing and `BodyMotion` instantiation phase.
-   **Destruction:** It becomes eligible for garbage collection immediately after the `build` method is called and the resulting `BodyMotion` object is returned. It holds no persistent state and is not retained by any long-lived system.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable, but only during the call to `readConfig`. During this phase, its numerous `Holder` fields are populated from the source JSON. After initialization, the object should be treated as effectively immutable. The values *retrieved* via its getters are dynamic, but the underlying configuration *holders* do not change.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for single-threaded use within the asset loading pipeline. The `readConfig` method performs unsynchronized writes to its internal fields. Concurrent access would lead to a corrupted or unpredictable configuration state.

## API Surface
The public API is divided into two distinct phases: configuration and value resolution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | BuilderBodyMotionFindBase | O(N) | **Entry Point.** Populates the builder's internal state from a JSON object. N is the number of keys. Must be called exactly once after instantiation. |
| getRelativeSpeed(support) | double | O(1) | Resolves the configured relative speed using the runtime context from BuilderSupport. |
| getMaxPathLength(support) | int | O(1) | Resolves the maximum allowed path length using the runtime context. |
| isUseSteering(support) | boolean | O(1) | Resolves whether steering behaviors are enabled, respecting the constructor flag and JSON config. |
| getParsedDebugFlags() | EnumSet | O(1) | Returns the set of parsed debugging flags for troubleshooting pathfinding. |

## Integration Patterns

### Standard Usage
This builder is not used directly in game logic. It is part of the NPC asset definition pipeline. A system responsible for loading NPC behaviors would use it to parse a JSON fragment and subsequently build a `BodyMotion` component.

```java
// Hypothetical usage within an NPC asset loader
JsonElement behaviorConfig = parseNpcJsonFile("creeper_behavior.json");
BuilderBodyMotionFindTarget builder = new BuilderBodyMotionFindTarget();

// 1. Configure the builder from the JSON data
builder.readConfig(behaviorConfig);

// 2. The builder is passed to a factory or component to build the final object
//    The 'build' method (in the subclass) uses the getters to configure the BodyMotion.
BodyMotion findMotion = builder.build(builderSupport);

// The builder instance is now discarded.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation in Game Logic:** Do not use `new BuilderBodyMotionFindBase()` during an active game loop. These builders are for configuration-time, not runtime.
-   **State Re-configuration:** Do not call `readConfig` more than once on a single instance. The internal state is not designed to be cleared or reset.
-   **Concurrent Access:** Never share an instance of this builder across multiple threads. It is fundamentally unsafe for concurrent use.
-   **Retaining Instances:** Do not store references to builder instances after the `BodyMotion` has been built. They are transient and should be garbage collected.

## Data Pipeline
The builder acts as a transformation step in the data flow from static configuration files to live game objects.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> `JsonElement` -> **BuilderBodyMotionFindBase.readConfig()** -> Populated `Holder` Fields -> `Subclass.build()` -> Configured `BodyMotion` Instance -> NPC Behavior System

