---
description: Architectural reference for BuilderActionTimerRestart
---

# BuilderActionTimerRestart

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionTimerRestart extends BuilderActionTimer {
```

## Architecture & Concepts
The BuilderActionTimerRestart class is a concrete implementation of the **Builder Pattern**, specifically designed for the server-side NPC (Non-Player Character) behavior system. Its sole responsibility is to construct an ActionTimer instance that is pre-configured to perform a **RESTART** operation on a target Timer.

This class is a key component in the engine's data-driven design. It allows NPC behaviors, which are defined in external asset files (e.g., JSON or HOCON), to be decoupled from the underlying Java implementation. A higher-level asset loading system discovers and instantiates this builder based on a string identifier in the asset file. This approach enables designers to specify complex NPC logic without writing code, while the engine uses builders like this one to translate that data into executable runtime objects.

In essence, BuilderActionTimerRestart acts as a specialized factory, encapsulating the creation logic for one specific type of timer action within the broader NPC AI framework.

### Lifecycle & Ownership
- **Creation:** Instances of BuilderActionTimerRestart are not meant to be long-lived. They are typically instantiated on-demand by a higher-level system, such as an NPC asset parser or a behavior tree compiler, when it encounters a "restart timer" action definition.
- **Scope:** The object's lifetime is extremely short and transient. It exists only for the duration of the `build` method call.
- **Destruction:** After the `build` method returns the configured ActionTimer, the BuilderActionTimerRestart instance is no longer referenced and becomes eligible for garbage collection. It does not persist in memory.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is defined entirely by its type and overridden methods.
- **Thread Safety:** As a stateless object, BuilderActionTimerRestart is inherently **thread-safe**. The `build` method can be invoked by multiple threads concurrently without risk of data corruption or race conditions.

## API Surface
The public API is minimal and focused on metadata and object construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTimer | O(1) | Constructs and returns a new ActionTimer configured with the RESTART action. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for use in tooling or logs. |
| getLongDescription() | String | O(1) | Provides a more detailed description for use in editor tooltips or documentation. |
| getTimerAction() | Timer.TimerAction | O(1) | Returns the hardcoded Timer.TimerAction.RESTART enum, which is the core logic this builder provides. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay logic. It is an internal component of the asset-to-runtime pipeline. A higher-level system consumes it as part of a factory or registry pattern.

```java
// Hypothetical usage within an NPC asset loading system

// 1. A registry maps action strings from a data file to builder classes.
Map<String, Class<? extends BuilderActionTimer>> registry = ...;
registry.put("restart_timer", BuilderActionTimerRestart.class);

// 2. The loader instantiates the builder via reflection.
BuilderActionTimer builder = registry.get("restart_timer").newInstance();

// 3. The builder is used to create the runtime action component.
// The resulting 'action' is then added to an NPC's behavior tree.
ActionTimer action = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually instantiate this class in game systems. NPC behaviors should be defined in data assets. The engine's loading pipeline is responsible for selecting and using the correct builder.
- **Stateful Subclassing:** Do not extend this class to add state. The builder pattern in this context relies on stateless, transient objects. Adding state would violate its design contract and is not thread-safe.

## Data Pipeline
This builder sits at the translation point between declarative NPC configuration and executable server-side objects.

> Flow:
> NPC Behavior Asset (JSON/HOCON) -> Asset Parser -> **BuilderActionTimerRestart** -> ActionTimer Instance -> NPC Behavior Tree -> Runtime Execution

