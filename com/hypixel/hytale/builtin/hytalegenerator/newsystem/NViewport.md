---
description: Architectural reference for NViewport
---

# NViewport

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem
**Type:** Transient

## Definition
```java
// Signature
public class NViewport {
```

## Architecture & Concepts
The NViewport class is a high-level orchestrator for bulk chunk operations within a defined spatial boundary. It acts as a command object, translating a world-space bounding box (Bounds3i) into a discrete set of chunk indices. Its primary role is to manage the asynchronous loading of these chunks from the persistent storage layer into active memory.

This component is not a persistent service but rather a transient utility instantiated to perform a specific, wide-area task. It bridges high-level server commands, such as world generation previews or administrative refreshes, with the low-level, asynchronous capabilities of the World's ChunkStore. By abstracting away the complexity of chunk index calculation and concurrent data fetching, it provides a simple interface for ensuring a region of the world is fully loaded.

## Lifecycle & Ownership
- **Creation:** An NViewport is instantiated directly via its constructor, typically within a server command handler or a specialized world generation system. It is not managed by a service locator or dependency injection container.
- **Scope:** The object's lifetime is ephemeral and tied directly to the scope of the operation that created it. It is designed to be created, used for a single `refresh` call, and then discarded.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the caller releases its reference. The asynchronous operations initiated by `refresh` will continue to completion independently of the NViewport object's lifecycle.

## Internal State & Concurrency
- **State:** The NViewport is effectively immutable after construction. Its internal fields, including the reference to the World and the calculated set of `affectedChunkIndices`, are final. The set of chunks it operates upon is determined once in the constructor and cannot be modified.

- **Thread Safety:** This class is conditionally thread-safe. The `refresh` method can be safely invoked from any thread, as it dispatches its work to the underlying ChunkStore, which is expected to be a thread-safe system. The internal state is read-only during the `refresh` operation.

    **Warning:** While the class itself is safe, invoking `refresh` multiple times on the same instance is redundant and will generate unnecessary load on the ChunkStore. The object is intended for a single invocation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NViewport(Bounds3i, World, CommandSender) | constructor | O(N) | Creates a viewport. Complexity is proportional to the number of chunks (N) within the bounds. |
| refresh() | void | O(N) | Asynchronously requests all chunks within the viewport to be loaded from the ChunkStore. The method returns immediately. |

## Integration Patterns

### Standard Usage
The NViewport is designed to be used as a fire-and-forget mechanism for loading a world area. It is typically instantiated and invoked within the context of a server command.

```java
// Example from a server command execution handler
public void execute(CommandSender sender, World world, Bounds3i targetArea) {
    // 1. Instantiate the viewport for the target area
    NViewport viewport = new NViewport(targetArea, world, sender);

    // 2. Trigger the asynchronous refresh operation
    viewport.refresh();

    // 3. The command can complete while chunks load in the background
    sender.sendMessage("Asynchronous refresh initiated for the specified area.");
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not maintain a reference to an NViewport instance in a long-lived service or component. It holds a reference to the World and a potentially large set of chunk indices, which can lead to memory leaks if not properly discarded after use.
- **State Modification:** Do not attempt to modify the NViewport after creation. It is an immutable command object; if a different area needs to be refreshed, create a new instance.
- **Synchronous Blocking:** Do not attempt to block program execution waiting for the `refresh` operation to complete. The entire design is asynchronous; leverage the `CompletableFuture` chain if subsequent actions depend on the chunks being loaded.

## Data Pipeline
The NViewport processes data by converting a high-level spatial query into a series of low-level, asynchronous storage requests.

> Flow:
> Server Command (with Bounds3i) -> **NViewport Constructor** (calculates chunk indices) -> **NViewport.refresh()** (dispatches async requests) -> ChunkStore -> In-Memory World Cache

