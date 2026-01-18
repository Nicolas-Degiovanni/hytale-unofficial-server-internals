---
description: Architectural reference for SpawnPrefabInteraction
---

# SpawnPrefabInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class SpawnPrefabInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The SpawnPrefabInteraction class is a data-driven command object responsible for placing a predefined structure, or Prefab, into the world. It represents a concrete, single-shot action within the server's broader Interaction System.

Its primary architectural role is to serve as a deserialized configuration payload. Rather than hard-coding placement logic, server designers define the behavior in external asset files (e.g., JSON). The static CODEC field handles the deserialization of this data into a fully-formed SpawnPrefabInteraction instance at runtime.

This class acts as a bridge between a high-level interaction trigger (like a player using an item) and the low-level world modification APIs. It encapsulates all necessary parameters—such as the target prefab, offset, and rotation—and orchestrates the placement process. Crucially, it does not modify the world state directly. Instead, it delegates this work to the PrefabUtil, which queues all block and entity changes into a CommandBuffer. This pattern ensures that world modifications are transactional, deterministic, and thread-safe, preventing partial updates and race conditions during the main server tick.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during server startup or asset hot-reloading. The server reads an interaction asset file, and the static CODEC deserializes the data into a new SpawnPrefabInteraction object. Direct manual instantiation is an anti-pattern.
- **Scope:** The object is stateless and transient. Its lifetime is tied to the loaded server configuration. An instance is effectively immutable post-creation and is used to execute the `firstRun` logic whenever its corresponding interaction is triggered. It does not persist beyond the execution of a single interaction event.
- **Destruction:** Managed by the Java garbage collector. As a simple data object with no external resources, it requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state (prefabPath, offset, rotationYaw, etc.) is configured once during deserialization and should be considered **immutable**. All fields are read-only during the execution of the `firstRun` method. The class does not cache world data or maintain any session-specific state.
- **Thread Safety:** This class is inherently thread-safe for read operations. The `firstRun` method is designed to be executed on the primary world thread. It achieves safe concurrency for world modifications by submitting all operations to the CommandBuffer, which serializes and executes them during a safe phase of the server game loop.

**WARNING:** Calling `firstRun` from multiple threads concurrently with a shared InteractionContext would be unsafe and lead to unpredictable behavior. The interaction system guarantees serialized execution.

## API Surface
The primary contract is the `firstRun` method, inherited from its parent and triggered by the interaction system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(P) | Executes the prefab placement logic. P is the number of blocks and entities in the prefab. Throws IllegalArgumentException for unhandled OriginSource. Aborts silently if the target chunk or prefab is not found. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. Instead, it is configured declaratively within an interaction asset file. The game engine is responsible for loading this configuration and triggering the interaction.

A conceptual asset definition might look like this:
```json
{
  "type": "SpawnPrefabInteraction",
  "PrefabPath": "prefabs/structures/small_well.pfb",
  "Offset": { "x": 0, "y": 1, "z": 0 },
  "RotationYaw": "None",
  "OriginSource": "BLOCK",
  "Force": false
}
```
The engine then uses this data to instantiate and execute the interaction when triggered.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SpawnPrefabInteraction()`. The object is not designed for manual setup and will be incomplete. Always define interactions in asset files to be loaded by the server's CODEC system.
- **Manual Execution:** Do not invoke the `firstRun` method directly. This bypasses the server's Interaction System, including critical checks for permissions, cooldowns, and event triggers.
- **State Mutation:** Do not attempt to modify the fields of a SpawnPrefabInteraction instance after it has been created. These are configuration values and are expected to be immutable.

## Data Pipeline
The flow of data and control for a prefab spawning event is linear and robust, ensuring safe world modification.

> Flow:
> Player Action -> Server Interaction System -> **SpawnPrefabInteraction.firstRun()** -> Prefab Asset Lookup -> Position & Rotation Calculation -> PrefabUtil.paste() -> CommandBuffer Queue -> Synchronized World Update Tick

