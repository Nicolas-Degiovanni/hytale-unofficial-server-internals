---
description: Architectural reference for InteractionManager
---

# InteractionManager

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Transient

## Definition
```java
// Signature
public class InteractionManager implements Component<EntityStore> {
```

## Architecture & Concepts

The InteractionManager is the server-authoritative state machine for all complex entity actions, such as combat, block breaking, and item usage. It acts as the central nervous system for the server-side interaction simulation, mediating between network input from clients and the server's game logic.

Its primary responsibility is to manage the lifecycle of **InteractionChains**. An InteractionChain represents a complete, multi-stage action, composed of individual **Operations**. For example, a sword swing might be a chain consisting of a wind-up operation, a strike operation, and a recovery operation.

The manager's architecture is fundamentally designed for a client-server model with server authority. It continuously processes a queue of `SyncInteractionChain` packets from the client, which contain the client's predicted state for an action. The InteractionManager validates this input against its own simulation, resolves discrepancies, and broadcasts authoritative state back to the client. This synchronization model is critical for providing responsive gameplay while preventing cheating.

Key architectural pillars include:
-   **State Reconciliation:** The core `tick` loop consumes client packets and compares them against the server's state. It handles out-of-order packets, timeouts, and desynchronization events, ensuring the server's view of the world remains consistent.
-   **Rules-Based Conflict Resolution:** The `applyRules` method implements a sophisticated rules engine. Before starting a new InteractionChain, it checks against all currently active chains to determine if the new action is permitted, if it should interrupt an existing action, or if it is blocked.
-   **Simulation Handler:** It delegates the execution of individual interaction operations to an `IInteractionSimulationHandler`, decoupling the state management logic from the specific game mechanics of each action.
-   **Component-Based Design:** As an implementation of the `Component` interface, the InteractionManager is tightly bound to a single `LivingEntity`. It is not a global singleton; each interacting entity possesses its own instance, ensuring state isolation.

## Lifecycle & Ownership

-   **Creation:** An InteractionManager is instantiated by the entity system when a `LivingEntity` is created. It is a standard component attached to the entity's data structure. Its constructor requires the owning entity and a reference to the player's network handler if applicable.
-   **Scope:** The instance persists for the entire lifetime of its owning `LivingEntity`. All state, including active interaction chains and cooldowns, is encapsulated within this component.
-   **Destruction:** The InteractionManager and all its internal state are garbage collected when the parent `LivingEntity` is removed from the world's `EntityStore`. The `clear` method can be invoked to prematurely terminate all interactions and notify the client, typically during entity death or despawn.

## Internal State & Concurrency

-   **State:** The InteractionManager is highly stateful and mutable. Its primary state consists of the `chains` map, which holds all active `InteractionChain` instances. It also maintains cooldown timers, packet queues, and synchronization metadata. This state is volatile and changes continuously during the game loop.

-   **Thread Safety:** **Not thread-safe.** The InteractionManager is designed to be operated exclusively by the main server thread within the `tick` method. The `commandBuffer` field is temporarily populated during a tick and nulled afterwards, a pattern indicating single-threaded access.

    **WARNING:** Concurrent modification of its internal collections or state from outside the `tick` cycle will lead to critical simulation errors, desynchronization, and server instability. All interactions with this component must be dispatched through the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(ref, commandBuffer, dt) | void | O(N * M) | The main update loop. Processes packet queues and ticks all active chains. N is the number of active chains, M is the number of operations processed per tick. |
| tryStartChain(ref, cb, type, ctx, root) | boolean | O(C) | Attempts to initiate a server-side interaction. Validates rules against C active chains. Returns false if blocked. |
| sync(ref, storage, packet) | void | O(D) | Applies client state from a sync packet to a server-side chain. D is the number of data points in the packet. |
| cancelChains(chain) | void | O(F) | Recursively cancels an interaction chain and all its F forked sub-chains, sending cancellation packets to the client. |
| sendSyncPacket(chain, baseIndex, data) | void | O(1) | Queues a synchronization packet to be sent to the client, containing the server's authoritative state for an interaction. |
| getChains() | Int2ObjectMap | O(1) | Returns an unmodifiable view of the active interaction chains. |

## Integration Patterns

### Standard Usage

The InteractionManager is driven by the server's main entity update loop. Game logic systems initiate new interactions by calling `tryStartChain` or `startChain`.

```java
// Within a server-side game logic system (e.g., AI or PlayerInputSystem)
InteractionManager manager = entity.getComponent(InteractionManager.class);

// Example: Forcing an entity to perform an action
RootInteraction attackInteraction = RootInteraction.getAssetMap().getAsset("player_sword_attack");
InteractionContext context = InteractionContext.forInteraction(manager, ref, InteractionType.Held, commandBuffer);

// tryStartChain validates rules before execution
boolean success = manager.tryStartChain(ref, commandBuffer, InteractionType.Held, context, attackInteraction);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new InteractionManager()`. It is a component managed by the ECS framework and must be retrieved from its parent entity. Direct instantiation will result in a disconnected and non-functional manager.
-   **External State Modification:** Never directly add or remove entries from the map returned by `getChains`. Use the provided API methods like `tryStartChain` and `cancelChains` to manage the interaction lifecycle.
-   **Cross-Thread Access:** Do not call any method on the InteractionManager from any thread other than the main server thread. This will break the component's internal state and cause unpredictable behavior.

## Data Pipeline

The InteractionManager is a critical hub in the client-server data flow for player actions.

> **Client Input to Server Simulation:**
> Client Input -> Client-side Prediction -> `SyncInteractionChain` Packet -> Network Layer -> `GamePacketHandler` Packet Queue -> **InteractionManager.tick()** -> `tryConsumePacketQueue()` -> `syncStart()` / `sync()` -> `InteractionChain` State Update -> Server Game Logic

> **Server Authority to Client State:**
> Server Logic (e.g., AI) -> `tryStartChain()` -> New `InteractionChain` -> **InteractionManager.tick()** -> `sendSyncPacket()` -> `syncPackets` Queue -> Network Layer -> `SyncInteractionChain` Packet -> Client-side State Correction

---

