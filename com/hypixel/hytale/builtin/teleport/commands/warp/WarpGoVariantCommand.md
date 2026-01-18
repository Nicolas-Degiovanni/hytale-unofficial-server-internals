---
description: Architectural reference for WarpGoVariantCommand
---

# WarpGoVariantCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.warp
**Type:** Transient Command Handler

## Definition
```java
// Signature
public class WarpGoVariantCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WarpGoVariantCommand class is a concrete implementation of the Command pattern, designed to integrate specifically with the server's command processing system. It represents a leaf node in the command tree, responsible for handling the player-facing action of teleporting to a predefined warp point.

Its primary architectural function is to act as a translator between raw player input and the core server teleportation logic. It achieves this by:
1.  **Defining Arguments:** It declares the required arguments for the command, in this case a single string representing the warp's name.
2.  **Enforcing Permissions:** It integrates with the HytalePermissions system to ensure the executing player has the necessary privileges (`warp.go`).
3.  **Contextual Execution:** By extending AbstractPlayerCommand, it guarantees that the command can only be executed by a player entity, providing the necessary player context for the operation.
4.  **Delegation:** It does not contain the teleportation logic itself. Instead, it extracts the necessary information from the CommandContext and delegates the operation to the centralized WarpCommand utility. This maintains a clean separation of concerns, where this class handles command parsing and validation, while WarpCommand manages the state and logic of warps.

## Lifecycle & Ownership
-   **Creation:** A single instance of WarpGoVariantCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is discovered and registered alongside other built-in commands.
-   **Scope:** The object instance persists for the entire duration of the server session. It is held in memory by the central command registry, ready to be invoked.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless from an execution perspective. It holds a final, immutable reference to a RequiredArg object, which serves as a template for parsing arguments. This state is configured once at construction and is never mutated. All execution-specific data is passed via method parameters in the `execute` call.
-   **Thread Safety:** The instance is inherently thread-safe due to its immutable internal state. The `execute` method is designed to be called by the server's main thread, which processes game logic and commands sequentially. Direct invocation from other threads is not a supported use case and would bypass the server's core processing loop.

## API Surface
The public API is minimal and dictated by its base class. The primary entry point is the `execute` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses the warp name from the context and delegates the teleportation logic to WarpCommand.tryGo. The complexity of this method is constant; however, the delegated call may have higher complexity (e.g., database lookup for the warp location). |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and used by the server's command system. A user interacts with it by typing the corresponding command in the game client.

The system's interaction with a registered instance is conceptually similar to the following:

```java
// PSEUDO-CODE: How the command system invokes this handler

// 1. System parses player input and finds the matching command handler
WarpGoVariantCommand handler = commandRegistry.getHandlerFor("/warp go");

// 2. System builds the execution context
CommandContext context = buildContextForPlayer(player, "my_warp_name");
Store<EntityStore> store = world.getPrimaryEntityStore();
Ref<EntityStore> ref = store.getRef();
PlayerRef playerRef = player.getRef();

// 3. System invokes the handler
handler.execute(context, store, ref, playerRef, world);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WarpGoVariantCommand()` in your own code. Command objects are managed exclusively by the server's command registration lifecycle.
-   **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the permission checks, argument parsing, and context validation performed by the upstream command processing system, which can lead to an inconsistent or unstable state.

## Data Pipeline
The class acts as a specific stage in the server's command processing pipeline, transforming a structured command context into a specific, high-level game action.

> Flow:
> Player Chat Input (`/warp go my_base`) -> Network Packet -> Server Command Parser -> **WarpGoVariantCommand** -> `WarpCommand.tryGo` -> Player Component Mutation (Position) -> Network Sync to Client

