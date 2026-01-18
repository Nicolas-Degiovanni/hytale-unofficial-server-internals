---
description: Architectural reference for BuilderActionTimer
---

# BuilderActionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Abstract Builder

## Definition
```java
// Signature
public abstract class BuilderActionTimer extends BuilderActionBase {
```

## Architecture & Concepts
BuilderActionTimer is an abstract base class that serves as a factory for creating actions that manipulate server-side NPC Timers. It is a critical component of the declarative NPC asset system, acting as the bridge between a static JSON configuration and the dynamic, stateful Timer system at runtime.

This class is not intended for direct use. Instead, concrete subclasses (e.g., a hypothetical BuilderActionStartTimer or BuilderActionResetTimer) implement the `getTimerAction` method to define a specific operation. The primary responsibility of BuilderActionTimer is to parse and hold a reference to the target Timer's name.

It leverages a `StringHolder` to store the timer's name, which allows for dynamic name resolution using an `ExecutionContext` at runtime. This design decouples the NPC definition from the live game state, enabling more flexible and reusable NPC behaviors.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's NPC asset deserialization pipeline when it encounters a JSON object corresponding to a timer action. It is never created manually by game logic developers.
- **Scope:** Transient. An instance of this class exists only during the asset loading and linking phase. Its purpose is to be configured from JSON and then used to produce a `Timer.TimerAction` object.
- **Destruction:** The builder object is eligible for garbage collection immediately after the NPC's behavior graph is fully constructed. It holds no persistent state and is not retained by the runtime system.

## Internal State & Concurrency
- **State:** The internal state, primarily the `name` field, is mutable during the configuration phase via the `readConfig` method. Once this phase is complete, the object should be treated as immutable. It holds configuration data, not live runtime state.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within the single-threaded context of the NPC asset loading system. Concurrent calls to `readConfig` or modification of its internal fields will result in undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderActionTimer | O(1) | Populates the builder's internal state from a JSON source. Validates that the "Name" field is a non-empty string. |
| getTimerAction() | Timer.TimerAction | O(1) | **Abstract.** Subclasses must implement this to return a concrete action to be performed on a Timer. |
| getTimer(BuilderSupport support) | Timer | O(log N) | Resolves the configured timer name into a live Timer instance using the provided runtime support context. Throws if the timer does not exist. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class via Java code. Instead, they define the action declaratively within an NPC's JSON asset file. The engine's asset loader instantiates the appropriate builder and configures it.

The following conceptual JSON snippet would trigger the creation and configuration of a concrete subclass of BuilderActionTimer:

```json
// Example from an NPC asset file
{
  "action": "StartTimer",
  "Name": "AttackCooldownTimer"
}
```

The system would then internally use the configured builder to link this action to the NPC's behavior tree.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** This class is abstract and cannot be instantiated. Instantiating concrete subclasses manually circumvents the asset pipeline and will fail, as they will not be properly configured.
- **State Re-Configuration:** Calling `readConfig` multiple times on the same instance can lead to an inconsistent or corrupted state. Builders are designed for a single, linear configuration flow.
- **Context-less Operation:** Invoking `getTimer` without a fully populated `BuilderSupport` context from the runtime engine will result in exceptions, as the timer name cannot be resolved.

## Data Pipeline
The flow of data from configuration to runtime execution is a key aspect of this component's design.

> Flow:
> NPC Asset JSON -> Server Asset Deserializer -> **BuilderActionTimer.readConfig()** -> Concrete Subclass.getTimerAction() -> NPC Behavior Graph -> Live Timer System

