---
description: Architectural reference for PlayerSkinComponent
---

# PlayerSkinComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Transient Data Component

## Definition
```java
// Signature
public class PlayerSkinComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The PlayerSkinComponent is a data-centric component within the server's Entity-Component-System (ECS) architecture. Its sole responsibility is to associate an entity, typically a player, with its visual skin data, represented by the PlayerSkin object.

This component embodies the ECS principle of separating data from logic. It holds the state, but contains no game logic itself. Systems, such as the network synchronization system or player appearance systems, operate on entities that possess this component.

A critical architectural feature is the **isNetworkOutdated** flag. This is a "dirty flag" used as an optimization to signal that the component's state has changed and needs to be synchronized with clients. The network layer can efficiently query for components with this flag set, avoiding the need to send redundant skin data on every game tick.

## Lifecycle & Ownership
- **Creation:** A PlayerSkinComponent is instantiated and attached to an entity when a player joins the server or when their appearance is modified. This is typically handled by a higher-level system responsible for player session management.
- **Scope:** The component's lifetime is strictly bound to the entity it is attached to. It persists as long as the entity exists within the world's EntityStore.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the game world. The ECS framework manages this process implicitly.

## Internal State & Concurrency
- **State:** The component's state is mutable. While the core PlayerSkin object is final and assigned at construction, the **isNetworkOutdated** flag is designed to be changed during runtime. The PlayerSkin object itself is assumed to be an immutable data transfer object.
- **Thread Safety:** This component is **not thread-safe** and must only be accessed from the main server game loop thread. The methods for managing the **isNetworkOutdated** flag are not atomic and will lead to race conditions and unpredictable network behavior if called from multiple threads. Synchronization is the responsibility of the calling system, which is expected to be single-threaded.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, static type identifier for this component class from the central EntityModule. |
| consumeNetworkOutdated() | boolean | O(1) | Returns the current state of the network dirty flag and immediately resets it to false. This is a classic read-and-clear operation. |
| getPlayerSkin() | PlayerSkin | O(1) | Returns the immutable PlayerSkin data object. |
| setNetworkOutdated() | void | O(1) | Sets the network dirty flag to true, signaling that this component needs to be synchronized. |
| clone() | Component | O(1) | Creates a new component instance that shares the same underlying PlayerSkin object. |

## Integration Patterns

### Standard Usage
A network synchronization system would use this component to determine if a player's appearance needs to be updated for clients.

```java
// A system iterating over entities...
PlayerSkinComponent skin = entity.getComponent(PlayerSkinComponent.class);

if (skin != null && skin.consumeNetworkOutdated()) {
    // The skin has changed since the last sync.
    // Serialize skin.getPlayerSkin() and send to clients.
    Packet packet = createSkinUpdatePacket(entity.getId(), skin.getPlayerSkin());
    networkManager.sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new PlayerSkinComponent()** and attempt to manage it yourself. Components must be attached to an entity via the appropriate world or entity manager, which ensures they are registered with all relevant systems.
- **Asynchronous Modification:** Do not modify the network flag from a separate thread. Accessing **setNetworkOutdated** or **consumeNetworkOutdated** outside the main server tick will break the network synchronization logic.

## Data Pipeline
This component acts as a stateful flag in the data flow from game logic to the client.

> Flow:
> Player Appearance System -> **PlayerSkinComponent.setNetworkOutdated()** -> Server Network Tick System -> **PlayerSkinComponent.consumeNetworkOutdated()** -> Packet Serializer -> Network Buffer -> Client Render Engine

