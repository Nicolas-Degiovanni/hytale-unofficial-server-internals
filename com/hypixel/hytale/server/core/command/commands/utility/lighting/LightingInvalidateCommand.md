---
description: Architectural reference for LightingInvalidateCommand
---

# LightingInvalidateCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.lighting
**Type:** Transient

## Definition
```java
// Signature
public class LightingInvalidateCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The LightingInvalidateCommand is a server-side administrative command designed for debugging and world maintenance. It provides a direct interface for operators to trigger the recalculation of lighting data within the game world. This is a crucial tool for resolving visual artifacts, such as incorrect shadows or light levels, that may arise from world generation bugs or manual world edits.

This class extends AbstractWorldCommand, which anchors it within the server's command processing system and provides it with a verified World context upon execution. Its primary role is to act as a bridge between a user-initiated command and the low-level ChunkLightingManager. It translates a simple text command into a high-priority work request for the lighting engine.

The command supports two modes of operation:
1.  **Targeted Invalidation:** Using the `one` flag, it invalidates the lighting for the single chunk section currently occupied by the command's sender. This is a precise, low-impact operation for fixing localized issues.
2.  **Global Invalidation:** By default, it invalidates the lighting for *all* currently loaded chunks in the world. This is a resource-intensive, broad-spectrum operation intended to fix widespread lighting corruption.

## Lifecycle & Ownership
-   **Creation:** A single instance of LightingInvalidateCommand is created by the server's command registration system during the server bootstrap sequence. It is discovered via reflection or a manual registry and stored for the server's lifetime.
-   **Scope:** The object instance is a long-lived singleton managed by the command system, persisting for the entire server session. However, its execution context is transient; the `execute` method is invoked for a brief period each time the command is run and is then discarded.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The class contains a single, immutable state field: `oneFlag`. This field is initialized in the constructor to define the command's optional flag argument. The `execute` method itself is stateless, operating exclusively on the context and arguments provided to it. It does not cache data or modify its own state between invocations.
-   **Thread Safety:** This class is **not thread-safe** and must only be invoked from the main server thread. The command system guarantees this execution model. The `execute` method directly manipulates sensitive world state, including chunk data and the lighting manager's work queue. Calling this method from an asynchronous task would lead to severe world corruption, race conditions, and server instability.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) or O(N) | Entry point for the command. Complexity is O(1) when using the `one` flag (scheduling cost). Complexity is O(N) without the flag, where N is the number of loaded chunks. |

## Integration Patterns

### Standard Usage
This command is not intended for programmatic invocation by other systems. It is designed to be executed by a server administrator or a player with sufficient permissions via the in-game chat or server console.

**Example Console Invocation:**
```console
# Invalidate lighting for the chunk section at the player's position
/invalidate --one

# Invalidate lighting for all loaded chunks in the current world
/invalidate
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new LightingInvalidateCommand()`. The command will not be registered with the server's command system and will be non-functional.
-   **Manual Execution:** Do not call the `execute` method directly. This bypasses the command system's critical context setup, permission checks, and argument parsing, which will result in exceptions and undefined behavior.
-   **Asynchronous Invocation:** Never invoke this command's logic from a separate thread. World modification must be synchronized with the main server tick, a constraint enforced by the command system but not by the method signature itself.

## Data Pipeline
The command initiates a data flow that results in updated lighting information being sent to clients.

> Flow:
> User Input (`/invalidate`) -> Server Command Parser -> **LightingInvalidateCommand.execute()** -> World & ChunkStore Lookup -> ChunkLightingManager.addToQueue() / invalidateLoadedChunks() -> Asynchronous Lighting Engine -> Updated Chunk Data -> Network Packet -> Client Render Update

