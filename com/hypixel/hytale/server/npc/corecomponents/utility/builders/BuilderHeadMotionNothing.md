---
description: Architectural reference for BuilderHeadMotionNothing
---

# BuilderHeadMotionNothing

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderHeadMotionNothing extends BuilderHeadMotionBase {
```

## Architecture & Concepts
The BuilderHeadMotionNothing class is a concrete implementation of the Builder design pattern, specialized for constructing NPC behavior components. It serves as a factory for creating `HeadMotionNothing` instances, which represent a "no-op" or null object behavior for an NPC's head movement.

Within the server's NPC architecture, behaviors like head motion are designed to be modular and data-driven. A central system deserializes an NPC's definition from an asset file and uses a corresponding builder class to construct each behavior component. BuilderHeadMotionNothing is the designated factory for the simplest case: when an NPC's head should remain static and perform no specific actions.

This approach is critical for system stability. By providing a concrete, non-null object that does nothing, the engine's core NPC update loop can operate on a homogeneous collection of `HeadMotion` components without requiring constant null-checks. This simplifies the game logic and prevents potential NullPointerExceptions for NPCs that lack a defined head motion behavior.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderHeadMotionNothing is created by the NPC asset loading system. This typically occurs when parsing an NPC's configuration data (e.g., a JSON or HOCON file) that specifies the head motion type as "nothing". It is never instantiated directly by game logic code.
- **Scope:** The builder's lifecycle is extremely short and confined to the NPC instantiation process. It exists only to be configured and to have its `build` method invoked.
- **Destruction:** The builder object is eligible for garbage collection immediately after the `build` method returns the final `HeadMotionNothing` component. It is a temporary, single-use object, and no long-term references to it are maintained.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no member fields and its behavior is entirely determined by its type. As a result, it is effectively immutable.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. Multiple threads can invoke the `build` method on a shared instance without risk of interference, as each call will produce a new, independent `HeadMotionNothing` object. In practice, however, it is almost always used within a single-threaded asset loading context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | HeadMotionNothing | O(1) | Constructs and returns a new instance of the HeadMotionNothing component. |
| getShortDescription() | String | O(1) | Returns a brief, human-readable description for use in tooling or logs. |
| getLongDescription() | String | O(1) | Returns a detailed, human-readable description for use in tooling or logs. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the development status of this component, indicating it is stable and ready for production use. |

## Integration Patterns

### Standard Usage
This builder is not intended for direct use by gameplay programmers. It is invoked by the underlying NPC factory or asset management system during entity creation. The system identifies the required behavior type from data and uses the corresponding builder to construct the component.

```java
// Hypothetical usage within an NPC Factory
// The factory would select this builder based on NPC definition data.
BuilderHeadMotionBase motionBuilder = new BuilderHeadMotionNothing();

// The builder is used to create the final runtime component.
HeadMotionNothing motionComponent = (HeadMotionNothing) motionBuilder.build(builderSupport);

// The component is then attached to the NPC.
npc.setHeadMotion(motionComponent);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation in Game Logic:** Never instantiate this builder directly within entity update logic or gameplay systems. NPC behaviors should be defined in data assets and loaded by the engine.
- **Retaining Builder References:** Do not store a reference to the builder after its `build` method has been called. It is a temporary object that should be discarded. Storing it is a memory leak.

## Data Pipeline
BuilderHeadMotionNothing acts as a transformation step in the data pipeline that converts declarative NPC definitions into live, in-game entities.

> Flow:
> NPC Definition Asset (e.g., JSON) -> Asset Deserializer -> **BuilderHeadMotionNothing** -> `build()` Invocation -> `HeadMotionNothing` Instance -> Attached to NPC Entity State

