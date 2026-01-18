---
description: Architectural reference for ChainingInteraction
---

# ChainingInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient

## Definition
```java
// Signature
public class ChainingInteraction extends Interaction {
```

## Architecture & Concepts

The ChainingInteraction class implements a stateful "combo" or "chain" mechanic within the server's interaction system. It functions as a conditional router or a state machine node, redirecting the flow of execution to another Interaction based on the timing and sequence of player actions.

Architecturally, this class separates its static configuration from its dynamic runtime state. The ChainingInteraction object itself is a stateless, immutable blueprint loaded from asset files. The runtime state, such as the current position in a combo chain for a specific entity, is stored externally in a dedicated `ChainingInteraction.Data` component attached to that entity.

Its primary role is to evaluate the time elapsed since the last activation against a configured `chainingAllowance`. If the new activation occurs within this time window, it advances to the next Interaction in the `next` array. If the window is exceeded, the chain resets. This allows for the creation of gameplay mechanics like multi-stage attacks or sequential crafting steps.

The `compile` method is a critical part of the engine's bootstrap process. It translates the declarative structure of the `next` and `flags` arrays into a low-level jump table using the OperationsBuilder. This pre-compilation step optimizes runtime execution by converting a high-level configuration into a more efficient, direct-dispatch format.

This interaction requires client-side synchronization, as indicated by `needsRemoteSync` returning true and `getWaitForDataFrom` returning Client. The server is the authority, but the client runs predictive logic via `simulateTick0` to provide responsive feedback.

## Lifecycle & Ownership

-   **Creation:** Instances are deserialized and instantiated exclusively by the Hytale `CODEC` system during the loading of game assets (e.g., from JSON configuration files). The static `CODEC` field defines the schema for this process.
-   **Scope:** An instance of ChainingInteraction persists for the lifetime of the loaded game assets. It is treated as a shared, immutable configuration object, referenced by the InteractionManager.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the corresponding asset module or during server shutdown. There is no manual destruction method.

## Internal State & Concurrency

-   **State:** The ChainingInteraction object is effectively **immutable** after its fields are populated by the codec during the `afterDecode` phase. It holds configuration data like `chainingAllowance` and the list of subsequent interactions (`next`). All mutable, per-entity runtime state is managed externally in the `ChainingInteraction.Data` component.

-   **Thread Safety:** The class is **thread-safe for reads**. As a stateless configuration object, its methods can be safely invoked by multiple threads, provided the passed `InteractionContext` and `CommandBuffer` are thread-local or properly synchronized. State mutations are deferred through the `CommandBuffer` system, a core pattern in Hytale's ECS to ensure safe concurrent modifications to entity components.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | void | O(1) | Server-side execution logic. Reads client state from the context and jumps to the appropriate chained interaction. |
| simulateTick0(...) | void | O(1) | Client-side prediction logic. Updates the entity's Data component via a command buffer to advance or reset the chain state. |
| compile(builder) | void | O(N) | Translates the `next` and `flags` arrays into a jump table for the interaction virtual machine. N is the total number of chained interactions. |
| walk(collector, context) | boolean | O(N) | Traverses the graph of possible subsequent interactions for analysis or validation. N is the number of interactions in `next`. |
| configurePacket(packet) | void | O(N) | Populates a network packet with the configuration data required for the client to simulate this interaction. N is the total number of chained interactions. |

## Integration Patterns

### Standard Usage

This class is not intended for direct instantiation or invocation by game logic developers. It is designed to be defined declaratively within asset files. The engine's InteractionModule is responsible for loading, compiling, and executing these interactions.

A typical asset definition would resemble this conceptual structure:

```json
{
  "type": "ChainingInteraction",
  "id": "player_sword_combo",
  "chainId": "sword_combo_state",
  "chainingAllowance": 0.75,
  "next": [
    "asset:player_sword_swing_1",
    "asset:player_sword_swing_2",
    "asset:player_sword_swing_3"
  ]
}
```

The server's InteractionManager would then execute this interaction in response to a player action.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ChainingInteraction()`. The object is complex and requires its internal state to be configured by the `CODEC` system to function correctly.
-   **Stateful Implementation:** Do not modify the ChainingInteraction class to store per-entity state. This violates the design principle of separating configuration from runtime data and will break in a multithreaded environment. All state must be managed via ECS components like `ChainingInteraction.Data`.
-   **Manual Execution:** Avoid calling `tick0` or `simulateTick0` directly. All interactions should be processed through the InteractionManager to ensure proper context setup and lifecycle management.

## Data Pipeline

The ChainingInteraction is involved in two distinct data flows: a one-time configuration pipeline and a recurring runtime execution pipeline.

**Configuration Pipeline:**
> Asset File (JSON) -> Hytale `CODEC` System -> **ChainingInteraction Instance** -> `compile()` -> Interaction Operation List -> InteractionManager Registry

**Runtime Execution Pipeline (Server Authority):**
> Client Input Packet -> Network Layer -> `InteractionManager` -> `tick0` on **ChainingInteraction** -> Evaluation of `ChainingInteraction.Data` -> `InteractionContext.jump()` -> Execution of next Interaction in chain

