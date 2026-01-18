---
description: Architectural reference for CanBreakRespawnPointInteraction
---

# CanBreakRespawnPointInteraction

**Package:** com.hypixel.hytale.builtin.adventure.objectives.interactions
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class CanBreakRespawnPointInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The CanBreakRespawnPointInteraction class is a server-authoritative rule that functions as a predicate or gatekeeper for block interactions. It is not a general-purpose service but a highly specific, data-driven behavior handler designed to be attached to blocks that function as player respawn points, such as beds or spawn crystals.

Its primary architectural role is to enforce ownership rules before allowing a destructive action, like breaking the block, to proceed. The core logic verifies if the entity initiating the interaction is the registered owner of the respawn point. If the respawn point has no owner, the interaction is permitted.

This class is a concrete implementation of the engine's data-driven interaction system. It is designed to be instantiated and configured via Hytale's asset loading pipeline, using the provided static CODEC field. This allows designers to apply this ownership rule to any block type without writing new code, simply by referencing it in the block's configuration files.

Crucially, this interaction declares its authority as server-side via the getWaitForDataFrom method. This instructs the engine that the client cannot predict the outcome of this interaction and must wait for the server's final judgment, preventing exploits where a client might incorrectly simulate breaking a respawn point they do not own.

## Lifecycle & Ownership
- **Creation:** Instances are created by the engine's Codec system during the asset loading phase. This class is not intended for manual instantiation by developers. It is deserialized from configuration files that define block behaviors.
- **Scope:** An instance is logically stateless. It may be cached by the asset system as a singleton prototype for the duration of the server session or instantiated on-demand for each interaction check. Its effective lifetime is ephemeral, tied only to the execution of a single interaction query.
- **Destruction:** Managed by the Java Garbage Collector. As a stateless object, it has no explicit cleanup or destruction logic.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields that store data between method calls. Its entire behavior is derived from the arguments passed into the interactWithBlock method.

- **Thread Safety:** The class is inherently **thread-safe** due to its stateless nature. However, it is designed to operate on world state objects (World, ChunkStore, CommandBuffer) that are **not thread-safe**. Therefore, all calls to its methods must be performed exclusively on the main server tick thread to prevent data corruption and race conditions within the world simulation.

## API Surface
The public contract is defined by its role as a SimpleBlockInteraction implementation. The primary method of interest is the server-side logic handler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the server-authoritative ownership check. Modifies the InteractionContext state to Finished if the check passes, or Failed if the interacting entity is not the owner. This method performs several constant-time lookups within the world's chunk data. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns WaitForDataFrom.Server, signaling to the interaction system that this logic is non-predictable and the client must await a server response. |

## Integration Patterns

### Standard Usage
This class is not invoked directly in procedural code. Instead, it is specified within a block's asset definition file (e.g., a JSON file) to be triggered by a specific player action. The engine's interaction module is responsible for invoking it.

A conceptual configuration might look like this:

```json
// Example: my_respawn_bed.json (Conceptual)
{
  "blockName": "my_respawn_bed",
  "components": {
    "RespawnBlock": {}
  },
  "interactions": [
    {
      "trigger": "ON_BLOCK_BREAK",
      "handler": {
        "type": "CanBreakRespawnPointInteraction"
      }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new CanBreakRespawnPointInteraction()`. The engine's data-driven systems manage its lifecycle. Manually creating it bypasses the entire configuration and asset loading pipeline.
- **Client-Side Simulation:** Do not attempt to replicate this ownership check on the client for predictive purposes. The design explicitly centralizes this authority on the server to prevent cheating.
- **Misconfiguration:** Attaching this interaction handler to a block that does not also have the RespawnBlock component will cause the check to always pass. The logic gracefully handles a missing RespawnBlock component by allowing the interaction, which may be an unintended security or gameplay loophole if misconfigured.

## Data Pipeline
The flow of data for an interaction governed by this class follows a strict client-server validation path.

> Flow:
> Player Input (Break Block) -> Client sends Interaction Packet -> Server Interaction Module receives packet -> **CanBreakRespawnPointInteraction.interactWithBlock()** is invoked -> Ownership is verified against World state -> InteractionContext state is set to `Finished` or `Failed` -> Server proceeds with or aborts the block breaking logic -> Server sends result packet to Client -> Client UI and World state are updated.

