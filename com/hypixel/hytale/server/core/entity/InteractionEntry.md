---
description: Architectural reference for InteractionEntry, the server-side state container for entity interactions.
---

# InteractionEntry

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Transient

## Definition
```java
// Signature
public class InteractionEntry {
```

## Architecture & Concepts

The InteractionEntry class is a stateful data container that represents a single, ongoing interaction for an entity on the server. It is a fundamental component of the server's Interaction System, which manages complex, multi-stage actions like crafting, using abilities, or interacting with world objects.

Its primary architectural role is to facilitate a **State Reconciliation** pattern between the server and a remote client. To achieve this in a high-latency environment, it maintains three distinct views of an interaction's state:

1.  **Server State:** The authoritative, canonical state of the interaction as calculated by the server's game logic. This is the ultimate source of truth.
2.  **Client State:** The last known state reported by the client. This is used for validation and desynchronization detection.
3.  **Simulation State:** A speculative, predictive state that the server can use to run "what-if" scenarios. This is critical for implementing client-side prediction, allowing the server to anticipate the results of a client action before the final authoritative state is committed.

The class acts as the nexus for all data related to one interaction, including its current progress, associated metadata via a DynamicMetaStore, and timing information. The `useSimulationState` flag is the central control mechanism that allows higher-level systems to toggle between reading the authoritative state and the predictive state, enabling seamless transitions between prediction and reconciliation.

## Lifecycle & Ownership

-   **Creation:** An InteractionEntry is not created directly. It is instantiated and managed by a higher-level container, typically an **InteractionComponent** attached to a server-side entity. An entry is created when an entity initiates a new interaction.

-   **Scope:** The object's lifetime is strictly bound to the duration of the interaction it represents. It persists from the moment the interaction begins until it either completes successfully, is canceled by the user, or is terminated by game logic.

-   **Destruction:** The owning InteractionComponent is responsible for removing and discarding the InteractionEntry instance when the interaction concludes. Once all references are dropped, it is eligible for garbage collection. Holding a reference to an InteractionEntry beyond its interaction's lifecycle is a memory leak.

## Internal State & Concurrency

-   **State:** The internal state of InteractionEntry is highly **mutable**. Its core purpose is to be modified over time by game logic and network updates to reflect the progress of an interaction. It caches all three state representations (server, client, simulation) and various timing and synchronization flags.

-   **Thread Safety:** **CRITICAL:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. All methods and field access must be performed exclusively from the main server game loop thread that owns the associated entity. Concurrent access from multiple threads will lead to race conditions, state corruption, and unpredictable server behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getState() | InteractionSyncData | O(1) | Returns the currently active state, which is either the server or simulation state based on the `useSimulationState` flag. This is the primary accessor for game logic. |
| setClientState(data) | boolean | O(1) | Updates the entry with the latest state from the client. Performs a critical validation check; returns false and logs a warning if a desync is detected. |
| setUseSimulationState(flag) | void | O(1) | Switches the object's context between authoritative server state and predictive simulation state. |
| flagDesync() | void | O(1) | Manually sets the internal desynchronization flag. This signals to the network layer that a state correction packet must be sent to the client. |
| consumeDesyncFlag() | boolean | O(1) | Returns the current value of the desynchronization flag and immediately resets it to false. This is a "read-once" operation. |
| consumeSendInitial() | boolean | O(1) | Returns true if this interaction is new and its initial state needs to be sent to the client, then resets the flag to false. |
| getMetaStore() | DynamicMetaStore | O(1) | Provides access to the key-value store for custom data associated with this interaction. |

## Integration Patterns

### Standard Usage

An InteractionEntry is almost never handled directly. It is managed by a parent system, such as an InteractionComponent, which processes updates during the entity's tick cycle.

```java
// Conceptual example within an InteractionComponent
void onEntityTick(Entity entity) {
    for (InteractionEntry entry : this.activeInteractions) {
        // 1. Process game logic, updating the entry's serverState
        updateInteractionLogic(entry);

        // 2. Check for desync and flag for correction if needed
        if (entry.consumeDesyncFlag()) {
            networkSystem.forceStateUpdate(entity, entry.getIndex(), entry.getServerState());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new InteractionEntry()`. The lifecycle must be managed by the entity's InteractionComponent to ensure it is correctly registered and cleaned up.
-   **Cross-Thread Access:** **WARNING:** Never read or write to an InteractionEntry from any thread other than the main server thread for that entity. This will cause severe state corruption.
-   **State Indecision:** Do not frequently toggle `useSimulationState`. The flag should be set to true during a predictive phase and set back to false when the prediction is resolved or discarded. Flickering the state can lead to inconsistent behavior.

## Data Pipeline

The InteractionEntry serves as a central point for synchronizing interaction state between the client and server.

> Flow:
> Client Input Packet -> Protocol Decoder -> **InteractionEntry.setClientState()** -> [Validation & Desync Detection] -> Server Game Logic Tick -> [Updates serverState] -> Network Sync System -> **InteractionEntry.getState()** -> Protocol Encoder -> Server State Packet -> Client

