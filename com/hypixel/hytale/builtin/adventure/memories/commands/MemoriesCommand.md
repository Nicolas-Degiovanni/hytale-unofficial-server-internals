---
description: Architectural reference for MemoriesCommand
---

# MemoriesCommand

**Package:** com.hypixel.hytale.builtin.adventure.memories.commands
**Type:** Transient

## Definition
```java
// Signature
public class MemoriesCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The MemoriesCommand class is a structural component that acts as a namespace or a command group. It does not implement any game logic itself. Instead, its sole responsibility is to aggregate multiple related sub-commands under a single, user-facing entry point: *memories*.

By extending AbstractCommandCollection, it participates in the server's command system, which uses a **Composite Pattern** to build a tree-like structure of commands. MemoriesCommand is a non-leaf node in this tree, delegating execution to its children (MemoriesClearCommand, MemoriesLevelCommand, etc.) based on the arguments provided by the user. This design decouples the parent command from its children, promoting modularity and making it simple to add or remove functionality from the memories system without altering the core command router.

### Lifecycle & Ownership
-   **Creation:** An instance of MemoriesCommand is created during the server's bootstrap sequence. A higher-level service, likely responsible for initializing the Adventure Mode systems, will instantiate this class and register it with the central CommandManager.
-   **Scope:** The object's lifecycle is tied to the server's command registry. It persists for the entire duration of a server session, from startup to shutdown.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down and the command registry is cleared. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The class is effectively **immutable** after its constructor completes. The list of sub-commands is populated once and is not designed to be modified at runtime. It holds no session-specific or mutable game state.
-   **Thread Safety:** This class is inherently thread-safe. Since its internal state does not change after construction, it can be safely accessed by the command processing system without locks. **Warning:** While this container is thread-safe, the thread safety of the sub-commands it executes is not guaranteed by this class and is the responsibility of each individual sub-command implementation. The server's command dispatcher is expected to execute commands on the main game thread.

## API Surface
The primary interaction with this class is through its constructor during system initialization. It has no other meaningful public API surface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MemoriesCommand() | constructor | O(1) | Constructs the command collection, registering four sub-commands for the "memories" namespace. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be registered with the server's command management system.

```java
// Example of how the server would register this command group
// This typically occurs once during server initialization.
CommandManager commandManager = serverContext.getCommandManager();
commandManager.register(new MemoriesCommand());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Creating an instance via `new MemoriesCommand()` without registering it with the CommandManager is useless. The object will have no effect on the server and will be immediately garbage collected.
-   **Stateful Logic:** Do not add fields or logic to this class that depend on game state. It is a routing and structural component only. All stateful operations must be delegated to the sub-commands.

## Data Pipeline
The MemoriesCommand acts as a router in the command processing pipeline. It inspects the user's input and dispatches the request to the appropriate sub-command for execution.

> Flow:
> Player Input (`/memories unlock ...`) -> Network Packet -> Server Command Parser -> **MemoriesCommand** (Routing) -> MemoriesUnlockCommand (Execution) -> Game Logic -> Player Feedback

