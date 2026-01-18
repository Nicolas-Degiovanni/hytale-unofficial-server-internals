---
description: Architectural reference for ChargingInteraction
---

# ChargingInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Data Model

## Definition
```java
// Signature
public class ChargingInteraction extends Interaction {
```

## Architecture & Concepts
The ChargingInteraction is a data-driven node within the server's interaction state machine. It models time-based "charge-up" mechanics, such as drawing a bow, charging a spell, or holding a block. It is a server-authoritative interaction that delegates the time-keeping responsibility to the client for responsiveness, making it a hybrid client-server behavior.

Architecturally, it serves three primary functions:
1.  **Client-Side Simulation Contract:** It defines the parameters for a client-side simulation, including whether to display a progress bar, if the charge can be held indefinitely, and how mouse sensitivity should be adjusted during the charge. This configuration is serialized and sent to the client upon initiation.
2.  **Server-Side State Gate:** On the server, its primary role is to wait for a definitive state update from the client. The `getWaitForDataFrom` method returns `WaitForDataFrom.Client`, which instructs the InteractionManager to pause execution of this interaction chain until a corresponding `InteractionSyncData` packet arrives.
3.  **Conditional Dispatcher:** Upon receiving the final charge duration from the client, it acts as a dispatcher. It uses the `next` map, which links charge time thresholds (in seconds) to subsequent Interaction assets, to determine the next state in the interaction chain. This allows for complex behaviors where a short press triggers one action and a long press triggers another.

A key feature is the concept of **forking**. While a ChargingInteraction is active (e.g., holding a shield up), the system can process other inputs (e.g., a primary attack button) to initiate a *new, parallel* interaction chain, such as a shield bash. This enables layered and complex combat abilities without terminating the original charge action.

## Lifecycle & Ownership
-   **Creation:** Instances are not created imperatively with `new`. They are deserialized and instantiated from game asset files (e.g., JSON definitions) by the `BuilderCodec` system during server startup or when configurations are hot-reloaded. A single instance represents the static definition of the interaction.
-   **Scope:** An instance of ChargingInteraction is a stateless template. Its lifetime is tied to the server's asset registry and persists for the entire server session. It is effectively a singleton from the perspective of game logic.
-   **Destruction:** The object is eligible for garbage collection only when the asset registry is cleared, typically during a full server shutdown.

## Internal State & Concurrency
-   **State:** The ChargingInteraction object is **effectively immutable** after being constructed by the codec. Its fields define the static configuration of the interaction and are not modified during runtime. All mutable state related to an entity's *active* charge (e.g., the current charge time) is managed externally within the `InteractionContext` and `InteractionSyncData` objects, which are specific to that entity's execution.
-   **Thread Safety:** This class is **thread-safe**. As a stateless and immutable configuration object, it can be safely accessed by multiple entity-processing threads simultaneously without requiring locks or synchronization. The engine ensures that each entity's `InteractionContext` is handled in a thread-safe manner.

## API Surface
The primary contract is with the InteractionManager, not end users.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Signals the engine that this interaction must wait for client data before proceeding. |
| tick0(...) | void | O(F) | Server-side execution logic. Processes client sync data, handles forks, and dispatches to the next interaction based on charge value. F is the number of active forks. |
| simulateTick0(...) | void | O(1) | Client-side prediction logic. Determines if the charge is ongoing or complete. **Warning:** This method is for simulation and does not execute on the server. |
| compile(...) | void | O(N) | Translates the declarative configuration into a low-level operation sequence for the state machine. N is the number of charge thresholds in the `next` map. |
| configurePacket(...) | void | O(N+F) | Serializes the interaction's configuration into a network packet for the client. N is the number of charge thresholds, F is the number of forks. |

## Integration Patterns

### Standard Usage
ChargingInteraction is not instantiated or called directly. It is defined within an asset file and referenced by other game objects, such as an item. The engine's InteractionManager is responsible for executing it.

**Conceptual Asset Definition (e.g., `charged_bow.json`)**
```json
{
  "id": "charged_bow_interaction",
  "type": "ChargingInteraction",
  "allowIndefiniteHold": false,
  "displayProgress": true,
  "next": {
    "0.5": "asset:weak_shot_interaction",
    "1.5": "asset:strong_shot_interaction"
  },
  "failed": "asset:fizzle_interaction"
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ChargingInteraction()`. The object will be uninitialized and will not be registered with the asset system, causing runtime failures. All interactions must be loaded via the codec.
-   **State Modification:** Do not attempt to modify the fields of a loaded ChargingInteraction at runtime. This is a shared, stateless template. Modifying it would affect all entities using that interaction and is not a thread-safe operation.
-   **Server-Side Time Calculation:** Do not attempt to calculate charge time within the server's `tick0` method. The design explicitly delegates this to the client. The server logic is designed to be a gate that waits for the client's final result.

## Data Pipeline
The flow of data for a complete charge-and-release action is a round trip between the client and server.

> Flow:
> 1.  **Client Input (Press):** Player presses and holds the action key.
> 2.  **Client Simulation:** The client's InteractionManager begins simulating the `ChargingInteraction`. It tracks elapsed time and may render a progress bar based on the configuration received from the server.
> 3.  **Client Input (Release):** Player releases the action key.
> 4.  **Client Finalization:** `simulateTick0` determines the interaction is finished and captures the final elapsed time as `chargeValue`.
> 5.  **Network Sync:** The client sends an `InteractionSyncData` packet to the server containing the final `chargeValue`.
> 6.  **Server Execution:** The server's InteractionManager, which was paused, receives the packet.
> 7.  **Server Dispatch:** It invokes `tick0` on the **ChargingInteraction** instance. The method reads the `chargeValue` from the packet.
> 8.  **State Transition:** `jumpToChargeValue` is called, which looks up the appropriate next interaction from the `next` map and instructs the InteractionManager to jump to that point in the compiled interaction chain.

