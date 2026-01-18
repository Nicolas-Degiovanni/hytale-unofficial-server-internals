---
description: Architectural reference for KillFeedEvent
---

# KillFeedEvent

**Package:** com.hypixel.hytale.server.core.modules.entity.damage.event
**Type:** Static Event Namespace

## Definition
```java
// Signature
public class KillFeedEvent {
    // Contains static nested event classes
}
```

## Architecture & Concepts

The KillFeedEvent class is not a traditional object but a static namespace that groups a family of related, cancellable ECS events. It serves as the primary communication contract for the server-side kill feed system. When an entity's death is processed by the core damage module, it does not directly manipulate UI or chat. Instead, it broadcasts instances of the events defined within KillFeedEvent.

This decoupled, event-driven architecture allows other game modules to subscribe to and influence the final appearance of kill notifications without modifying the core combat logic. The pipeline is intentionally segmented into three distinct events, each targeting a different audience:

*   **KillerMessage:** An event specifically for constructing the message sent to the killer.
*   **DecedentMessage:** An event for constructing the message sent to the entity that was killed.
*   **Display:** A general-purpose event for creating the public kill feed entry visible to other players.

By extending CancellableEcsEvent, each stage of this pipeline can be intercepted and terminated by a higher-priority system, enabling game modes to selectively suppress or completely override default kill feed behavior.

### Lifecycle & Ownership

As a static container, KillFeedEvent itself is not instantiated and has a lifecycle tied to the JVM classloader. The following applies to the *nested event objects* it defines (e.g., Display, KillerMessage).

-   **Creation:** Instantiated exclusively by the server's core damage processing system at the exact moment an entity's death is confirmed within a game tick.
-   **Scope:** These event objects are extremely short-lived and transient. Their scope is confined to the single game tick in which they are created and dispatched on the ECS event bus.
-   **Destruction:** They become eligible for garbage collection immediately after the ECS event bus has finished notifying all registered listeners for that tick. Holding a reference to an event object beyond the scope of its handler method is a critical anti-pattern and potential memory leak.

## Internal State & Concurrency

-   **State:** The outer KillFeedEvent class is stateless. The inner event classes are stateful and **highly mutable**. Their primary purpose is to carry data (like Damage, Message, icon) that is expected to be modified by event listeners during the dispatch process. For example, a listener might receive a Display event and call setIcon to add a headshot icon.

-   **Thread Safety:** These event objects are **not thread-safe**. They are designed to be created, dispatched, and handled exclusively on the main server game thread. Any attempt to access or modify an event object from an asynchronous task or worker thread will result in race conditions, data corruption, and severe server instability. All interactions must be synchronized with the server tick.

## API Surface

The KillFeedEvent class itself has no public API. The API surface consists of the methods within its nested event classes.

### KillFeedEvent.DecedentMessage

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDamage() | Damage | O(1) | Retrieves the final Damage object that caused the death. |
| getMessage() | Message | O(1) | Gets the formatted chat message to be sent to the decedent. May be null. |
| setMessage(Message) | void | O(1) | Sets or overrides the chat message for the decedent. |

### KillFeedEvent.Display

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBroadcastTargets() | List<PlayerRef> | O(1) | Returns the list of players who should see this kill feed entry. |
| getDamage() | Damage | O(1) | Retrieves the final Damage object that caused the death. |
| getIcon() | String | O(1) | Gets the resource path for the icon used in the kill feed UI. May be null. |
| setIcon(String) | void | O(1) | Sets or overrides the icon for the kill feed UI. |

### KillFeedEvent.KillerMessage

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDamage() | Damage | O(1) | Retrieves the final Damage object that caused the death. |
| getTargetRef() | Ref<EntityStore> | O(1) | Gets a stable reference to the entity that was killed. |
| getMessage() | Message | O(1) | Gets the formatted chat message to be sent to the killer. May be null. |
| setMessage(Message) | void | O(1) | Sets or overrides the chat message for the killer. |

## Integration Patterns

### Standard Usage

The intended pattern is to create a system that listens for one or more of these events on the ECS event bus. The system can then inspect the event data and optionally modify it to create custom game logic.

**WARNING:** Do not create and fire these events manually. They are a product of the core damage pipeline.

```java
// Example of a system that adds a special icon for headshot kills
public class HeadshotIconSystem extends EcsSystem {

    @Subscribe
    public void onKillFeedDisplay(KillFeedEvent.Display event) {
        Damage damage = event.getDamage();
        
        // Check if the damage has a "headshot" tag
        if (damage.hasTag("headshot")) {
            // Modify the event to add a custom icon
            event.setIcon("asset://hytale/ui/icons/headshot.png");
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Manual Instantiation:** Do not manually create and fire these events. Doing so bypasses the core damage logic and can lead to a desynchronized game state, where a kill is announced but the entity is not actually dead.
-   **Asynchronous Handling:** Do not process these events on a separate thread. Modifying the event object while the main game thread is also processing it will cause a ConcurrentModificationException or other unpredictable behavior.
-   **Reference Caching:** Do not store a reference to an event object in a field for later use. Event objects are ephemeral and should be considered invalid after the handler method returns.

## Data Pipeline

The KillFeedEvent family acts as a central hub in the data flow that transforms a gameplay death into player-facing information.

> Flow:
> Entity Health drops to zero -> Core Damage Module confirms kill -> **KillFeedEvent (Display, KillerMessage, DecedentMessage)** fired -> ECS Event Bus dispatches to listeners -> Subscribed systems (e.g., ChatSystem, KillFeedUISystem) read/modify events -> Finalized Message/UI data sent via Network Packet to clients

