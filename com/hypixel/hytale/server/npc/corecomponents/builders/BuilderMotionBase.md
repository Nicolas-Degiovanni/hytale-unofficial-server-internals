---
description: Architectural reference for BuilderMotionBase
---

# BuilderMotionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class BuilderMotionBase<T extends Motion> extends BuilderBase<T> {
```

## Architecture & Concepts
BuilderMotionBase is an abstract base class within the NPC asset definition framework. It serves as the foundational template for all builders responsible for constructing concrete *Motion* instructions from JSON configuration files. A Motion instruction is a component that dictates how an NPC physically moves within the world, such as walking, flying, or pathfinding.

Architecturally, this class acts as a policy enforcer at asset load-time. It ensures that any custom motion builder adheres to a common set of rules and validation logic, centralizing behavior that is universal to all NPC movements. Its primary role is to bridge the declarative JSON data with the engine's internal object representation of an NPC's behavior, specifically for the motion domain.

By inheriting from BuilderMotionBase, a concrete builder (e.g., BuilderWalk, BuilderFly) automatically gains common configuration parsing and, most importantly, critical validation logic that prevents unstable or illogical NPC behaviors from being loaded into the game.

### Lifecycle & Ownership
- **Creation:** Instances of BuilderMotionBase subclasses are created by a factory system during the server's asset loading phase. When the asset loader parses an NPC JSON file and encounters an instruction of a motion type, it requests the appropriate builder from the factory.
- **Scope:** Transient. The lifecycle of a builder object is extremely short, confined entirely to the duration of the asset parsing and validation process for a single NPC instruction.
- **Destruction:** The object is eligible for garbage collection immediately after the corresponding Motion object has been successfully constructed and integrated into the NPC's behavior definition. It holds no state that persists into the runtime game loop.

## Internal State & Concurrency
- **State:** Highly mutable. During its brief existence, a builder instance accumulates state from the parsed JSON data. This state is temporary and used solely for the construction of the final, immutable Motion object.
- **Thread Safety:** This class is not thread-safe and is not designed to be. The entire NPC asset loading pipeline is a single-threaded process. Concurrent access to a builder instance would result in a corrupted or unpredictable configuration state.

## API Surface
The public contract is designed for use by the asset loading system, not for general-purpose game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canRequireFeature() | boolean | O(1) | Confirms that motion instructions can declare dependencies on other engine features. Always returns true. |
| readCommonConfig(JsonElement) | Builder<T> | O(N) | Parses shared configuration from the JSON tree. Enforces that the instruction type is a valid motion. |
| isEnabled(ExecutionContext) | boolean | O(1) | A runtime check to see if the instruction is active. Hardcoded to true, making all motions enabled by default. |
| validate(...) | boolean | O(1) | Performs critical load-time validation. Its key function is to prevent a motion from being controlled by a sensor set to trigger only once. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. Developers create new NPC motions by extending it. The engine's asset loading system will then discover and use the concrete implementation.

```java
// Example of a concrete implementation
public class BuilderWalk extends BuilderMotionBase<MotionWalk> {
    // ... implementation for parsing walk-specific properties ...

    @Override
    public MotionWalk build(ExecutionContext context, Scope globalScope) {
        // ... construct and return a new MotionWalk object ...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate a subclass of BuilderMotionBase directly with `new`. The asset loading factory is solely responsible for its creation and lifecycle.
- **Runtime Usage:** This is a load-time class. Attempting to access or use a builder during the main game loop is a severe architectural violation and will fail.
- **Ignoring Validation Failures:** The `validate` method is a critical guard against invalid NPC configurations. A `false` return indicates a serious authoring error in the NPC JSON file that could lead to unpredictable runtime behavior or server crashes.

## Data Pipeline
BuilderMotionBase operates within the server's asset loading pipeline. It transforms declarative data into an executable object.

> Flow:
> NPC JSON File -> Asset Deserializer -> Builder Factory -> **BuilderMotionBase Subclass** -> Validation Logic -> Instantiated Motion Object -> NPC Behavior Tree

