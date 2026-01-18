---
description: Architectural reference for BuilderActionTimerPause
---

# BuilderActionTimerPause

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTimerPause extends BuilderActionTimer {
```

## Architecture & Concepts
The BuilderActionTimerPause class is a specialized, concrete implementation of the Builder pattern. It functions as a factory for a single, specific type of runtime object: an ActionTimer configured to perform a **pause** operation.

Within the Hytale server's NPC (Non-Player Character) behavior system, this class acts as a bridge between the static data representation of an NPC's logic (e.g., a JSON asset file) and the live, executable game objects. Its sole responsibility is to translate a declarative "pause timer" instruction from an asset into a functional ActionTimer instance that can be integrated into an NPC's state machine or behavior tree.

This pattern decouples the asset loading and parsing system from the complexities of the runtime ActionTimer's implementation, allowing for clean separation of concerns. The existence of this class implies a corresponding set of builders for other timer actions, such as *start* or *reset*.

### Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset pipeline during the deserialization and build process. The asset loader identifies a "pause timer" action in the NPC definition file and creates an instance of BuilderActionTimerPause to represent it.
- **Scope:** Ephemeral and short-lived. An instance of this class exists only for the duration of the asset-to-object build phase. It holds no persistent state and is not retained by any long-term system.
- **Destruction:** The object is immediately eligible for garbage collection after its `build` method has been invoked and the resulting ActionTimer product has been returned to the caller.

## Internal State & Concurrency
- **State:** This class is stateless and effectively immutable. Its behavior is determined entirely by its type and the hardcoded return value of its `getTimerAction` method. It does not cache data or maintain any internal state across method calls.
- **Thread Safety:** BuilderActionTimerPause is inherently thread-safe. As a stateless factory, multiple threads can safely invoke the `build` method, even on a shared instance, without risk of race conditions or data corruption. In practice, the asset system likely creates a new instance for each build operation.

## API Surface
The public contract is minimal, focusing exclusively on object creation and metadata retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTimer | O(1) | Factory method. Instantiates and returns a new ActionTimer configured with the PAUSE action. |
| getTimerAction() | Timer.TimerAction | O(1) | Returns the constant Timer.TimerAction.PAUSE. This is the primary mechanism for configuring the built product. |
| getShortDescription() | String | O(1) | Provides a human-readable string for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides a more detailed human-readable string. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is an internal component of the asset build system. The system would use it as follows, where `npcActionBuilder` is an instance of this class created from an asset file.

```java
// The asset pipeline provides the builder instance
BuilderActionTimer npcActionBuilder = asset.getTimerActionBuilder();

// The build support context provides necessary dependencies
BuilderSupport support = new BuilderSupport(...);

// The builder creates the final runtime object
ActionTimer runtimeAction = npcActionBuilder.build(support);

// The runtimeAction is then added to the NPC's behavior controller
npc.getBehaviorController().addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderActionTimerPause()`. The NPC asset system is solely responsible for the lifecycle of builder objects. Manual creation bypasses the asset pipeline and can lead to unconfigured or invalid runtime objects.
- **Subclassing:** Do not extend this class. It is a final, concrete implementation for a specific action. To define a new timer action, create a new, separate class that extends BuilderActionTimer.

## Data Pipeline
This builder is a key transformation step in the pipeline that converts static NPC data into live server-side game logic.

> Flow:
> NPC Behavior Asset (JSON) -> Asset Deserializer -> **BuilderActionTimerPause** -> `build()` -> ActionTimer (Runtime Object) -> NPC Behavior Controller

