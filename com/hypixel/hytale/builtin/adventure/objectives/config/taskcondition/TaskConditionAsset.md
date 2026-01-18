---
description: Architectural reference for TaskConditionAsset
---

# TaskConditionAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.taskcondition
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class TaskConditionAsset {
```

## Architecture & Concepts
The TaskConditionAsset is an abstract base class that defines the contract for all objective conditions within the Adventure Mode system. It serves as a polymorphic template for specific, concrete conditions such as checking a player's inventory, location, or other world states.

This class is a cornerstone of Hytale's data-driven design philosophy for quests and objectives. Its primary architectural feature is the static **CodecMapCodec**, which acts as a factory and deserializer. This codec allows game designers to specify objective conditions using simple string identifiers (e.g., "SoloInventory") within asset files. At runtime, the engine uses this codec to look up and instantiate the corresponding concrete Java class (e.g., SoloInventoryCondition), effectively decoupling the quest data from the game logic.

This pattern is a variation of the **Strategy Pattern**, where each concrete subclass encapsulates a specific algorithm for checking and consuming a condition. The engine's objective processing system interacts with the abstract TaskConditionAsset interface, ignorant of the underlying implementation details.

## Lifecycle & Ownership
- **Creation:** Instances of TaskConditionAsset are **never instantiated directly** using the new keyword. They are created exclusively by the Hytale serialization framework via the static CODEC field during the loading of parent assets, such as an AdventureObjectiveAsset. The engine reads a "Type" field from the asset data and uses it to resolve the correct subclass to instantiate.

- **Scope:** The lifetime of a TaskConditionAsset instance is tied to its parent asset. These objects are effectively immutable configuration data. They persist in memory as long as the quest or objective they belong to is loaded.

- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup when their parent asset is unloaded from memory, for example, when a zone is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** A TaskConditionAsset is designed to be **Immutable and Stateless**. It serves as a template for logic and should not contain any mutable fields that change during gameplay. All stateful operations are performed on the external game state (the EntityStore) passed into its methods.

- **Thread Safety:** This class is inherently **Thread-Safe**. Because instances are immutable, their methods can be safely invoked from any thread without external synchronization.
    **Warning:** While the asset itself is thread-safe, the methods operate on the ComponentAccessor and EntityStore, which are not. The calling system, typically the main server thread, is responsible for ensuring safe access to the game world state.

## API Surface
The public contract is defined by two core abstract methods that must be implemented by all concrete condition types.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isConditionFulfilled(accessor, entity, participants) | boolean | Varies | Evaluates the game state to determine if the condition is met for the given participants. This is a read-only operation. |
| consumeCondition(accessor, entity, participants) | void | Varies | Executes a state-mutating action to "consume" the resources or state required by the condition. Throws if the condition is not met. |
| equals(obj) | boolean | Varies | Subclasses must implement value-based equality. |
| hashCode() | int | Varies | Subclasses must implement a hash code consistent with their equals implementation. |

## Integration Patterns

### Standard Usage
The primary interaction pattern for a developer is not to *call* this class, but to *extend* it to create a new, custom objective condition. This involves implementing the abstract methods and registering the new class with a codec.

```java
// 1. Define the concrete condition class
public class MyCustomCondition extends TaskConditionAsset {
    public static final Codec<MyCustomCondition> CODEC = ...; // Define a codec

    // Asset fields defined here, e.g., requiredItem, requiredAmount

    @Override
    public boolean isConditionFulfilled(...) {
        // Implement logic to check player inventory or world state
        return true;
    }

    @Override
    public void consumeCondition(...) {
        // Implement logic to remove items or alter world state
    }
    
    // Implement equals and hashCode
}

// 2. Register the new condition type with the engine, typically in a static initializer
// This step is critical for the engine to recognize the "MyCustomCondition" type in assets.
TaskConditionAsset.CODEC.register("MyCustomCondition", MyCustomCondition.class, MyCustomCondition.CODEC);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MyCustomCondition()`. The engine's asset loading and serialization pipeline is solely responsible for creating these objects. Direct instantiation bypasses the data-driven framework.
- **Mutable State:** Do not add mutable fields to subclasses. A condition asset must represent static configuration data. Storing runtime state on the asset instance will lead to severe concurrency issues and incorrect behavior for different players.
- **Assuming Order:** Do not call `consumeCondition` without first verifying the state with `isConditionFulfilled`. While not enforced by the interface, systems are designed with the expectation that consumption only occurs on a fulfilled condition.

## Data Pipeline
The TaskConditionAsset is a key component in the data flow from game assets to live game logic.

> Flow:
> Adventure Objective JSON Asset -> Asset Loading Service -> **TaskConditionAsset.CODEC** -> Concrete TaskConditionAsset Instance -> Objective Processing System -> `isConditionFulfilled()` check on main game thread

