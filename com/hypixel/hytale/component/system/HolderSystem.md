---
description: Architectural reference for HolderSystem
---

# HolderSystem

**Package:** com.hypixel.hytale.component.system
**Type:** Component System (Abstract Base)

## Definition
```java
// Signature
public abstract class HolderSystem<ECS_TYPE> extends System<ECS_TYPE> implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts
The HolderSystem is a foundational abstract class within Hytale's Entity-Component-System (ECS) architecture. It establishes a reactive, event-driven contract for systems that must respond immediately to the presence or absence of entities matching a specific component query.

Unlike systems that iterate over a collection of entities each game tick, a HolderSystem acts as a direct listener to the ECS data layer. The core engine automatically invokes its methods when an entity's component composition changes to either match or no longer match the system's criteria. This makes it highly efficient for managing setup and teardown logic, such as registering a physics body when a PhysicsComponent is added or unregistering it upon removal.

The generic parameter, ECS_TYPE, is the key link that associates a concrete implementation of this class with a specific component type it is designed to manage.

## Lifecycle & Ownership
-   **Creation:** Concrete implementations of HolderSystem are not instantiated manually. They are discovered and instantiated by a central service, such as a SystemRegistry or WorldInitializer, during the engine's bootstrap sequence. This process is typically automated.
-   **Scope:** An instance of a HolderSystem subclass is a singleton scoped to a specific game world (e.g., a client session or a server instance). It persists for the entire duration that the world is active.
-   **Destruction:** The system instance is marked for garbage collection when its associated game world is unloaded. This occurs when a player leaves a server, a server shuts down, or the game transitions away from an active session.

## Internal State & Concurrency
-   **State:** This abstract class is stateless. However, concrete subclasses are inherently stateful. They are expected to maintain internal state that reflects the set of active entities they manage, such as caching component references or managing handles to external resources.
-   **Thread Safety:** **WARNING:** Implementations of HolderSystem are **not** thread-safe by default. The `onEntityAdd` and `onEntityRemoved` callbacks are invoked synchronously on the thread that is modifying the ECS world state. Any long-running operations, blocking I/O, or cross-thread communication within these methods will directly impact engine performance and can lead to deadlocks. Logic must be lightweight, or work must be deferred to a separate, thread-safe queue for processing on a different thread or a later tick.

## API Surface
The public API of HolderSystem is a contract to be implemented by subclasses, not to be called by external user code. The engine's component management layer is the sole caller.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(N) | **Callback.** Invoked by the engine when an entity gains components matching the system's query. Complexity is determined by the subclass implementation. |
| onEntityRemoved(holder, reason, store) | void | O(N) | **Callback.** Invoked by the engine when an entity loses components and no longer matches the system's query. Complexity is determined by the subclass implementation. |

## Integration Patterns

### Standard Usage
A developer does not call methods on a HolderSystem. Instead, they extend it to create a new piece of game logic that reacts to entity lifecycle events.

```java
// Example: A system to manage playing a sound when an entity with a SoundEmitter appears.
public class SoundEmitterSystem extends HolderSystem<SoundEmitterComponent> {

    @Override
    public void onEntityAdd(@Nonnull Holder<SoundEmitterComponent> holder, @Nonnull AddReason reason, @Nonnull Store<SoundEmitterComponent> store) {
        // This code runs automatically when a SoundEmitterComponent is added to an entity.
        SoundEmitterComponent comp = holder.get();
        AudioEngine.playSound(comp.getSoundOnCreate());
    }

    @Override
    public void onEntityRemoved(@Nonnull Holder<SoundEmitterComponent> holder, @Nonnull RemoveReason reason, @Nonnull Store<SoundEmitterComponent> store) {
        // This code runs automatically when the component is removed.
        // For example, stop a looping sound.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new MySystem()`. The engine's SystemRegistry is responsible for the lifecycle of all systems. Manually created instances will not be registered and will not receive callbacks.
-   **Blocking Operations:** Do not perform file I/O, network requests, or complex computations inside `onEntityAdd` or `onEntityRemoved`. These methods are on a performance-critical path and must return quickly to avoid stalling the main game loop.
-   **Assuming Thread Context:** Do not assume these callbacks execute on the main render thread. State mutations that affect rendering or other thread-sensitive systems must be marshaled to the correct thread.

## Data Pipeline
The HolderSystem is a terminal endpoint in the entity modification data flow. It is triggered by changes in the component data layer.

> Flow:
> Engine API Call (e.g., world.addEntity) -> ComponentStore State Change -> **HolderSystem.onEntityAdd** -> System-Specific Logic (e.g., Physics registration, AI behavior activation)

