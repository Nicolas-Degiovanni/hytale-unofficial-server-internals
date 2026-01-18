---
description: Architectural reference for TeleportInstanceInteraction
---

# TeleportInstanceInteraction

**Package:** com.hypixel.hytale.builtin.instances.interactions
**Type:** Data-Driven Object

## Definition
```java
// Signature
public class TeleportInstanceInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The TeleportInstanceInteraction is a server-authoritative, data-driven class that orchestrates the process of moving a player entity from one World to a separate, dynamically-generated "instance" World. It serves as a critical link between the player-facing Interaction System and the server's core Universe and World management systems.

This class is not intended for direct programmatic use. Instead, it is configured within game asset files (for example, as a component of a block's behavior) and deserialized at runtime by the engine's asset loader via its static CODEC field. This pattern allows designers and developers to create complex portal and instancing mechanics without writing new Java code.

Its primary responsibilities include:
-   Resolving or creating a target instance World based on configuration.
-   Managing the lifecycle of block-associated instances, including creation and potential cleanup.
-   Calculating and persisting a "return point" transform for when the player eventually leaves the instance.
-   Applying a PendingTeleport component to the player entity, which signals the network and entity systems to begin the world transition process.

The entire operation is designed to be non-blocking to prevent stalling the main server thread. It heavily leverages CompletableFuture to handle the high-latency operation of spawning a new World asynchronously.

## Lifecycle & Ownership
-   **Creation:** An instance of TeleportInstanceInteraction is never created using the `new` keyword in gameplay logic. It is deserialized from an asset definition file (e.g., JSON or HOCON) by the engine's asset pipeline using the provided static CODEC. It exists as part of a larger configuration, such as a BlockType.
-   **Scope:** The object is effectively a stateless configuration template. Its lifetime is tied to the asset it is a part of. It persists in memory as long as the parent asset is loaded by the server.
-   **Destruction:** The object is eligible for garbage collection when its parent asset is unloaded from the Asset Registry, typically during a server shutdown, world change, or resource reload.

## Internal State & Concurrency
-   **State:** The object's state is considered **immutable** after its initial creation and deserialization via the CODEC. All its configuration fields, such as instanceName and positionOffset, are set once during the asset loading phase and are only read from during the execution of the `firstRun` method.
-   **Thread Safety:** The class itself is inherently thread-safe due to its immutable nature. However, its primary method, `firstRun`, is designed to be executed exclusively on a World's main tick thread. It interacts with non-thread-safe systems like the CommandBuffer and ChunkStore.

    **WARNING:** The use of `CompletableFuture.thenRunAsync(..., world)` is a deliberate and critical concurrency pattern. It ensures that world-modifying logic, which runs after the asynchronous instance creation, is scheduled back onto the correct world thread, preventing race conditions and data corruption.

## API Surface
The public contract is minimal, as the class is primarily driven by the interaction system, not direct calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | Non-Blocking | Executes the core teleportation logic. Initiates an asynchronous world-spawning operation if required. Throws IllegalArgumentException if configured with OriginSource.BLOCK but no target block is available. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns WaitForDataFrom.Server, indicating this interaction is server-authoritative and the client should await server confirmation. |

## Integration Patterns

### Standard Usage
This class is not instantiated or called directly. It is configured within a game asset definition. The following pseudo-JSON demonstrates how it would be defined as part of a block's interaction list.

```json
// Example: my_portal_block.json
{
  "interactions": [
    {
      "type": "TeleportInstanceInteraction",
      "InstanceName": "dungeon/level_one",
      "OriginSource": "BLOCK",
      "PositionOffset": { "x": 0.0, "y": 1.0, "z": 0.0 },
      "PersonalReturnPoint": true,
      "CloseOnBlockRemove": true
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TeleportInstanceInteraction()`. This bypasses the CODEC-based deserialization, resulting in a default, unconfigured object that will fail at runtime.
-   **State Mutation:** Do not attempt to modify the fields of a loaded TeleportInstanceInteraction at runtime. This violates its immutable design and can lead to unpredictable behavior for all interactions using that shared asset.
-   **Blocking Operations:** Do not modify `firstRun` to include blocking I/O or long-running computations. The asynchronous pattern using CompletableFuture must be preserved to maintain server performance.

## Data Pipeline
The flow of data and control for a typical player-triggered teleport is a multi-system process orchestrated by this class.

> Flow:
> Player Input (e.g., right-click) -> Server receives Interaction Packet -> Interaction System resolves to **TeleportInstanceInteraction** -> `firstRun` is invoked -> `InstancesPlugin.spawnInstance` is called (async) -> `CompletableFuture<World>` is created -> Player Entity receives `PendingTeleport` component -> Entity System and Network Layer handle the world transition -> Client begins loading the new instance World.

