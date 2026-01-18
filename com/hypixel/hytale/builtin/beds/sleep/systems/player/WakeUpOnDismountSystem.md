---
description: Architectural reference for WakeUpOnDismountSystem
---

# WakeUpOnDismountSystem

**Package:** com.hypixel.hytale.builtin.beds.sleep.systems.player
**Type:** Transient

## Definition
```java
// Signature
public class WakeUpOnDismountSystem extends RefChangeSystem<EntityStore, MountedComponent> {
```

## Architecture & Concepts
The WakeUpOnDismountSystem is a reactive, event-driven system within the Hytale Entity Component System (ECS) framework. It is not a traditional system that executes every tick. Instead, it subscribes to state changes for a specific component, MountedComponent, by extending the RefChangeSystem base class.

Its sole responsibility is to enforce a specific game rule: when an entity stops being mounted to a bed, its sleep state must be reset to awake. This system acts as a clean, decoupled bridge between the mounting mechanic and the sleep mechanic. The mounting system does not need to know about sleep, and the sleep system does not need to know about mounts; this class handles the interaction between them by observing the state of the world.

This design pattern is highly effective for preventing complex inter-dependencies between otherwise unrelated game features.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's core System Manager during world initialization. Systems are typically discovered via classpath scanning or explicit registration and are not created manually by game logic developers.
- **Scope:** The lifecycle of a WakeUpOnDismountSystem instance is tied to the lifecycle of the world it is registered with. It persists for the entire server session.
- **Destruction:** The instance is garbage collected when the associated world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It contains no member fields and all data required for its operations is passed as arguments to its methods. This is a deliberate and critical design choice for ECS systems, ensuring they are reusable and predictable.
- **Thread Safety:** The class is **not thread-safe** for direct, concurrent invocation. However, it is designed to be executed within the single-threaded context of the Hytale ECS world update loop. The use of a CommandBuffer for all state mutations is a key pattern that guarantees thread safety at the framework level. Commands are queued and executed at a designated synchronization point in the game loop, preventing race conditions and concurrent modification exceptions.

## API Surface
The public API is primarily defined by its parent, RefChangeSystem, and is intended for framework invocation only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| componentType() | ComponentType | O(1) | Framework callback to declare interest in MountedComponent. |
| getQuery() | Query | O(1) | Framework callback to define the component query. |
| onComponentRemoved(...) | void | O(1) | **Core Logic.** Invoked by the ECS framework when a MountedComponent is removed from any entity. It checks if the mount was a bed and, if so, queues a command to set the entity's PlayerSomnolence component to AWAKE. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Its functionality is triggered implicitly by other systems that modify the MountedComponent.

The triggering action is the removal of a MountedComponent from an entity that was previously mounted on a bed.

```java
// Another system's logic that would TRIGGER WakeUpOnDismountSystem
// Note: This code is hypothetical and would exist in a different class.

// An entity 'playerRef' is currently in a bed.
// Some game event causes the player to get out of bed.
commandBuffer.removeComponent(playerRef, MountedComponent.getComponentType());

// The framework will now automatically invoke WakeUpOnDismountSystem.onComponentRemoved
// for the playerRef, which will in turn set them to AWAKE.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WakeUpOnDismountSystem()`. The system's lifecycle is managed entirely by the engine. Manual creation will result in a non-functional object that is not registered to receive component change events.
- **Direct Invocation:** Never call the `onComponentRemoved` method directly. Doing so bypasses the ECS framework's transactional integrity and state management. All component modifications must go through a CommandBuffer to ensure proper ordering and prevent data corruption.

## Data Pipeline
This system operates as a reactive listener in a larger data flow. It does not initiate a pipeline but rather responds to an event within one.

> Flow:
> Player Input or Game Event -> Command to remove MountedComponent -> ECS System Runner -> **WakeUpOnDismountSystem.onComponentRemoved** -> CommandBuffer receives `putComponent(PlayerSomnolence.AWAKE)` -> ECS framework applies CommandBuffer -> Entity state is updated.

