---
description: Architectural reference for TeleportHistory
---

# TeleportHistory

**Package:** com.hypixel.hytale.builtin.teleport.components
**Type:** Component

## Definition
```java
// Signature
public class TeleportHistory implements Component<EntityStore> {
```

## Architecture & Concepts
The TeleportHistory component provides state management for an entity's location history, enabling "back" and "forward" teleportation functionality similar to a web browser's navigation buttons. It is a fundamental building block of the server's teleportation command system.

Architecturally, this class is a pure data component within an Entity-Component-System (ECS) design. It does not perform the teleportation itself. Instead, its primary responsibility is to maintain two distinct stacks of previous locations: a **back** stack for past waypoints and a **forward** stack for undone waypoints.

When a navigation action like `back` or `forward` is invoked, TeleportHistory does not directly manipulate the entity's position. It follows a decoupled, message-passing pattern by adding a new `Teleport` component to the entity. A dedicated `TeleportSystem` is responsible for detecting this component in the game loop, executing the physical teleport, and then removing the `Teleport` component. This design cleanly separates the concerns of history management from the core mechanics of entity movement.

The history is capped at 100 entries to prevent unbounded memory growth for entities that teleport frequently.

## Lifecycle & Ownership
- **Creation:** An instance of TeleportHistory is not created globally. It is instantiated and attached to a specific entity, typically a player, when that entity first performs an action that needs to be recorded (e.g., using a teleport command).
- **Scope:** The component's lifetime is strictly bound to the entity it is attached to. It persists as long as the entity exists within an `EntityStore`.
- **Destruction:** The component is marked for garbage collection and its state is lost when the owning entity is removed from the world or server.

## Internal State & Concurrency
- **State:** The internal state is highly mutable, consisting of two `ArrayDeque` collections, `back` and `forward`, which store `Waypoint` objects. The state represents a complete, ordered history of an entity's teleports. It acts as a stateful cache of past locations.

- **Thread Safety:** **This component is not thread-safe.** The underlying `ArrayDeque` collections are not concurrent. All method calls that mutate internal state, such as `append`, `back`, and `forward`, must be executed on the same thread that owns the entity's `EntityStore`, which is typically the main server tick thread.

    **Warning:** Accessing or modifying a TeleportHistory instance from an asynchronous task or a different thread will lead to `ConcurrentModificationException` or other unpredictable state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Static accessor to retrieve the unique type identifier for this component. |
| append(world, pos, rot, key) | void | O(1) | Adds a new waypoint to the history. This clears the forward stack. |
| back(ref, count) | void | O(N) | Moves back N steps in history. Adds a `Teleport` component to the entity. |
| forward(ref, count) | void | O(N) | Moves forward N steps in history. Adds a `Teleport` component to the entity. |
| getBackSize() | int | O(1) | Returns the number of locations available in the back history. |
| getForwardSize() | int | O(1) | Returns the number of locations available in the forward history. |

## Integration Patterns

### Standard Usage
This component is typically retrieved from an entity's `Store` and manipulated by a higher-level system, such as a command handler.

```java
// Example: Implementing a /back command
PlayerRef playerRef = ...; // The target player
Store<EntityStore> store = playerRef.getStore();
Ref<EntityStore> entityRef = playerRef.getRef();

// Retrieve the component; it may be null if the player has no history
TeleportHistory history = store.getComponent(entityRef, TeleportHistory.getComponentType());

if (history != null && history.getBackSize() > 0) {
    // Request a teleport by invoking the component's logic
    history.back(entityRef, 1);
} else {
    playerRef.sendMessage(Message.translation("server.commands.teleport.noHistory"));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TeleportHistory()`. Components must be managed by the `EntityStore` via `store.addComponent()`. Direct instantiation detaches the component from the ECS lifecycle and will not function.
- **Asynchronous Modification:** Do not call `append`, `back`, or `forward` from a separate thread. This will corrupt the history stacks and cause server instability. All interactions must be synchronized with the main game loop.
- **State Tampering:** Do not attempt to retrieve the internal `back` or `forward` collections via reflection to modify them directly. The component's API is the sole contract for state changes.

## Data Pipeline
The primary data flow for this component is request-based. It transforms a history navigation request into a teleport action request.

> Flow:
> Player Command (`/back`) -> Command Handling System -> **TeleportHistory.back()** -> `EntityStore.addComponent(Teleport)` -> Teleport System (detects `Teleport` component) -> Entity Position Updated in World

---

