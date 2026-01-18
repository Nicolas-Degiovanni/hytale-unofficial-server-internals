---
description: Architectural reference for BuilderHeadMotionSequence
---

# BuilderHeadMotionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderHeadMotionSequence extends BuilderMotionSequence<HeadMotion> {
```

## Architecture & Concepts
The BuilderHeadMotionSequence is a transient, stateful object that serves as a factory within the server's NPC asset pipeline. Its sole responsibility is to accumulate the configuration for a sequence of head movements and, upon request, construct an immutable, game-ready HeadMotionSequence instance.

This class is not intended for direct use in gameplay logic. Instead, it acts as an intermediate representation of data that is deserialized from an NPC's asset definition files. It bridges the gap between static asset data (e.g., JSON on disk) and the live, optimized objects used by the NPC behavior system at runtime. The dependency on BuilderSupport signifies its role within a larger, context-aware build process that may involve resolving references or inheriting properties.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by a deserialization framework (e.g., Gson) during the server's asset loading phase. An NPC definition file containing a head motion sequence will be parsed directly into a BuilderHeadMotionSequence object.
- **Scope:** The lifecycle of this object is extremely brief. It exists only for the duration of the asset-to-object build process.
- **Destruction:** Once the build method is invoked and the final HeadMotionSequence is created, the builder instance is no longer referenced by the asset pipeline and becomes immediately eligible for garbage collection. Holding a reference to it post-build is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** This object is fundamentally mutable. Its primary internal state is the `steps` field, a BuilderObjectListHelper which contains a list of individual HeadMotion definitions. This state is progressively built up during deserialization.
- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access within the asset loading system. Any concurrent modification of its internal list of steps will result in unpredictable behavior, data corruption, and potential crashes. All interactions must be confined to the thread responsible for loading NPC assets.

## API Surface
The public API is minimal, exposing only the methods required by the asset build system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | HeadMotionSequence | O(N) | Terminal operation. Consumes the builder's state to produce an immutable HeadMotionSequence. N is the number of steps. |
| getSteps(BuilderSupport) | HeadMotion[] | O(N) | Builds the list of motions and returns them as a concrete array. Primarily for internal use by the constructed HeadMotionSequence. |
| category() | Class<HeadMotion> | O(1) | Returns the class literal for HeadMotion, used for type registration and reflection within the builder framework. |

## Integration Patterns

### Standard Usage
This class is used implicitly by the asset loading system. A developer defines the sequence in a data file, and the system handles the builder's lifecycle automatically.

```java
// PSEUDO-CODE: How the asset system uses this builder internally

// 1. Deserializer creates the builder from an asset file
BuilderHeadMotionSequence builder = Deserializer.fromJson(assetFile, BuilderHeadMotionSequence.class);

// 2. The asset pipeline invokes the build method
BuilderSupport supportContext = new BuilderSupport(...);
HeadMotionSequence finalSequence = builder.build(supportContext);

// 3. The builder is now discarded, and the finalSequence is attached to an NPC
npc.setHeadMotion(finalSequence);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderHeadMotionSequence()` in gameplay code. This class is not designed for runtime configuration; it is a loading-time construct.
- **State Re-use:** Do not retain a reference to the builder after calling `build`. The builder's purpose is complete at that point. Modifying it further will have no effect on the already created HeadMotionSequence.
- **Concurrent Access:** Never pass a builder instance across threads or access it from an asynchronous task. The internal state is not protected by locks.

## Data Pipeline
This builder is a critical step in transforming static data into a usable runtime component.

> Flow:
> NPC Asset File (JSON) -> Deserialization Engine -> **BuilderHeadMotionSequence** -> `build()` call with BuilderSupport -> Immutable `HeadMotionSequence` -> NPC Behavior System

