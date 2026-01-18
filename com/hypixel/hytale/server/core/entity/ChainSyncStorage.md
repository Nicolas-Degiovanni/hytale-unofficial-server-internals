---
description: Architectural reference for ChainSyncStorage
---

# ChainSyncStorage

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface ChainSyncStorage {
```

## Architecture & Concepts
The ChainSyncStorage interface defines the contract for server-side state management of complex, multi-step player interactions. It serves as a stateful buffer and synchronization mechanism between a client's interaction sequence and the server's authoritative game state.

In Hytale's architecture, many player actions are not atomic; they are part of a "chain" of related steps, such as a multi-stage crafting recipe or a complex world interaction. This interface is responsible for tracking the client's progress through such a chain, validating incoming data, and resolving discrepancies between the client's predicted state and the server's actual state.

It is a critical component in the server's netcode, specifically for ensuring that player interactions are processed reliably and securely, even under adverse network conditions like high latency or packet loss. The core responsibility is to store and manage `InteractionSyncData` for each step in a chain, providing the `InteractionManager` with the necessary context to execute game logic.

## Lifecycle & Ownership
As an interface, ChainSyncStorage itself has no lifecycle. However, its implementations follow a strict, entity-bound lifecycle.

- **Creation:** An object implementing this interface is typically instantiated when an entity capable of performing interaction chains (e.g., a Player) is created or loaded into the world. It is owned by that parent entity.
- **Scope:** The instance persists for the entire lifetime of its owning entity. Its internal state represents the ongoing interaction state for that specific entity.
- **Destruction:** The instance is marked for garbage collection when its owning entity is unloaded or removed from the world. This typically involves a cleanup routine to release any held resources or references.

## Internal State & Concurrency
- **State:** Implementations of this interface are inherently stateful and highly mutable. They are expected to cache the client's last known `InteractionState`, a collection of `InteractionEntry` objects, and associated synchronization data. This state is volatile and changes frequently in response to network packets.
- **Thread Safety:** **WARNING:** Implementations of ChainSyncStorage must be thread-safe. This component operates at the intersection of the server's network thread, which receives packets, and the main game tick thread, which processes entity logic. Methods like `syncFork` will be called from the network processing pipeline, while other game logic may query the state from the main thread. Implementations must use appropriate concurrency controls (e.g., synchronized blocks, concurrent collections) to prevent race conditions and state corruption.

## API Surface
The public contract is designed for managing the synchronization of an interaction sequence.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getClientState() | InteractionState | O(1) | Retrieves the last known interaction state reported by the client. |
| setClientState(state) | void | O(1) | Updates the server's record of the client's interaction state. |
| getInteraction(id) | InteractionEntry | O(1) | Fetches a specific interaction entry by its sequence ID. Assumes a map-like backing store. |
| putInteractionSyncData(id, data) | void | O(1) | Stores synchronization data for a specific step in the chain. |
| updateSyncPosition(id) | void | O(1) | Advances the server's synchronization marker to the specified ID. |
| isSyncDataOutOfOrder(id) | boolean | O(1) | Checks if a received synchronization ID is older than the current position, indicating a stale or out-of-order packet. |
| syncFork(store, manager, chain) | void | O(N) | The primary synchronization entry point. Reconciles a client's reported interaction chain with the server's state. This is a high-complexity operation that may trigger significant game logic via the InteractionManager. |
| clearInteractionSyncData(id) | void | O(1) | Purges stored synchronization data for a given ID, typically after it has been successfully processed. |

## Integration Patterns

### Standard Usage
The `InteractionManager` is the primary consumer of this interface. It holds a `ChainSyncStorage` instance for each relevant player. When a `SyncInteractionChain` packet is received from a client, the manager delegates the complex task of state reconciliation to the storage object.

```java
// Within a server-side packet handler or InteractionManager
void handleInteractionPacket(Player player, SyncInteractionChain packet) {
    ChainSyncStorage storage = player.getInteractionStorage();
    Ref<EntityStore> storeRef = world.getEntityStore();
    
    // Delegate the entire synchronization and state update logic
    // to the storage implementation.
    storage.syncFork(storeRef, this, packet);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation Outside of syncFork:** Modifying the internal state of a `ChainSyncStorage` implementation via methods like `putInteractionSyncData` or `setClientState` outside the context of the `syncFork` process can lead to severe desynchronization between the client and server. All state changes should be driven by the authoritative `syncFork` call.
- **Ignoring Out-of-Order Data:** Failing to use `isSyncDataOutOfOrder` before processing an interaction can cause the server to process old, irrelevant packets, potentially leading to exploits or state corruption.
- **Memory Leaks:** Forgetting to call `clearInteractionSyncData` for processed or aborted interaction chains will result in a memory leak, as the storage object will accumulate stale data for the lifetime of the entity.

## Data Pipeline
ChainSyncStorage is a key processing stage for player interaction data flowing from the network to the game simulation.

> Flow:
> Client Input -> `SyncInteractionChain` Packet -> Server Network Layer -> `InteractionManager` -> **`ChainSyncStorage.syncFork()`** -> `EntityStore` State Update -> World Simulation Tick

