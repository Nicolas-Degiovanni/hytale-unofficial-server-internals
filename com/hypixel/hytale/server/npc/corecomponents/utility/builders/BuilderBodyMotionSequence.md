---
description: Architectural reference for BuilderBodyMotionSequence
---

# BuilderBodyMotionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient (Builder)

## Definition
```java
// Signature
public class BuilderBodyMotionSequence extends BuilderMotionSequence<BodyMotion> {
```

## Architecture & Concepts
The BuilderBodyMotionSequence is a specialized, transient data container used exclusively during the server-side NPC asset loading pipeline. It embodies the Builder pattern to construct an immutable, runtime-ready BodyMotionSequence object.

Its primary role is to act as an intermediary representation of an NPC's motion sequence as defined in an asset file (e.g., a JSON or HOCON definition). The deserialization framework populates an instance of this builder with raw, unlinked data. The final, usable BodyMotionSequence is then constructed by invoking the build method, which uses a BuilderSupport context to resolve any references and finalize the object graph.

This two-phase construction (deserialization into a builder, then building the final object) is a critical engine pattern. It decouples the raw asset data from the live game-state, allowing for complex object linking and validation to occur in a controlled "baking" step before the asset is used by the NPC simulation.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's asset deserialization framework when it parses an NPC definition file. It is not intended for manual instantiation by developers.
- **Scope:** The lifecycle of a BuilderBodyMotionSequence instance is extremely short. It exists only for the duration of the asset loading and linking process for a single NPC definition.
- **Destruction:** Once the build method is called and the final BodyMotionSequence is returned, the builder instance has served its purpose. It holds no further references and is immediately eligible for garbage collection.

## Internal State & Concurrency
- **State:** This object is fundamentally mutable and stateful. Its core responsibility is to accumulate the list of BodyMotion steps that will comprise the final sequence. The internal state is held within the BuilderObjectListHelper field.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be confined to the single thread responsible for loading a specific NPC asset. Concurrent access will lead to a corrupted internal list and unpredictable server behavior.

## API Surface
The public API is minimal, exposing only the necessary methods for the asset pipeline to construct the final object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionSequence | O(N) | Constructs the final, immutable BodyMotionSequence. N is the number of steps. This is the terminal operation for the builder. |
| getSteps(BuilderSupport) | BodyMotion[] | O(N) | Resolves and builds the list of BodyMotion objects into a concrete array. Primarily for internal use by the BodyMotionSequence constructor. |
| category() | Class<BodyMotion> | O(1) | Returns the class literal for BodyMotion, used for type metadata and reflection within the asset system. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in standard game logic. It is an internal component of the asset loading system. The following example illustrates its conceptual place within that pipeline.

```java
// Conceptual example of the asset loader's internal logic
// Developer Note: You would not write this code yourself.

// 1. Deserializer creates the builder from a file
BuilderBodyMotionSequence motionBuilder = deserializer.load("npc_motions.json");

// 2. The asset manager provides a context and triggers the build
BuilderSupport buildContext = assetManager.createBuildContext();
BodyMotionSequence finalSequence = motionBuilder.build(buildContext);

// 3. The final sequence is attached to an NPC
npc.setMotionSequence(finalSequence);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderBodyMotionSequence()`. The object is useless without being populated by the asset deserializer.
- **State Reuse:** Do not attempt to modify or reuse a builder after its build method has been called. Treat it as a single-use object.
- **Manual Population:** Avoid manually adding steps to the builder. All state should originate from the authoritative NPC asset files to ensure data integrity.

## Data Pipeline
The BuilderBodyMotionSequence is a key stage in the transformation of static asset data into a live, in-memory game object.

> Flow:
> NPC Asset File (JSON) -> Server Deserializer -> **BuilderBodyMotionSequence** (unlinked state) -> `build()` with BuilderSupport -> `BodyMotionSequence` (linked, runtime object) -> NPC Behavior System

