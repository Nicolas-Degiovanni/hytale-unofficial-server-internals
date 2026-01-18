---
description: Architectural reference for BuilderBodyMotionNothing
---

# BuilderBodyMotionNothing

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Factory

## Definition
```java
// Signature
public class BuilderBodyMotionNothing extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionNothing class is a factory component within the server-side NPC behavior system. It embodies the **Builder Pattern** to construct a specific type of NPC motion component: BodyMotionNothing.

Its primary architectural role is to provide a concrete, "do nothing" implementation for an NPC's body motion. This is a critical piece of the behavior system, representing a null or idle state. Rather than using null references, which can lead to runtime errors, the system uses the product of this builder—BodyMotionNothing—to explicitly define a state where no motion should occur.

This class is not a runtime component attached to an active NPC entity. Instead, it serves as a blueprint or configuration object used by the NPC asset loading pipeline. When the server deserializes an NPC definition from a file, it instantiates this builder to later construct the final, immutable BodyMotionNothing component that will be used by live NPCs.

## Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset management system during server initialization or when NPC definition files are loaded. The system likely uses reflection or a service locator to discover and register all available builder types, including this one.
- **Scope:** The builder instance is typically a singleton within the scope of the asset registry. It persists for the entire server session, or until a hot-reload of NPC assets occurs. It is not tied to an individual NPC entity's lifecycle.
- **Destruction:** The object is garbage collected when the server shuts down or when its containing asset registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is defined entirely by its methods, which return constant values.
- **Thread Safety:** The class is inherently **thread-safe**. Its immutability guarantees that it can be safely accessed and used by multiple threads concurrently during the NPC asset loading process without requiring any synchronization or locks. The build method produces a new, distinct object on each invocation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionNothing | O(1) | Factory method. Constructs and returns a new instance of BodyMotionNothing. |
| getShortDescription() | String | O(1) | Provides a human-readable name for this behavior, used in tooling or logs. |
| getLongDescription() | String | O(1) | Provides a detailed human-readable description for this behavior. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the development status of this component, indicating it is stable and ready for use. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in standard game logic. It is invoked by the underlying NPC asset pipeline. A game designer would typically reference this builder declaratively in an NPC's definition file.

The system would then use the builder as follows:

```java
// Conceptual example of the asset loader's internal logic
// Note: This is a system-level operation, not typical user code.

// 1. The loader instantiates the builder based on the NPC definition file.
BuilderBodyMotionBase builder = new BuilderBodyMotionNothing();

// 2. The builder is used to construct the actual runtime component.
BodyMotionNothing motionComponent = (BodyMotionNothing) builder.build(builderSupport);

// 3. The component is attached to the NPC's behavior tree.
npc.getBehaviorController().setBodyMotion(motionComponent);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation in Game Logic:** Never instantiate this builder directly within gameplay code (e.g., quest scripts, combat logic). The NPC definition pipeline is the sole owner of this process. Interacting with builders at runtime violates the separation of data (definitions) and logic (live entities).
- **Storing Builder References:** Do not cache or store references to this builder in long-lived game objects. The runtime logic should only ever interact with the *product* of the builder (the BodyMotionNothing instance), not the builder itself.

## Data Pipeline
This builder acts as a transformation step in the data pipeline that converts static NPC definitions into live, in-game entities.

> Flow:
> NPC Definition File (JSON/HOCON) -> Asset Deserializer -> **BuilderBodyMotionNothing.build()** -> BodyMotionNothing Instance -> Attached to NPC Entity State

