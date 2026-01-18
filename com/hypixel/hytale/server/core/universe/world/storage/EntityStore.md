---
description: Architectural reference for EntityStore
---

# EntityStore

**Package:** com.hypixel.hytale.server.core.universe.world.storage
**Type:** Transient

## Definition
```java
// Signature
public class EntityStore implements WorldProvider {
```

## Architecture & Concepts
The EntityStore is the central authority for entity identity and lookup within a specific server World. It acts as a critical bridge between the abstract, high-performance Entity Component System (ECS) and the rest of the game server. Its primary responsibility is to maintain fast-access indices that map persistent or network-transient identifiers to the internal ECS entity reference, known as a Ref.

While the underlying ECS Store manages components and entities using opaque integer IDs, other systems need to find entities using more meaningful identifiers:
- **UUID:** A globally unique and persistent identifier for an entity (e.g., a specific player or a placed block entity).
- **NetworkId:** A session-specific, integer-based identifier used for efficient network replication between the server and clients.

EntityStore achieves this by leveraging two internal reactive systems, UUIDSystem and NetworkIdSystem. These systems listen for the addition or removal of entities that possess a UUIDComponent or NetworkId component. When such an event occurs, these systems automatically update the EntityStore's internal lookup maps. This design decouples the core ECS from game-specific identity management, ensuring that the indexing logic is co-located with the data store it manages.

## Lifecycle & Ownership
- **Creation:** An EntityStore is instantiated by its parent World object during the world's initialization phase. It is fundamentally tied to the lifecycle of a single world instance.
- **Scope:** The instance persists for the entire duration that its parent World is loaded and active in memory. It is not a global singleton; each world has its own distinct EntityStore.
- **Destruction:** The shutdown method is invoked when the parent World is unloaded. This triggers the shutdown of the underlying ECS Store, releasing all associated entity and component memory, and clears the lookup maps.

## Internal State & Concurrency
- **State:** The EntityStore maintains a highly mutable state, consisting of the core ECS Store and two primary lookup maps:
    - entitiesByUuid: A map from a persistent UUID to a transient entity Ref.
    - networkIdToRef: A map from a session-specific integer NetworkId to a transient entity Ref.
    It also manages an atomic counter for generating new NetworkIds.

- **Thread Safety:** This class exhibits mixed-threading characteristics and requires careful handling.
    - The entitiesByUuid map uses a ConcurrentHashMap, making UUID-based lookups and updates thread-safe.
    - The networkIdCounter uses an AtomicInteger, ensuring that NetworkId generation is thread-safe.
    - **WARNING:** The networkIdToRef map uses a non-thread-safe Int2ObjectOpenHashMap. All modifications to this map are performed by the NetworkIdSystem within the single-threaded context of the ECS CommandBuffer processing. Read access from other threads via getRefFromNetworkId is **not safe** if it can race with entity creation or destruction ticks. Assume all interactions with NetworkId happen on the main world thread unless explicitly documented otherwise.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start(IResourceStorage) | void | O(1) | Initializes the internal ECS store. Must be called once before use. |
| shutdown() | void | O(N) | Shuts down the ECS store and clears all internal maps. |
| getStore() | Store | O(1) | Provides direct access to the underlying ECS Store. Use with caution. |
| getRefFromUUID(UUID) | Ref | O(1) | Retrieves an entity reference using its persistent UUID. Thread-safe. |
| getRefFromNetworkId(int) | Ref | O(1) | Retrieves an entity reference using its transient network ID. Not thread-safe for cross-thread reads. |
| takeNextNetworkId() | int | O(1) | Atomically generates and returns a new, unique network ID. Thread-safe. |

## Integration Patterns

### Standard Usage
The primary use case is to resolve a known UUID or NetworkId into an ECS Ref, which can then be used to query or modify the entity's components. This is the main entry point for systems that operate on entities but do not hold a direct reference.

```java
// Example: Finding a player entity by its UUID
EntityStore entityStore = world.getEntityStore();
UUID playerUuid = ...; // The UUID of the player to find

Ref<EntityStore> playerRef = entityStore.getRefFromUUID(playerUuid);

if (playerRef != null) {
    // Now use the Ref to interact with the entity's components
    Store<EntityStore> store = entityStore.getStore();
    PositionComponent pos = store.getComponent(playerRef, PositionComponent.class);
    // ... do work with the position
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new EntityStore(). The instance is managed by the World and requires a specific initialization sequence via its start method. Always retrieve the active instance from the World context.
- **Manual Map Modification:** Do not attempt to modify the internal entitiesByUuid or networkIdToRef maps directly. These are managed exclusively by the internal UUIDSystem and NetworkIdSystem to maintain consistency with the ECS Store. Manual changes will lead to desynchronization and undefined behavior.
- **Cross-Thread NetworkId Lookup:** Do not call getRefFromNetworkId from an asynchronous task or different thread without external synchronization. This can lead to race conditions where you read from the map while the main world thread is modifying it during an entity tick.

## Data Pipeline
EntityStore does not process a stream of data but rather acts as a reactive index. The flow of identity registration is as follows:

> Flow:
> Entity created in ECS Store -> UUIDComponent added to entity -> **UUIDSystem** callback triggered -> **EntityStore**'s *entitiesByUuid* map is updated -> Entity is now discoverable via getRefFromUUID.

