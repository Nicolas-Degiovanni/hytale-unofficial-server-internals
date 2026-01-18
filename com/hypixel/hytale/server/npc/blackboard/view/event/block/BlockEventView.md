---
description: Architectural reference for BlockEventView
---

# BlockEventView

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event.block
**Type:** Stateful Service

## Definition
```java
// Signature
public class BlockEventView extends EventView<BlockEventView, BlockEventType, EventNotification> {
```

## Architecture & Concepts
The BlockEventView is a world-scoped service that acts as a specialized observer and dispatcher for block-related world events. Its primary function is to translate low-level game events, such as a player breaking a block, into meaningful notifications for the Non-Player Character (NPC) AI system.

It operates as a component of the NPC **Blackboard**, which is the central data repository for an NPC's state and knowledge. This view specifically populates the Blackboard with information about changes to the physical world, allowing NPCs to react to their environment being altered.

The core of its design is an event-driven, publish-subscribe pattern.
1.  **Subscription:** During its initialization, the BlockEventView subscribes to global server events like PlayerInteractEvent, DamageBlockEvent, and BreakBlockEvent.
2.  **Registration:** When an NPCEntity is initialized within the world, the BlockEventView's `initialiseEntity` method is called. This registers the NPC's interest in specific block events, as defined in its configuration (e.g., "notify me when any block in the 'Ores' set is damaged"). These interests are stored internally, mapped by BlockEventType.
3.  **Filtering & Dispatch:** When a subscribed global event fires, the view processes it. It performs crucial filtering, such as checking if the action was performed by a player in creative mode with NPC detection disabled. It then identifies the block involved and queries its internal registry to find all registered NPCs that are interested in that specific block event.
4.  **Notification:** For each interested NPC, it invokes a callback (NPCEntity::notifyBlockChange), which updates the NPC's internal state, potentially triggering a new behavior from the AI controller.

This architecture decouples the core game event loop from the AI system. Instead of every NPC listening to every world event—a significant performance bottleneck—the BlockEventView provides a centralized, optimized hub for block event processing.

### Lifecycle & Ownership
- **Creation:** An instance of BlockEventView is created and owned by the **Blackboard** resource associated with a specific **World**. It is not a global singleton; each game world maintains its own distinct instance.
- **Scope:** The object's lifecycle is tightly bound to its parent World. It persists for the entire duration that the world is loaded and active on the server.
- **Destruction:** The BlockEventView is eligible for garbage collection when its parent World is unloaded and the associated Blackboard resource is destroyed.

## Internal State & Concurrency
- **State:** The BlockEventView is highly stateful. Its primary state is stored in the `entityMapsByEventType` field, a map which contains `EventTypeRegistration` objects. These objects maintain spatial data structures that index NPCs by their location and the block sets they are interested in. This state is mutable, as NPCs are continuously added or removed as they spawn and despawn.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be accessed exclusively from the main server thread for its corresponding World (the "tick thread"). All event handling and state modifications must occur within the synchronized game loop to prevent race conditions and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUpdatedView(ref, accessor) | BlockEventView | O(1) | Retrieves the correct view instance for an entity, crucial for handling entities that move between worlds. |
| initialiseEntity(ref, npc) | void | O(k) | Registers an NPC's interest in various block events. Complexity is proportional to the number of event types. |
| onEntityDamageBlock(ref, event) | void | O(log n) | Entry point for handling DamageBlockEvent. Complexity depends on the underlying spatial query. |
| onEntityBreakBlock(ref, event) | void | O(log n) | Entry point for handling BreakBlockEvent. Complexity depends on the underlying spatial query. |

## Integration Patterns

### Standard Usage
The BlockEventView is an internal system component and is not intended for direct use by most game logic developers. The engine's Entity Component System (ECS) and AI Blackboard system manage its lifecycle and interactions automatically. An NPC's interest in block events is declared declaratively in its component data.

```java
// Engine-level interaction during an event callback
// This code would exist within a higher-level system that processes events.

void handleBlockBreak(BreakBlockEvent event) {
    World world = event.getWorld();
    Blackboard blackboard = world.getResource(Blackboard.getResourceType());
    BlockEventView view = blackboard.getView(BlockEventView.class);

    // The view's handler is invoked, triggering the internal notification pipeline.
    view.onEntityBreakBlock(event.getInitiatorRef(), event);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BlockEventView(world)`. This would create a disconnected view that is not registered with the world's Blackboard, will not receive entity registrations, and will fail to function. Always retrieve the canonical instance from the world's Blackboard.
- **Manual Event Firing:** Do not call methods like `onEntityBreakBlock` directly to simulate an event. This bypasses the server's official event bus, which can lead to desynchronization and prevent other systems from reacting to the event. Always fire a standard `BreakBlockEvent` on the event bus.
- **Cross-Thread Access:** Accessing a BlockEventView from any thread other than its owner World's main tick thread will lead to severe concurrency issues, including `ConcurrentModificationException` and corrupted state.

## Data Pipeline
The flow of data for a block destruction event demonstrates the component's role as a filter and dispatcher within the server architecture.

> Flow:
> Player Input -> Server Game Logic -> **BreakBlockEvent** published to Event Bus -> **BlockEventView::onEntityBreakBlock** -> Creative Mode & Settings Check -> Block ID Lookup -> Internal Spatial Query for interested NPCs -> **NPCEntity::notifyBlockChange** -> NPC Blackboard Update -> AI Behavior Tree Reacts

