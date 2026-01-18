---
description: Architectural reference for ReputationCompletion
---

# ReputationCompletion

**Package:** com.hypixel.hytale.builtin.adventure.objectivereputation
**Type:** Transient

## Definition
```java
// Signature
public class ReputationCompletion extends ObjectiveCompletion {
```

## Architecture & Concepts
The ReputationCompletion class is a concrete implementation of the **Strategy Pattern**, designed to handle a specific type of reward when a game objective is completed. It acts as a bridge between the generic Objective system and the specialized Reputation system.

Its sole responsibility is to grant a configured amount of reputation with a specific faction to all participants of an objective. This class is not a standalone service; it is a data-driven "handler" or "effector" that is instantiated and executed by a parent Objective object. The behavior of each ReputationCompletion instance is defined entirely by its corresponding ReputationCompletionAsset, making the objective reward system highly configurable and extensible without code changes.

## Lifecycle & Ownership
- **Creation:** An instance of ReputationCompletion is created by the server's objective loading mechanism. When an Objective asset is parsed and found to contain a "reputation" completion block, this class is instantiated and supplied with the corresponding ReputationCompletionAsset. It is then owned by the parent Objective instance.
- **Scope:** The lifecycle of a ReputationCompletion object is strictly bound to its parent Objective. It persists as long as the Objective is active in the game world.
- **Destruction:** The object is eligible for garbage collection when its parent Objective is unloaded or destroyed. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is effectively **immutable**. Its configuration is provided by the ReputationCompletionAsset during construction and does not change during its lifetime. The handle method is stateless, operating exclusively on the parameters it is given and the global state managed by other systems.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to be invoked exclusively from the server's main game loop thread as part of the objective completion sequence. The underlying systems it interacts with, such as the ComponentAccessor and ReputationPlugin, assume single-threaded access within a given game tick.

## API Surface
The public API is minimal, exposing only the logic required by the Objective system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(objective, accessor) | void | O(N) | Executes the reputation reward logic. N is the number of participants in the objective. Throws NullPointerException if arguments are null. |
| getAsset() | ReputationCompletionAsset | O(1) | Returns the asset that configures this completion handler. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in Java code. Instead, they define its behavior declaratively within an objective's asset file. The system then instantiates and invokes it automatically.

The following conceptual asset definition would result in the system creating and using a ReputationCompletion instance:

```json
{
  "id": "example.objective.winters_favor",
  "completion": {
    "type": "reputation",
    "reputationGroupId": "faction.winter.guard",
    "amount": 250
  }
}
```

The game engine's Objective system is responsible for calling the handle method when the objective is completed.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ReputationCompletion()`. Instances are managed entirely by the objective asset loader. Manually creating one circumvents its connection to a parent Objective and its history data.
- **Manual Invocation:** Do not call the `handle` method directly. This bypasses the state checks and lifecycle management of the parent Objective, potentially leading to rewards being granted multiple times or at incorrect moments.

## Data Pipeline
ReputationCompletion acts as a specific step in the data flow of an objective's completion. It reads configuration from its asset and writes changes to the global player and objective state.

> Flow:
> Game Event triggers Objective Completion -> Objective.complete() -> **ReputationCompletion.handle()** -> ReputationPlugin.changeReputation() -> Player Reputation State Update & ObjectiveHistoryData Update

