---
description: Architectural reference for SpawnDeployableAtHitLocationInteraction
---

# SpawnDeployableAtHitLocationInteraction

**Package:** com.hypixel.hytale.builtin.deployables.interaction
**Type:** Configurable Handler

## Definition
```java
// Signature
public class SpawnDeployableAtHitLocationInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The SpawnDeployableAtHitLocationInteraction class is a server-side, data-driven implementation of a specific game action: spawning a "deployable" entity at the precise point of a player's interaction with the world. It is a concrete subclass of SimpleInstantInteraction, signifying that it represents a discrete, one-off event rather than a continuous or charged action.

Architecturally, this class serves as a leaf node in the server's Interaction System. It is not intended for direct, programmatic invocation. Instead, instances are defined in external configuration files (e.g., JSON) and are deserialized at runtime via the provided CODEC. This pattern decouples game logic from game content, allowing designers to create new items or abilities that use this interaction without modifying engine code.

Its core responsibility is to translate a high-level InteractionContext, which contains information about the player's action, into a low-level world modification command. It achieves this by using a CommandBuffer, a standard engine pattern for queueing state changes to be executed deterministically at the end of a server tick. This ensures that world modifications are atomic and thread-safe.

## Lifecycle & Ownership
-   **Creation:** Instances are not created using the *new* keyword. They are deserialized from configuration data by the server's content loading pipeline using the static BuilderCodec field named CODEC. This typically occurs once during server startup or world loading.
-   **Scope:** An instance of this class, once loaded, persists for the lifetime of the server or world. It acts as a stateless template or prototype for the interaction type it represents.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or the world it belongs to is unloaded. There is no manual destruction logic.

## Internal State & Concurrency
-   **State:** The class holds a single state field: *config* of type DeployableConfig. This field is populated during deserialization and is considered **immutable** for the object's lifetime. The core logic within the firstRun method is stateless, operating exclusively on the parameters provided in the InteractionContext.
-   **Thread Safety:** The object's internal state is inherently thread-safe due to its immutability post-creation. However, the primary method, firstRun, is **not re-entrant** and is designed to be executed exclusively on the main server thread. It manipulates world state indirectly via a CommandBuffer, which is the designated mechanism for managing concurrency in the Hytale server architecture.

**WARNING:** Calling firstRun from any thread other than the main server tick thread will corrupt world state and lead to server instability. The CommandBuffer system is not designed for multi-threaded producers.

## API Surface
The public contract is defined by its parent class, SimpleInstantInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the interaction logic. Reads hit location from the context and queues a spawn command. This method is the entry point for the server's InteractionModule. |
| needsRemoteSync() | boolean | O(1) | Returns false, indicating the interaction itself does not require special state synchronization with the client. The resulting entity spawn is handled by separate systems. |

## Integration Patterns

### Standard Usage
This class is not used directly in code. It is configured as part of a larger component definition, typically for an item, in a data file. The server's InteractionModule invokes it automatically when a player triggers the associated action.

A conceptual configuration might look like this:

```json
// Example: rock.json (Conceptual)
{
  "id": "hytale:rock",
  "components": {
    "item": { ... },
    "interaction": {
      "type": "SpawnDeployableAtHitLocationInteraction",
      "Config": {
        "entityToSpawn": "hytale:deployable_campfire",
        "placementRules": [ ... ]
      }
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SpawnDeployableAtHitLocationInteraction()`. The object is uninitialized and useless without its *config* field, which is only populated by the CODEC during deserialization.
-   **Manual Invocation:** Do not call the *firstRun* method directly. This bypasses the server's core interaction pipeline, including permission checks, cooldowns, and context validation, which can lead to exploits or server crashes.
-   **State Mutation:** Do not attempt to modify the *config* field after the object has been loaded. This behavior is undefined and will affect all subsequent interactions of this type.

## Data Pipeline
The flow of data and control for this interaction follows a standard client-server-engine pattern.

> Flow:
> Client Player Input -> Network Packet (Interaction Request) -> Server Network Layer -> InteractionModule -> **SpawnDeployableAtHitLocationInteraction.firstRun()** -> CommandBuffer.queue(SpawnEntityCommand) -> End-of-Tick Command Flush -> EntityStore Update -> World State Synchronization -> Client Receives New Entity Data

