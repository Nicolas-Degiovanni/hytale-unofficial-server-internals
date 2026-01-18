---
description: Architectural reference for StoreSystem
---

# StoreSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class StoreSystem<ECS_TYPE> extends System<ECS_TYPE> {
```

## Architecture & Concepts
The StoreSystem class is an abstract template that forms a foundational contract within the Entity-Component-System (ECS) architecture. It represents a specialized type of System that requires direct awareness of and interaction with its parent **Store**. The Store is the central container that manages the lifecycle of all entities and components within a given context, such as a game world or a UI scene.

By extending StoreSystem, a concrete system implementation gains critical lifecycle hooks that are invoked by the Store itself. This implements an Inversion of Control (IoC) pattern, where the framework (the Store) manages the system's lifecycle, rather than the system managing itself. This design ensures that systems can safely acquire necessary resources upon registration and release them upon deregistration, preventing memory leaks and maintaining a predictable state.

The primary purpose of this class is to provide a formal contract for systems that need to perform setup or teardown logic that is dependent on the Store. For example, a system might need to pre-cache certain components or register itself with other services managed by the Store.

### Lifecycle & Ownership
The lifecycle of a StoreSystem is strictly managed by the ECS framework and is inextricably tied to the lifecycle of the Store it is registered with.

-   **Creation:** Concrete implementations of StoreSystem are instantiated by the component framework, typically when a world or a specific game state is initialized. They are then registered with a corresponding Store instance. Direct instantiation by client code is a critical anti-pattern.
-   **Scope:** An instance of a StoreSystem persists for exactly as long as it remains registered within its parent Store. Its lifetime is equivalent to the scope of the Store.
-   **Destruction:** When the parent Store is destroyed (e.g., a player leaves a server, or a world is unloaded), it iterates through its registered systems and invokes the onSystemRemovedFromStore callback. After this method completes, the system is deregistered and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** As an abstract class, StoreSystem is stateless. However, concrete subclasses are almost always stateful. They are expected to hold a reference to the Store provided during the onSystemAddedToStore callback and use it to query for entities and components during their processing phase.

-   **Thread Safety:** This base class is inherently thread-safe. Subclasses, however, are **not** guaranteed to be thread-safe and should be assumed to operate on a single thread, typically the main game loop thread. The ECS framework ensures that systems are updated in a deterministic, sequential order. Any multi-threaded access to a system or the components it manages must be handled with extreme care, using explicit locking or thread-safe data structures.

## API Surface
The public API consists of abstract lifecycle callbacks intended for framework invocation, not direct user calls.

| Symbol                               | Type | Complexity | Description                                                                                                                              |
| :----------------------------------- | :--- | :--------- | :--------------------------------------------------------------------------------------------------------------------------------------- |
| onSystemAddedToStore(Store)          | void | O(N)       | **Framework Callback.** Invoked once when the system is registered with a Store. N is the complexity of the system's initialization logic. |
| onSystemRemovedFromStore(Store)      | void | O(M)       | **Framework Callback.** Invoked once when the system is deregistered from a Store. M is the complexity of the system's cleanup logic.      |

## Integration Patterns

### Standard Usage
A developer does not instantiate or call methods on StoreSystem directly. Instead, they create a new class that extends it and implements the required lifecycle methods. The framework handles the rest.

```java
// Example of a concrete system implementation
public class PlayerHealthSystem extends StoreSystem<GameEntity> {

    private Store<GameEntity> parentStore;

    // This method is called by the framework upon registration.
    @Override
    public void onSystemAddedToStore(@Nonnull Store<GameEntity> store) {
        this.parentStore = store;
        // Perform initial setup, like caching component mappers
        System.out.println("PlayerHealthSystem has been added to the store.");
    }

    // This method is called by the framework upon removal.
    @Override
    public void onSystemRemovedFromStore(@Nonnull Store<GameEntity> store) {
        // Release resources, nullify references
        this.parentStore = null;
        System.out.println("PlayerHealthSystem has been removed from the store.");
    }

    // Other system logic, like an update() method, would go here.
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Never call onSystemAddedToStore or onSystemRemovedFromStore directly. These are lifecycle hooks exclusively managed by the parent Store. Manually invoking them can lead to a corrupted state or resource leaks.
-   **Assuming Immediate Availability:** Do not attempt to access the parent Store within the constructor of a subclass. The Store reference is only guaranteed to be valid after onSystemAddedToStore has been called.
-   **Ignoring Lifecycle:** Failing to release resources or nullify references in onSystemRemovedFromStore can lead to memory leaks, as the system object may be held in memory by other objects that are not being cleaned up.

## Data Pipeline
StoreSystem is a structural component and does not participate in a data processing pipeline itself. Instead, it defines the control flow for the initialization and destruction of systems that do.

> Control Flow:
> Store Initialization -> `Store.addSystem(new MySystem())` -> Framework invokes **`MySystem.onSystemAddedToStore(Store)`** -> System is now active and participates in the game loop -> Store Destruction -> Framework invokes **`MySystem.onSystemRemovedFromStore(Store)`** -> System is destroyed

