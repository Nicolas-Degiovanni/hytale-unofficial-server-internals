---
description: Architectural reference for BuilderActionNothing
---

# BuilderActionNothing

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient Builder

## Definition
```java
// Signature
public class BuilderActionNothing extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionNothing class is a concrete implementation of the Builder design pattern, specialized for constructing ActionNothing instances within the server-side NPC (Non-Player Character) behavior system.

Its primary function is to serve as a factory during the deserialization of NPC behavior trees from data assets, typically JSON files. Within this architecture, each type of NPC action (e.g., attack, move, wait) is represented by a corresponding builder class. The engine's configuration loader identifies the action type specified in the data and delegates instantiation to the appropriate builder.

BuilderActionNothing embodies the **Null Object pattern** for NPC actions. It produces a valid action object that performs no operation. This is a critical architectural component for creating placeholder behaviors, default states in a state machine, or terminal nodes in a behavior tree without introducing null checks or special-case logic throughout the NPC update loop. Its existence ensures that even an "empty" action is a first-class, valid citizen of the behavior system.

## Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by a higher-level factory or asset loader, such as an ActionBuilderRegistry. This process is triggered when the server parses an NPC configuration file and encounters an action definition of type "nothing". Developers should never instantiate this class directly.
-   **Scope:** Extremely short-lived and **transient**. An instance of BuilderActionNothing typically exists only for the duration of a single `build` operation. It is created, used to produce an ActionNothing object, and then immediately discarded.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the `build` method returns. It is not retained by any part of the engine's state.

## Internal State & Concurrency
-   **State:** **Stateless and Immutable**. The class contains no member fields and its behavior is constant. The `readConfig` method is a no-op, reinforcing that this action has no configurable parameters. All instances of BuilderActionNothing are functionally identical.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, a single instance could be used across multiple threads without any risk of race conditions or data corruption. No synchronization mechanisms are necessary.

## API Surface
The public API is minimal, focused exclusively on its role within the engine's builder framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionNothing | O(1) | Constructs and returns a new ActionNothing instance. |
| readConfig(JsonElement) | BuilderActionNothing | O(1) | No-op. Fulfills the BuilderActionBase contract but ignores input, as this action is not configurable. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling or debugging. |
| getLongDescription() | String | O(1) | Provides a more detailed description for tooling or debugging. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns a status indicating the production-readiness of this component. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is incorrect. It is designed to be invoked transparently by the NPC asset loading system. The following pseudo-code illustrates its intended place in the engine.

```java
// Engine-level pseudo-code for loading an NPC action from a JSON definition
//
// JSON Snippet:
// { "type": "nothing", "data": {} }

// Engine's Action Factory
BuilderActionBase builder = actionRegistry.getBuilderForType("nothing"); // Returns a BuilderActionNothing instance
builder.readConfig(actionJson.get("data"));
Action runtimeAction = builder.build(builderSupport); // Creates an ActionNothing instance

// The runtimeAction is then added to the NPC's behavior tree
npc.getBehaviorTree().setRootAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderActionNothing()`. The NPC system relies on a central registry to map action types from data files to the correct builder classes. Bypassing this registry will break configuration loading.
-   **Stateful Subclassing:** Do not extend this class to add state. Its contract is to be a stateless factory for a no-op action. Introducing state violates the core design and can lead to unpredictable behavior in the NPC system.
-   **Meaningful Configuration:** Do not attempt to pass meaningful data to the `readConfig` method. It is a no-op by design and will silently discard any provided configuration.

## Data Pipeline
BuilderActionNothing acts as a transformation step in the pipeline that converts declarative NPC definitions into executable, in-memory objects.

> Flow:
> NPC Definition Asset (JSON) -> Server Asset Parser -> Action Builder Registry -> **BuilderActionNothing** -> ActionNothing Instance -> Live NPC Behavior Tree

