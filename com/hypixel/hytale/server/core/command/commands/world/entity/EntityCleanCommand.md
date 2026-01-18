---
description: Architectural reference for EntityCleanCommand
---

# EntityCleanCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient Command Object

## Definition
```java
// Signature
public class EntityCleanCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntityCleanCommand is a concrete implementation within the server's Command System framework. It is not a long-lived service but a stateless object designed to execute a single, high-impact world modification task: the bulk removal of all non-player entities.

Architecturally, this class serves as a direct interface between the user-facing command system and the server's underlying Entity Component System (ECS). Its primary design characteristic is its reliance on the ECS's high-performance, parallel processing capabilities. By invoking `forEachEntityParallel`, it delegates the iteration over the entire set of world entities to the core engine, which can distribute the work across multiple threads.

The use of a `commandBuffer` to queue entity removals is a critical architectural pattern. This defers the actual deletion of entities to a later, safe synchronization point in the game loop. This prevents hazardous race conditions and concurrent modification exceptions that would otherwise occur when modifying a collection while iterating over it, especially in a parallel context.

## Lifecycle & Ownership
- **Creation:** A single instance is created by the server's command registration system during the initial bootstrap phase. The system likely discovers this class via reflection or a predefined list and registers it in a central command map.
- **Scope:** The object instance is a singleton that persists for the entire lifetime of the server session. However, its execution is ephemeral, triggered only when the corresponding command is invoked.
- **Destruction:** The instance is discarded and garbage collected during server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It contains no mutable instance fields. All necessary context, such as the target world and its entity store, is provided as arguments to the `execute` method.
- **Thread Safety:** The class itself is immutable and inherently thread-safe. The `execute` method initiates a highly concurrent operation. The safety of this operation is guaranteed not by this class, but by the underlying ECS architecture which provides the `forEachEntityParallel` and `commandBuffer` primitives. This design correctly places the responsibility of concurrency control on the core data structure framework.

**WARNING:** The operation initiated by this command is thread-safe, but it is also computationally intensive. Executing it on a world with millions of entities can cause a temporary but significant performance drop (tick lag) as the engine processes the parallel work and subsequent command buffer.

## API Surface
The public API is defined by its parent, `AbstractWorldCommand`, and is intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | protected void | O(N/k) | Overrides the parent method to perform the entity cleanup. Complexity is proportional to the number of entities (N) divided by the number of available cores (k). Queues removal for all non-Player entities. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via code. It is invoked by an authorized user (e.g., a server administrator) through the server console or in-game chat interface.

```text
// User-facing invocation via server console
> entity clean
```

The server's command dispatcher receives this input, resolves it to the registered `EntityCleanCommand` instance, verifies permissions, and calls the `execute` method with the appropriate context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new EntityCleanCommand()`. The command system manages the lifecycle of command objects. Direct instantiation bypasses registration and will result in a non-functional command.
- **Manual Execution:** Do not acquire the command from a registry and call `execute` manually. This bypasses critical framework features like permission checks, argument parsing, and context provision, which can lead to instability or security vulnerabilities.

## Data Pipeline
The command initiates a data transformation pipeline within the Entity Component System. The flow is one-way, resulting in the mass deletion of data from the world state.

> Flow:
> User Input (`/entity clean`) -> Command Dispatcher -> **EntityCleanCommand.execute()** -> Parallel Query on EntityStore -> Removal operations queued in CommandBuffer -> ECS Synchronization Point -> Batched Entity Deletion from World State

