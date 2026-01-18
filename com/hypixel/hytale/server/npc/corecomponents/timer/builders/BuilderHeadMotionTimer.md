---
description: Architectural reference for BuilderHeadMotionTimer
---

# BuilderHeadMotionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderHeadMotionTimer extends BuilderMotionTimer<HeadMotion> {
```

## Architecture & Concepts
The BuilderHeadMotionTimer is a transient, intermediate object used exclusively within the server-side NPC asset loading pipeline. It is a concrete implementation of the Builder Pattern, specialized for constructing a HeadMotionTimer component from a raw asset definition.

Its primary role is to act as a blueprint during the deserialization process. When the server parses an NPC definition file (e.g., a JSON or HOCON file), it creates an object graph of builder classes, including this one, that mirrors the structure of the asset.

A critical architectural feature is its use of the BuilderObjectReferenceHelper. This class does not hold a direct instance of a HeadMotion. Instead, it holds a *reference* to one, which is resolved later using the BuilderSupport context. This decouples the asset definition from the instantiation logic, allowing for a two-pass loading system. This system is essential for managing complex dependencies between different NPC components without enforcing a strict declaration order within asset files.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the asset deserialization framework when it encounters a head motion timer definition within an NPC asset. It is never created manually by developers.
- **Scope:** Its lifecycle is extremely short and confined to the NPC instantiation process. It exists only from the moment the asset is parsed until the final HeadMotionTimer object is built.
- **Destruction:** The builder object is eligible for garbage collection immediately after its `build` method has been invoked by the parent factory or builder. It holds no persistent state and is not retained.

## Internal State & Concurrency
- **State:** The class is stateful but intended for single-use. Its primary state is the unresolved reference to a HeadMotion asset, managed by the internal BuilderObjectReferenceHelper. This state is populated during deserialization and is considered immutable after that point.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. The entire NPC asset building process, including the BuilderSupport context it relies on, is designed to be executed within a single, synchronous thread during server startup or asset reloading. Concurrent modification would lead to unpredictable behavior and race conditions during reference resolution.

## API Surface
The public API is minimal, designed for internal consumption by the asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | HeadMotionTimer | O(L) | Constructs a HeadMotionTimer. Resolves the internal HeadMotion reference using the provided context. Returns null if the reference cannot be resolved. |
| category() | Class<HeadMotion> | O(1) | Returns the class literal for HeadMotion, used for type registration within the builder system. |

*Complexity Note: O(L) denotes a lookup operation within the BuilderSupport context, which is typically O(1) or O(log N).*

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked transparently by the NPC asset loading framework. The conceptual flow is as follows:

```java
// Conceptual example of the framework's internal logic.
// A developer would NOT write this code.

// 1. Deserializer creates the builder from an asset file.
BuilderHeadMotionTimer builder = deserializer.load("npc_asset.json");

// 2. The NPC factory invokes build(), providing the resolution context.
BuilderSupport context = ...; // The context containing all loaded assets.
HeadMotionTimer timer = builder.build(context);

// 3. The resulting timer is attached to the NPC.
npc.addComponent(timer);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderHeadMotionTimer()`. The object is useless without being populated by the deserialization framework.
- **State Mutation:** Do not attempt to modify the builder's state after it has been created by the deserializer. Its internal reference is meant to be configured only once.
- **Re-use:** Do not call the `build` method more than once on a single instance. Builders are single-use, transient objects.

## Data Pipeline
The BuilderHeadMotionTimer is a key transformation step in the data pipeline that converts a static asset file into a live game object component.

> Flow:
> NPC Asset File (JSON) -> Deserialization Engine -> **BuilderHeadMotionTimer Instance** -> NPC Factory (`build` call) -> Live HeadMotionTimer Instance -> Attached to NPC Entity

---

