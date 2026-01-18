---
description: Architectural reference for ObjectiveCompletion
---

# ObjectiveCompletion

**Package:** com.hypixel.hytale.builtin.adventure.objectives.completion
**Type:** Abstract Strategy

## Definition
```java
// Signature
public abstract class ObjectiveCompletion {
```

## Architecture & Concepts
The ObjectiveCompletion class is an abstract base class that defines the contract for objective completion logic within the adventure mode system. It embodies the **Strategy Pattern**, decoupling an Objective from the specific conditions required to complete it.

This class is not a service or a manager. Instead, it represents a single, stateless strategy for evaluating whether an Objective's criteria have been met. Each concrete implementation, such as a hypothetical *KillEntityCompletion* or *CollectItemCompletion*, provides a specific `handle` method that contains the actual game logic for that condition.

The core design principle is to make Objectives agnostic of their completion mechanics. An Objective simply holds a reference to an ObjectiveCompletion instance and delegates the completion check to it. This allows for a highly extensible system where new completion types can be added without modifying the core Objective class. The behavior of each strategy is driven by its corresponding configuration, the ObjectiveCompletionAsset, which is injected upon creation.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated by the objective system when an Objective is created from its configuration assets. The corresponding ObjectiveCompletionAsset is passed into the constructor, binding the logic to its data.
- **Scope:** The lifecycle of an ObjectiveCompletion instance is strictly tied to the Objective that owns it. It exists for as long as its parent Objective is active in the game world.
- **Destruction:** The object is marked for garbage collection when its owning Objective is completed, failed, or otherwise removed from the system. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The base class is effectively immutable. Its only state is the `asset` field, which is final and injected at construction. This asset contains the configuration data that drives the strategy's logic. Concrete subclasses are expected to be stateless and should not cache world-state information.
- **Thread Safety:** This class is not thread-safe by design. The `handle` method is expected to interact with core game state via the ComponentAccessor and EntityStore. All invocations of `handle` **must** occur on the main server thread to prevent race conditions and world-state corruption. Concurrent access is a critical anti-pattern.

## API Surface
The public contract is minimal, focusing entirely on configuration access and the primary strategy method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsset() | ObjectiveCompletionAsset | O(1) | Returns the immutable asset configuration for this completion strategy. |
| handle(Objective, ComponentAccessor) | void | Varies | Abstract method. Executes the completion logic. Complexity is implementation-dependent. |

## Integration Patterns

### Standard Usage
Developers do not typically instantiate or call this class directly. The primary interaction is to create a concrete subclass that implements a new type of completion logic. The system will then manage its lifecycle.

```java
// Example of a concrete implementation
public class KillEntityCompletion extends ObjectiveCompletion {

    public KillEntityCompletion(ObjectiveCompletionAsset asset) {
        super(asset);
    }

    @Override
    public void handle(Objective objective, ComponentAccessor<EntityStore> world) {
        // Custom logic to check if the target entity from the asset
        // has been killed by the player associated with the objective.
        // If conditions are met, this method would update the
        // parent Objective's state.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Do not call the `handle` method directly. It is designed to be invoked by the parent Objective or the objective management system in response to relevant game events.
- **Stateful Implementations:** Avoid storing mutable game state within a subclass. The strategy should be a pure function of its configuration and the provided world state. Caching entity references or player state can lead to desynchronization and memory leaks.
- **Cross-Thread Access:** Never invoke `handle` from an asynchronous task or a thread other than the main server thread. Doing so will lead to world corruption.

## Data Pipeline
The ObjectiveCompletion class acts as a processor within the larger objective update pipeline. It does not ingest data directly but is invoked by its owner to evaluate game state.

> Flow:
> Game Event (e.g., EntityDeathEvent) -> ObjectiveSystem -> Objective.onEvent() -> **ObjectiveCompletion.handle()** -> Objective.updateState()

