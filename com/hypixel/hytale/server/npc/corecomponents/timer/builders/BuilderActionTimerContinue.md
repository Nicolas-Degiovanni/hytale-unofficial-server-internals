---
description: Architectural reference for BuilderActionTimerContinue
---

# BuilderActionTimerContinue

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTimerContinue extends BuilderActionTimer {
```

## Architecture & Concepts
The BuilderActionTimerContinue class is a specialized, stateless factory within the server's NPC asset-to-entity pipeline. It serves as a concrete implementation of the Builder pattern, designed to translate a declarative instruction from an NPC behavior asset into a functional, runtime game component.

Its sole responsibility is to represent the **CONTINUE** timer action. During server startup or dynamic NPC spawning, the asset loading system parses behavior definitions (likely JSON or a similar format). When the parser encounters an instruction to "continue" a timer, it instantiates a BuilderActionTimerContinue object. This object is then immediately used to construct an ActionTimer component, which is configured with the specific `Timer.TimerAction.CONTINUE` behavior.

This class acts as a critical bridge between static data (the asset file) and the live, executable logic of an NPC's brain. It ensures that the abstract concept of "continue a timer" is correctly materialized into a component that the NPC's core logic can execute during the game loop.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the NPC asset deserialization system when processing a timer action of type CONTINUE. This class is not intended for direct manual instantiation by developers.
- **Scope:** Extremely short-lived and transient. An instance of this class exists only for the duration of the `build` method call within the asset loading sequence.
- **Destruction:** The object is eligible for garbage collection immediately after the `ActionTimer` it produces is created and registered with the parent NPC. It holds no persistent state and is not retained by any system.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. This class contains no member fields and its behavior is entirely defined by its type and method overrides. Its purpose is to provide a constant value (`Timer.TimerAction.CONTINUE`) to the ActionTimer constructor.
- **Thread Safety:** Inherently thread-safe due to its stateless and immutable nature. It can be safely used in a multi-threaded asset loading environment without synchronization, as it produces a new, distinct ActionTimer instance on every `build` call.

## API Surface
The public contract is minimal, focused entirely on its role as a factory and metadata provider for tooling.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTimer | O(1) | Factory method. Constructs and returns a new ActionTimer instance configured to perform a CONTINUE action. |
| getTimerAction() | Timer.TimerAction | O(1) | Returns the static enum value `Timer.TimerAction.CONTINUE`. |
| getShortDescription() | String | O(1) | Provides a human-readable string for tooling or debugging, "Continue a timer". |
| getLongDescription() | String | O(1) | Provides a detailed human-readable string for tooling or debugging. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. Instead, a developer defines the behavior declaratively in an NPC asset file. The engine's asset loader uses this class internally to realize the defined behavior.

A conceptual NPC behavior asset might look like this:
```json
// Example: part of an NPC behavior definition
{
  "onTakeDamage": [
    {
      "action": "startTimer",
      "timerId": "evade_cooldown",
      "duration": 5.0
    }
  ],
  "onHealed": [
    {
      "action": "timer",
      "timerId": "evade_cooldown",
      "timerAction": "continue" // This line causes the system to use BuilderActionTimerContinue
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new BuilderActionTimerContinue()`. The asset pipeline is solely responsible for the lifecycle of builder objects. Manually creating one bypasses the asset loading context and will result in non-functional components.
- **Misuse as a Component:** This is a builder, not a runtime component. Attempting to store or manage an instance of BuilderActionTimerContinue within an NPC's state will have no effect. The object's only purpose is to be used once to build the real ActionTimer.

## Data Pipeline
The class functions as a single, critical step in the data transformation pipeline from a static asset to a live game object.

> Flow:
> NPC Behavior Asset (JSON) -> Asset Deserializer -> **BuilderActionTimerContinue** -> build() -> ActionTimer (Instance) -> NPC Component Registry

