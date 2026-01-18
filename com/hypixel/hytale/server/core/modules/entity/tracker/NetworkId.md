---
description: Architectural reference for NetworkId
---

# NetworkId

**Package:** com.hypixel.hytale.server.core.modules.entity.tracker
**Type:** Data Component

## Definition
```java
// Signature
public final class NetworkId implements Component<EntityStore> {
```

## Architecture & Concepts
The NetworkId is a fundamental data component within Hytale's server-side Entity-Component-System (ECS) architecture. It serves a single, critical purpose: to assign a stable, unique integer identifier to an entity for the duration of its network-tracked lifetime.

This component acts as the canonical "network handle" for an entity. When the server needs to inform clients about an entity's creation, destruction, or state changes, it uses the integer value from this component. It is the primary key used by the network protocol to synchronize entity state between the server and connected clients.

Its implementation as a final, immutable class is a deliberate design choice. It ensures that once an entity's network identity is established, it cannot be accidentally or maliciously altered, preventing a wide range of potential synchronization bugs. The management of this component type is centralized within the EntityModule, which acts as the registry and authority for core entity components.

### Lifecycle & Ownership
- **Creation:** A NetworkId component is instantiated and attached to an entity by the server's entity tracking system. This typically occurs the moment an entity is spawned or enters a state where it needs to be synchronized to clients (for example, when it enters a player's view distance). The ID itself is allocated from a server-wide pool to guarantee uniqueness.
- **Scope:** The component's lifetime is strictly bound to the entity it is attached to. It persists as long as the entity exists and is managed by the server's world simulation.
- **Destruction:** The component is destroyed and its memory is reclaimed as part of the parent entity's destruction sequence. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of a single `final int id`, which is set at construction time and can never be changed. The `clone()` method returns the same instance (`this`), a common and highly efficient pattern for immutable objects.
- **Thread Safety:** **Fully thread-safe**. Due to its immutability, an instance of NetworkId can be safely passed between and read by multiple threads without any external locking or synchronization. This is critical for high-performance networking and game logic systems that may operate concurrently.

## API Surface
The public contract is minimal, exposing only the essential identifier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the canonical ComponentType definition from the EntityModule. This is the standard mechanism for identifying this component class within the ECS. |
| getId() | int | O(1) | Returns the unique integer identifier for the associated entity. |
| clone() | Component | O(1) | Returns the same instance. This is an optimization that avoids unnecessary object allocation, made possible by the component's immutability. |

## Integration Patterns

### Standard Usage
The component should always be retrieved from an entity instance. Systems that build network packets or perform lookups will query for this component to get the entity's network handle.

```java
// A system needs to serialize an entity's ID for a network packet.
Entity entity = ... // an entity retrieved from the world

// Retrieve the component using its static type definition
NetworkId networkIdComp = entity.getComponent(NetworkId.getComponentType());

if (networkIdComp != null) {
    int id = networkIdComp.getId();
    packetBuilder.writeVarInt(id);
} else {
    // This entity is not tracked over the network.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new NetworkId(someId)`. The entity tracking system is the sole authority for creating these components and assigning unique IDs. Manual creation will lead to ID collisions and severe network desynchronization issues.
- **State Assumption:** Do not assume an entity will always have a NetworkId. Entities that are purely server-side constructs (e.g., internal markers, triggers) may not be registered with the network tracker and will therefore not possess this component. Always perform a null check.

## Data Pipeline
The NetworkId is a source of data for the network serialization pipeline. It does not process data itself.

> Flow:
> Entity Spawning -> **Entity Tracking System assigns ID & attaches NetworkId component** -> Network Serialization System reads component -> ID written to `EntitySpawnPacket` -> Client receives packet and maps ID to a client-side entity.

