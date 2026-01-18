---
description: Architectural reference for InteractionChain
---

# InteractionChain

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Transient

## Definition
```java
// Signature
public class InteractionChain implements ChainSyncStorage {
```

## Architecture & Concepts

The InteractionChain is a stateful, runtime object that executes a sequence of game logic operations. It acts as a server-authoritative state machine for any complex, multi-step, or branching action an entity can perform, such as using an item, casting a spell, or progressing through a quest dialogue.

It is the central component for implementing the server's predictive networking model for gameplay actions. Each InteractionChain maintains a dual state: the authoritative server state and a simulated client state. This allows the server to process the canonical sequence of events while providing clients with the necessary data to predict outcomes and maintain a responsive user experience.

An InteractionChain should not be considered a static configuration. Instead, it is the live instance of a predefined interaction template, represented by a **RootInteraction** asset. The RootInteraction defines the *potential* operations and logic, while the InteractionChain executes and records the *actual* outcome of those operations for a specific context.

A key architectural feature is the ability for chains to fork. A single operation within a chain can spawn one or more new, dependent InteractionChains. This creates a tree-like structure of execution, enabling complex, non-linear gameplay logic such as branching skill trees or dialogue choices. These forks are tracked via the ForkedChainId identifier.

### Lifecycle & Ownership

-   **Creation:** InteractionChains are instantiated and managed exclusively by the InteractionManager. They are created in response to a gameplay trigger, such as a player input packet or an AI-driven event. Direct instantiation outside of the manager is an anti-pattern and will result in a non-functional, untracked object.
-   **Scope:** The lifetime of an InteractionChain is tied directly to the duration of the action it represents. It persists across multiple server ticks while the action is in progress and is destroyed upon completion, failure, or cancellation. It is a short-to-medium-lived object, not a session-scoped entity.
-   **Destruction:** An InteractionChain is marked for garbage collection once its final state is reached (either Finished or Failed) and its onCompletion callback has been executed. The managing InteractionManager is responsible for removing the completed chain from its collection of active chains.

## Internal State & Concurrency

-   **State:** The InteractionChain is fundamentally a mutable object. Its primary purpose is to transition through various states as it executes operations. It maintains a significant amount of internal state, including:
    -   A log of executed steps (the interactions list).
    -   A map of all active sub-chains (the forkedChains map).
    -   A call stack for nested interactions.
    -   Separate counters and state trackers for server execution and client-side simulation.
    -   Temporary buffers for synchronization data being prepared for network transmission.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single thread, typically the main server game loop thread corresponding to the world or region where the interaction occurs. All state modifications and reads must be performed from this thread. Unsynchronized access from other threads will lead to state corruption, race conditions, and unpredictable behavior.

## API Surface

The public API is designed for state progression and inspection by the owning InteractionManager and related gameplay systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| findForkedChain(id, data) | InteractionChain | O(1) | Retrieves or maps a forked sub-chain. Critical for navigating the interaction tree. |
| putForkedChain(id, chain) | void | O(1) | Registers a new forked sub-chain, adding a branch to the interaction tree. |
| pushRoot(next, simulate) | void | O(1) | Pushes a new RootInteraction onto the call stack, enabling nested interactions. |
| popRoot() | void | O(1) | Pops from the call stack, returning to the parent interaction's logic flow. |
| updateServerState() | void | O(N) | Advances the server-side state machine. Complexity depends on operation logic. |
| getOrCreateInteractionEntry(index) | InteractionEntry | O(1) | Retrieves or creates a record for an operation at a specific index in the chain. |
| syncFork(ref, manager, packet) | void | O(log N) | Entry point for applying network synchronization data from a client to a forked chain. |

## Integration Patterns

### Standard Usage

The InteractionChain is almost exclusively managed by the InteractionManager. A system that wishes to initiate an interaction will request it from the manager, which then creates and ticks the chain until completion.

```java
// Standard interaction flow managed by the InteractionManager
// Note: This is a conceptual example. Direct calls are handled by the manager.

// 1. Manager creates a chain for a player's action
InteractionChain chain = interactionManager.startChain(player, rootInteractionAsset);

// 2. In the server tick loop, the manager updates the chain
if (!chain.isComplete()) {
    interactionManager.tickChain(chain);
}

// 3. During the tick, an operation might fork
// This logic is typically inside an InteractionOperation implementation
if (someCondition) {
    ForkedChainId forkId = chain.getNextForkId();
    InteractionChain newFork = interactionManager.createFork(chain, forkId, otherRootAsset);
    chain.putForkedChain(forkId, newFork);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new InteractionChain()`. The InteractionManager is the sole authority for creating and registering chains. Bypassing it will result in an object that is not ticked, not synchronized with clients, and not properly cleaned up.
-   **Cross-Thread Modification:** Do not access or modify an InteractionChain from any thread other than the main game thread that owns it. This will corrupt its internal state and cause severe, difficult-to-debug issues.
-   **State Re-use:** An InteractionChain is a single-use object. Do not attempt to "reset" or re-run a completed chain. A new chain must be created for each new interaction instance.

## Data Pipeline

The InteractionChain is a critical node in the client-server state synchronization pipeline. It ensures that the client's predicted actions are eventually consistent with the server's authoritative state.

> **Flow: Server to Client Synchronization**
>
> 1.  **Server Event:** A player input or AI decision triggers the creation of an **InteractionChain** via the InteractionManager.
> 2.  **Server Tick:** The InteractionManager ticks the chain. Operations are executed, state changes, and new forks may be created.
> 3.  **State Serialization:** Changes to the chain's state (new entries, state transitions, new forks) are collected into InteractionSyncData objects.
> 4.  **Packet Assembly:** The InteractionManager packages the sync data into a SyncInteractionChain network packet.
> 5.  **Network Transmission:** The packet is sent to the relevant client.
> 6.  **Client Deserialization:** The client's network layer receives the packet and routes it to its local InteractionManager.
> 7.  **State Reconciliation:** The client's InteractionManager finds the corresponding predicted InteractionChain and calls `syncFork` or other update methods to apply the server's authoritative state, correcting any mispredictions.

