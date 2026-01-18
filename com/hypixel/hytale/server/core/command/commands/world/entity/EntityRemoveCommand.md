---
description: Architectural reference for EntityRemoveCommand
---

# EntityRemoveCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Service / Command Handler

## Definition
```java
// Signature
public class EntityRemoveCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntityRemoveCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's command processing system. It serves as a high-level, user-facing interface for performing destructive operations on the world's Entity Component System (ECS).

Its primary architectural role is to translate a text-based command, issued by a player or the server console, into a safe and validated world state mutation. It decouples the command invocation logic from the core ECS implementation by abstracting the complex process of entity removal. The class declaratively defines its expected inputs—a target entity and optional flags—using the argument-parsing framework, which handles the parsing and type conversion.

This command distinguishes between two primary modes of operation, controlled by the *others* flag:
1.  **Targeted Removal:** The default mode, which removes a single, specified entity.
2.  **Bulk Removal:** A high-impact mode that removes all entities in the world except for players and the specified target.

Crucially, the command incorporates safety checks, particularly verifying that a player can actually see an entity before allowing its removal. This prevents exploits where a player could otherwise manipulate entities outside their awareness range.

### Lifecycle & Ownership
-   **Creation:** A single instance of EntityRemoveCommand is instantiated by the server's command registration system during the initial server bootstrap phase. It is not created on a per-execution basis.
-   **Scope:** Application-scoped. The instance is held in a central command registry and persists for the entire lifetime of the server process.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless and immutable after its initial construction. Its fields, which define the command's arguments, are configured once in the constructor and are not modified during execution. All state required for an operation is passed into the execute method via its parameters.

-   **Thread Safety:** The object instance is inherently thread-safe due to its stateless design. The methods it invokes, however, operate on the highly concurrent ECS data store. The use of `forEachEntityParallel` and the submission of removal operations to a `commandBuffer` indicate that the underlying system is designed for concurrency. Operations are not applied immediately but are queued for safe processing within the engine's main tick loop, preventing race conditions and ensuring world state consistency.

## API Surface
The public API is primarily for framework consumption. The static helper method provides a reusable, safe way to perform the core logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) or O(1) | Framework entry point. Complexity is O(1) for single removal but O(N) when the *others* flag is used, where N is the total number of entities. |
| removeEntity(playerRef, entityRef, accessor) | static void | O(1) | Core logic for safely removing a single entity. Includes a critical visibility check if the action is initiated by a player. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly. The server's command system invokes the `execute` method in response to a player or console command. However, the static `removeEntity` method can be used programmatically by other server systems to safely remove an entity with the same validation logic as the command.

```java
// Example: A game system programmatically removing a specific entity
// while ensuring safety checks are performed.

// Assume 'world' and 'targetEntityRef' are already acquired.
ComponentAccessor<EntityStore> accessor = world.getStore(EntityStore.class);

// To remove without a player context (e.g., by the system itself)
EntityRemoveCommand.removeEntity(null, targetEntityRef, accessor);

// To remove on behalf of a player, including visibility checks
Ref<EntityStore> playerEntityRef = ...;
EntityRemoveCommand.removeEntity(playerEntityRef, targetEntityRef, accessor);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EntityRemoveCommand()` in game logic. The created instance is inert unless registered with the command system, which only happens at server startup.
-   **Bypassing Safety Checks:** Directly calling `componentAccessor.removeEntity` to replicate this command's function is dangerous. It bypasses the critical `EntityViewer` visibility check, potentially allowing players to remove entities they cannot see, which can be a vector for exploits.
-   **Stateful Modification:** Adding mutable instance fields to this class to store state between executions will break its design, violate thread safety, and lead to unpredictable behavior in a concurrent environment.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a modification to the global world state.

> Flow:
> Player Chat Input (`/remove`) -> Network Layer -> Server Command Parser -> **EntityRemoveCommand.execute()** -> ECS Command Buffer -> World State Update (End of Tick)

