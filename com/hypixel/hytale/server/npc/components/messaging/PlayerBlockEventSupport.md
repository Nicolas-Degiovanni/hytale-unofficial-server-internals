---
description: Architectural reference for PlayerBlockEventSupport
---

# PlayerBlockEventSupport

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Transient (Component)

## Definition
```java
// Signature
public class PlayerBlockEventSupport extends EventSupport<BlockEventType, EventNotification> implements Component<EntityStore> {
```

## Architecture & Concepts
PlayerBlockEventSupport is a specialized component within the server-side Entity-Component-System (ECS) architecture. Its primary function is to enable a Non-Player Character (NPC) entity to subscribe to and react to block-related world events initiated by a player, such as placing or breaking a block.

This class acts as a message-handling bridge. It connects the global server event stream to an individual NPC's internal logic, typically a Behavior Tree or a Finite State Machine. By inheriting from the generic EventSupport class, it provides a structured mechanism for different parts of an NPC's brain to register and unregister interest in specific BlockEventType values without maintaining complex state.

As an implementation of the Component interface, this class is a pure data and logic container. It does not exist in isolation; it is always attached to an EntityStore, which represents the data aggregate for a single NPC entity in the world.

### Lifecycle & Ownership
The lifecycle of a PlayerBlockEventSupport instance is strictly managed by the server's ECS framework and is inseparable from its parent entity.

-   **Creation:** Instances are not created directly via their constructor. They are instantiated by the entity management system when an NPC is spawned into the world. This process typically involves invoking the clone method on a registered component prototype, ensuring each NPC receives a unique, isolated instance.
-   **Scope:** The component's lifetime is perfectly aligned with the NPC entity it is attached to. It exists for as long as the NPC is active in the world.
-   **Destruction:** The component is marked for garbage collection when its parent NPC entity is despawned, killed, or otherwise removed from the world. The parent EntityStore is responsible for releasing all references to its components during its own destruction phase.

## Internal State & Concurrency
-   **State:** **Mutable**. The component's primary state is the collection of event listeners inherited from its EventSupport parent. This collection is dynamically modified at runtime as the NPC's behavior logic subscribes and unsubscribes from various block events.
-   **Thread Safety:** **Not thread-safe**. Component data is designed for single-threaded access within the context of the main server game loop (the "tick"). All modifications to its listener list and all event dispatches must occur on the server's primary thread. Unsynchronized access from worker threads or network threads will result in race conditions and catastrophic state corruption.

## API Surface
The public contract is dominated by methods for component management rather than direct event handling, which is an internal concern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, static type identifier for this component class from the central NPCPlugin registry. |
| clone() | Component | O(N) | Creates a new instance of PlayerBlockEventSupport, copying all registered listeners. N is the number of listeners on the prototype. |

**Warning:** The event handling methods inherited from EventSupport, such as listen and notify, should only be called by the NPC's internal behavior systems.

## Integration Patterns

### Standard Usage
This component is designed to be accessed from within an NPC's behavior logic, such as a Behavior Tree node or a state script. The logic retrieves the component from its own entity and registers a callback.

```java
// Conceptual code within an NPC's behavior script or node
EntityStore self = getEntityStore();
PlayerBlockEventSupport blockEvents = self.getComponent(PlayerBlockEventSupport.getComponentType());

// Register a listener to react when a player breaks a nearby log
blockEvents.listen(BlockEventType.BLOCK_BROKEN, (notification) -> {
    if (notification.getBlockType() == BlockTypes.LOG) {
        // Trigger an "angry" state in the NPC's behavior controller
        getBehaviorController().setTarget(notification.getPlayer());
    }
});
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PlayerBlockEventSupport()`. The ECS framework is solely responsible for component creation and lifecycle. Direct instantiation results in an unmanaged object that will not receive events.
-   **Cross-Entity Modification:** Avoid fetching this component from a different entity to add listeners to it. This breaks entity encapsulation and can lead to severe debugging challenges and memory leaks. Inter-entity communication should use a higher-level, global event bus.
-   **Caching Component References:** Do not cache a reference to this component outside the scope of its parent entity. If the entity is destroyed, the cached reference becomes stale and will cause NullPointerExceptions or use-after-free errors.

## Data Pipeline
This component is a key stage in the pipeline that translates a low-level player action into a high-level NPC reaction.

> Flow:
> Player Input -> Network Packet -> Server World System (processes block break) -> Global World Event Bus -> NPC Subsystem -> **PlayerBlockEventSupport** (filters and receives event for a specific NPC) -> Internal EventNotification -> NPC Behavior Logic (e.g., Behavior Tree Node) -> NPC Action (e.g., move, attack)

