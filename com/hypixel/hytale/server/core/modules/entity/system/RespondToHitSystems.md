---
description: Architectural reference for RespondToHitSystems
---

# RespondToHitSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility

## Definition
```java
// Signature
public class RespondToHitSystems {
```

## Architecture & Concepts

The **RespondToHitSystems** class is not a single system but a static container for a group of related Entity-Component-Systems (ECS). These systems collectively manage the lifecycle and network synchronization of the **RespondToHit** component. This component acts as a flag, indicating that an entity can be damaged or react to combat interactions.

This class exemplifies a common pattern in the Hytale engine: collocating multiple, tightly-coupled systems that operate on the same component into a single file for clarity and maintainability. The architecture is split into two primary responsibilities:

1.  **Game Logic Management:** The **OnPlayerSettingsChange** system implements a specific game rule: it dynamically adds or removes the **RespondToHit** component from a player based on their game mode and settings. This decouples game rules from the core player entity definition.

2.  **Network State Synchronization:** The **EntityTrackerAddAndRemove** and **EntityTrackerUpdate** systems work in concert to efficiently notify clients when an entity's ability to respond to hits changes. They integrate directly with the **EntityTrackerSystems** to ensure updates are only sent to players who can actually see the relevant entity, optimizing network traffic.

A central piece of this architecture is the **QueueResource**, a shared, tick-local data structure. It acts as a communication channel between the system that detects a change (**EntityTrackerAddAndRemove**) and the system that broadcasts the update (**EntityTrackerUpdate**), ensuring changes are processed in the correct phase of the server tick.

### Lifecycle & Ownership

-   **Creation:** The inner system classes (**EntityTrackerAddAndRemove**, **EntityTrackerUpdate**, **OnPlayerSettingsChange**) are not instantiated directly. They are instantiated by the server's ECS framework, likely during the initialization of the **EntityModule**, and registered with the main entity world.
-   **Scope:** These systems are singletons within the context of the server's ECS world. They persist for the entire duration of the server session.
-   **Destruction:** The systems are destroyed and cleaned up when the server shuts down and the ECS world is torn down.

## Internal State & Concurrency

-   **State:** The systems themselves are stateless. Their behavior is driven entirely by the component data they query from the ECS **Store**. The only state within this container is the **QueueResource**, which holds a set of entity references. This resource is mutable but its state is intentionally transient, designed to be cleared at the end of each server tick by the **EntityTrackerUpdate** system.

-   **Thread Safety:** The systems are designed to operate within a potentially parallelized game loop.
    -   The **EntityTrackerUpdate** system explicitly checks if it can run in parallel via the **isParallel** method.
    -   The **QueueResource** uses a **ConcurrentHashMap.newKeySet()** to store entity references. This is a critical design choice, making the shared queue thread-safe and allowing different systems or parallel worker threads to add to it without explicit locking.

## API Surface

The public contract is defined by the ECS framework interfaces implemented by the inner classes. Direct invocation is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityTrackerAddAndRemove | class | N/A | A **RefChangeSystem** that detects when **RespondToHit** is added to or removed from a tracked entity. |
| onComponentAdded(...) | void | O(1) | Adds the entity reference to the shared **QueueResource** for network synchronization. |
| onComponentRemoved(...) | void | O(N) | Immediately queues a removal packet for all N clients tracking the entity. |
| EntityTrackerUpdate | class | N/A | An **EntityTickingSystem** that processes the shared queue and broadcasts updates. |
| tick(...) | void | O(M) | Iterates over M entities in the queue or newly visible entities, queueing network updates. Clears the queue post-tick. |
| OnPlayerSettingsChange | class | N/A | A **RefChangeSystem** that enforces game rules based on player settings. |
| onComponentAdded/Set(...) | void | O(1) | Checks player's game mode and settings, then uses a **CommandBuffer** to add or remove the **RespondToHit** component. |
| QueueResource | class | N/A | A shared, tick-local resource for coordinating updates between systems. |

## Integration Patterns

### Standard Usage

Developers do not interact with these systems directly. They are registered into the ECS world at startup and are invoked automatically by the game loop. The primary interaction is indirect: modifying a component that these systems query will trigger their logic.

```java
// Example: A different system changes a player's settings.
// This action will implicitly trigger the OnPlayerSettingsChange system in a subsequent phase of the game loop.

// Assume 'playerRef' is a valid reference to a player entity.
// Assume 'commandBuffer' is an available CommandBuffer.

PlayerSettings newSettings = ...; // Create new settings where respondToHit is false
commandBuffer.setComponent(playerRef, newSettings);

// The ECS framework will now ensure OnPlayerSettingsChange.onComponentSet is called,
// which will in turn remove the RespondToHit component from the player.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create instances of the inner system classes using **new**. The ECS framework is solely responsible for their lifecycle. Direct instantiation will result in systems that are not registered with the game loop and will not function.
-   **Manual Queue Management:** Do not access or modify the **QueueResource** from outside the provided systems. Its state is only guaranteed to be consistent during the specific execution phases of **EntityTrackerAddAndRemove** and **EntityTrackerUpdate**. Manipulating it from other systems can lead to missed network updates or race conditions.

## Data Pipeline

The systems form two distinct data pipelines depending on the triggering event.

**Pipeline 1: Game Rule Enforcement**

> Flow:
> **PlayerSettings** Component Change -> **OnPlayerSettingsChange** System -> **CommandBuffer** -> **RespondToHit** Component Added/Removed

**Pipeline 2: Network Synchronization**

> Flow:
> **RespondToHit** Component Added -> **EntityTrackerAddAndRemove** System -> **QueueResource** -> **EntityTrackerUpdate** System -> **EntityViewer** -> **ComponentUpdate** Packet Sent to Client

