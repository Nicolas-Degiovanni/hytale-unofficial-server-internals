---
description: Architectural reference for BuilderBodyMotionWanderBase
---

# BuilderBodyMotionWanderBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public abstract class BuilderBodyMotionWanderBase extends BuilderBodyMotionBase {
```

## Architecture & Concepts
BuilderBodyMotionWanderBase is an abstract base class that serves as a foundational component in the server-side NPC behavior system. Its primary role is to deserialize, validate, and temporarily store configuration parameters for any NPC movement instruction that involves a "wandering" pattern.

This class embodies the engine's data-driven design philosophy. It acts as an intermediary between raw JSON asset definitions and the concrete, in-game BodyMotion objects that control an NPC's actions. It does not produce a usable object itself; rather, it provides a shared configuration surface and validation logic for a family of concrete builder implementations.

A key architectural feature is its use of Holder objects (e.g., DoubleHolder, BooleanHolder) instead of primitive types for its internal state. This pattern decouples the static configuration (read from JSON) from the dynamic, in-game evaluation of those values. This allows NPC behaviors to be context-sensitive, where parameters like speed or turn angle can be resolved at runtime via a provided ExecutionContext.

## Lifecycle & Ownership
- **Creation:** An instance of a concrete subclass is created by the NPC asset loading pipeline when it encounters a corresponding motion component type in an NPC's behavior definition file (JSON). It is never instantiated directly in game logic.
- **Scope:** The object's lifetime is extremely short and confined to the asset parsing and object construction phase. It is a transient, single-use object.
- **Destruction:** The builder is eligible for garbage collection immediately after its `build` method has been called and the resulting BodyMotion instance has been returned to the caller. It holds no persistent references and is not managed by any service registry.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The primary purpose of the class is to be populated by the `readCommonConfig` method. The state consists of a collection of Holder objects that represent the un-resolved configuration parameters for the wander behavior.

- **Thread Safety:** This class is **not thread-safe** and must be treated as thread-confined. It is designed for synchronous, single-threaded use within the asset loading system. Concurrent calls to `readCommonConfig` or its getter methods would result in a corrupted internal state and unpredictable behavior.

## API Surface
The primary contract is for configuration and construction, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readCommonConfig(data) | Builder<BodyMotion> | O(N) | Parses a JsonElement, populating the internal Holder fields. N is the number of keys in the JSON object. Validates each parameter against predefined rules. |
| build(builderSupport) | BodyMotionWanderBase | O(1) | Abstract method intended for subclasses to construct a concrete BodyMotionWanderBase instance using the configured state. The base implementation is a no-op. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly via code. Instead, they define the parameters in a JSON asset file, which the server's asset system uses to instantiate and configure a concrete builder.

A concrete subclass, for example `BuilderBodyMotionWander`, would be implemented as follows:

```java
// Hypothetical concrete implementation
public class BuilderBodyMotionWander extends BuilderBodyMotionWanderBase {
    @Override
    public BodyMotionWanderBase build(@Nonnull BuilderSupport support) {
        // The base class build method is called to set common requirements
        super.build(support);

        // Construct the final object using resolved values from the Holder fields
        return new BodyMotionWander(this, support);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderBodyMotionWander()` in game logic. The NPC behavior system manages the lifecycle of these builders during asset loading.
- **Instance Re-use:** A builder instance is stateful and must not be reused to build multiple `BodyMotion` objects. Doing so would cause all subsequent objects to share the same initial configuration, which is almost never the desired outcome.
- **Configuration After Build:** Do not attempt to call `readCommonConfig` or modify the builder after `build` has been invoked. The builder's purpose is complete once the `BodyMotion` object is constructed.

## Data Pipeline
This class is a critical stage in the NPC behavior deserialization pipeline. It transforms declarative JSON data into a validated, intermediate representation before the final game object is created.

> Flow:
> NPC Behavior JSON Asset -> JSON Parser -> `JsonElement` -> **BuilderBodyMotionWanderBase.readCommonConfig** -> Internal Holder State -> `ConcreteBuilder.build()` -> `BodyMotionWander` Instance -> NPC Behavior Controller

