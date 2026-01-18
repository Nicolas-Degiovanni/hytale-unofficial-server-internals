---
description: Architectural reference for EventView
---

# EventView

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event
**Type:** Stateful Base Class

## Definition
```java
// Signature
public abstract class EventView<ViewType extends IBlackboardView<ViewType>, EventType extends Enum<EventType>, NotificationType extends EventNotification>
   implements IBlackboardView<ViewType> {
```

## Architecture & Concepts
EventView is an abstract base class that forms a critical part of the server-side NPC AI infrastructure. Its primary architectural function is to serve as a highly optimized, world-specific event dispatcher. It solves the performance problem of broadcasting global game events to every single NPC, which would be computationally expensive and scale poorly.

Instead, EventView implements a publish-subscribe pattern scoped to a single game world. Concrete subclasses of EventView are responsible for a specific category of events (e.g., combat events, sensory events). NPCs register their interest in particular event types with the appropriate view. When a relevant game event occurs, the system routes it to the corresponding EventView, which then efficiently notifies only the small subset of NPCs that have subscribed.

A key design feature is its management of a local EventRegistry instance. This registry is a child of the global NPCPlugin event registry, creating a hierarchical event bus. This design isolates event processing, improves debuggability, and ensures that events and listeners are automatically garbage collected when a world is unloaded.

## Lifecycle & Ownership
The lifecycle of an EventView instance is strictly bound to the lifecycle of a server World object.

-   **Creation:** Concrete subclasses are instantiated by the world's blackboard management system when the World is initialized. They are not created directly by game logic.
-   **Scope:** An EventView instance persists for the entire duration of its parent World. It holds world-specific state and cannot be shared across different worlds.
-   **Destruction:** The `onWorldRemoved` method is the designated shutdown hook. This method **must** be called by the world management system when the world is being unloaded. It sets an internal shutdown flag and terminates its local EventRegistry, releasing all associated resources and listeners.

## Internal State & Concurrency
-   **State:** EventView is highly stateful. Its primary state is maintained in the `entityMapsByEventType` map, which caches collections of entity references that are subscribed to various events. It also maintains a mutable `reusableEventNotification` object, a performance optimization designed to prevent object allocation during the high-frequency process of event broadcasting.

-   **Thread Safety:** This class is **not thread-safe** and is designed to be accessed exclusively from its parent World's main server thread. While its internal EventRegistry uses thread-safe collections for listener management, the core event dispatch logic (`onEvent`) and state modifications are unsynchronized. Unmanaged multi-threaded access will lead to race conditions and unpredictable behavior. The `shutdown` flag is used to coordinate state between the main thread and the event registry's internal logic.

## API Surface
The public API is primarily composed of lifecycle and introspection methods. The core logic is exposed via the protected `onEvent` method, intended for use by concrete subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onWorldRemoved() | void | O(1) | **Critical Lifecycle Hook.** Shuts down the view and its event bus. |
| cleanup() | void | O(N) | Clears all entity subscriptions from all managed event types. N = number of event types. |
| getSetCount() | int | O(N) | Returns the total number of subscribed entities across all event types for metrics. |
| forEach(...) | void | O(M) | Executes a given consumer for every subscribed entity. M = total number of subscriptions. |
| onEvent(...) | protected void | O(K) | The core dispatch mechanism. Relays an event notification to all K entities subscribed to the specific event type. |

## Integration Patterns

### Standard Usage
A developer does not interact with EventView directly. Instead, game systems interact with a concrete implementation. The subclass provides strongly-typed methods that translate a game-level event into a call to the protected `onEvent` method.

```java
// In a game system, such as a CombatManager:

// 1. An entity takes damage.
// 2. The system retrieves the world-specific view for damage events.
World world = targetEntity.getWorld();
DamageEventView damageView = world.getBlackboard().getView(DamageEventView.class);

// 3. The system calls the specific handler on the concrete view.
//    This handler will, in turn, call the base class onEvent method.
if (damageView != null) {
    damageView.onEntityDamaged(initiatorRef, targetRef, damageInfo);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never instantiate a subclass of EventView with `new`. These objects are managed by the World's blackboard system and require proper integration into its lifecycle. Always retrieve instances via `world.getBlackboard().getView(...)`.
-   **State Leakage:** Do not store a reference to an EventView instance in a static field or any context that outlives the World it belongs to. This will prevent the World and all its associated resources from being garbage collected, causing a severe memory leak.
-   **Incorrect Shutdown:** Subclasses that override `onWorldRemoved` **must** call `super.onWorldRemoved()` to ensure the internal event registry is properly shut down. Failure to do so will result in a resource leak and orphaned threads.

## Data Pipeline
The primary responsibility of EventView is to act as a routing and filtering hub in the NPC event data pipeline.

> Flow:
> High-Level Game Event (e.g., a projectile hit) -> Game System (e.g., CombatManager) -> Concrete Subclass Handler (e.g., DamageEventView.onEntityDamaged) -> **EventView.onEvent** -> EventTypeRegistration -> Filtered set of Subscribed NPC Listeners

