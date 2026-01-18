---
description: Architectural reference for Nameplate
---

# Nameplate

**Package:** com.hypixel.hytale.server.core.entity.nameplate
**Type:** Data Component

## Definition
```java
// Signature
public class Nameplate implements Component<EntityStore> {
```

## Architecture & Concepts
The Nameplate is a fundamental data component within Hytale's Entity-Component-System (ECS) architecture. It serves as a simple data container, holding the string value to be displayed above an entity in the game world.

As a pure component, it contains no game logic. Its state is manipulated by various *Systems* which are responsible for the business logic of the game. For example, a PlayerManagementSystem might update the Nameplate text when a player changes their name.

A critical feature is the static CODEC field. This integrates the Nameplate component with the engine's serialization and networking layers. The codec defines how to encode and decode a Nameplate instance, allowing it to be persisted in world storage and transmitted to game clients.

The component also implements a "dirty flag" pattern via the *isNetworkOutdated* field. This is a performance optimization that allows the network synchronization system to efficiently transmit updates only for components whose state has changed, significantly reducing bandwidth usage.

## Lifecycle & Ownership
- **Creation:** A Nameplate is never instantiated directly in game logic. It is created and attached to an Entity by the entity management system, either upon the Entity's initial creation (from a template or archetype) or dynamically by a game system.
- **Scope:** The lifecycle of a Nameplate instance is strictly bound to its parent Entity. It exists only as long as the Entity exists and has this component attached.
- **Destruction:** The component is marked for garbage collection when its parent Entity is destroyed or when the Nameplate component is explicitly removed from the Entity. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The Nameplate component is mutable. Its primary state, the *text* field, is designed to be changed during gameplay. It also contains the transient *isNetworkOutdated* flag, which is modified as a side effect of state changes.
- **Thread Safety:** **This component is not thread-safe.** All interactions with an Entity and its components must occur on the main server thread (the world tick thread). Unsynchronized access from other threads will lead to severe race conditions, especially with the *consumeNetworkOutdated* method, potentially causing network desynchronization or data corruption.

## API Surface
The public API is minimal, focusing on state management and integration with the network layer. Trivial getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for the Nameplate component. |
| setText(String text) | void | O(1) | Sets the display text. Marks the component as "dirty" for network synchronization if the new text is different. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically checks and resets the network dirty flag. Returns true if the component's state has changed since the last check. This is intended for internal engine systems. |

## Integration Patterns

### Standard Usage
Game logic should retrieve the component from an entity and then call its methods. Never store a reference to a component longer than the immediate scope, as the component or its parent entity may be destroyed.

```java
// Correctly modifying an entity's nameplate
// Assume 'world' is the current world instance and 'entityId' is valid.

Entity entity = world.getEntity(entityId);
if (entity != null) {
    Nameplate nameplate = entity.getComponent(Nameplate.getComponentType());
    if (nameplate != null) {
        nameplate.setText("New Entity Name");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new Nameplate()`. Components must be added to entities via the appropriate entity management API, which ensures they are correctly registered with all relevant systems.
- **Manual Network Flag Management:** Do not call `consumeNetworkOutdated` from standard game logic. This method is exclusively for the network synchronization system. Calling it manually will prevent the system from detecting a state change, and the update will not be sent to clients.
- **Off-Thread Modification:** Never call `setText` from an asynchronous task or a different thread. All entity modifications must be queued for or executed on the main world thread.

## Data Pipeline
The Nameplate component is a source of data that flows from game logic to the network layer, and ultimately to the client for rendering.

> Flow:
> Game Logic System -> `nameplate.setText("...")` -> **Nameplate** (internal state `text` and `isNetworkOutdated` are updated) -> Network Synchronization System (reads dirty flag via `consumeNetworkOutdated`) -> Serializer (uses `Nameplate.CODEC`) -> Network Packet -> Client Rendering Engine

