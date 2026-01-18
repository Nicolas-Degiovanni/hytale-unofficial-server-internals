---
description: Architectural reference for WarpCommand
---

# WarpCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.warp
**Type:** Transient

## Definition
```java
// Signature
public class WarpCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The **WarpCommand** class serves as the primary entry point and router for all warp-related console commands. It is not a command that executes logic itself, but rather a collection that aggregates and registers multiple sub-commands (e.g., set, list, remove, go) under the single `/warp` namespace. This pattern simplifies command registration and creates a clear, hierarchical command structure.

The core teleportation logic is encapsulated within the static helper method **tryGo**. This design centralizes the complex process of validating a warp, interacting with the Entity Component System (ECS), and queuing a teleport operation. By making this logic static, it can be shared across different command variants (e.g., `/warp <name>` and `/warp go <name>`) without code duplication, ensuring consistent behavior.

This class acts as a high-level controller that translates player input into low-level ECS operations. It reads an entity's current state via components like **TransformComponent**, and its primary output is the addition of a **Teleport** component to that entity. A separate, underlying system is responsible for detecting and processing this component to execute the actual world transition.

## Lifecycle & Ownership
-   **Creation:** An instance of **WarpCommand** is created by the server's command registration system during the server bootstrap phase. It is typically discovered and instantiated via the plugin loading mechanism. Developers should never instantiate this class directly.
-   **Scope:** The object persists for the entire server session. Once registered, it remains in memory to handle incoming commands.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down or the parent **TeleportPlugin** is unloaded.

## Internal State & Concurrency
-   **State:** The **WarpCommand** class is fundamentally stateless. It does not store or cache any runtime data. All state it interacts with is external, primarily from the singleton **TeleportPlugin** (which holds the list of defined warps) and the **EntityStore** (which holds player component data).
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. Command execution is synchronized with the server's primary game loop. The core **tryGo** method performs direct reads and writes on the **EntityStore**, which is an unprotected operation. Calling this logic from an asynchronous task will lead to race conditions, data corruption, and server instability.

## API Surface
The primary public contract of this class is its registration as a command handler. The key internal API, shared within its package, is the static **tryGo** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryGo(context, warp, ref, store) | void | O(1) | Executes the teleport sequence for a given entity. Validates warp existence, records teleport history, and adds a **Teleport** component to the target entity. Sends feedback messages to the command source. |

## Integration Patterns

### Standard Usage
This class is not invoked directly. Its sub-commands, such as **WarpGoCommand**, are responsible for parsing arguments and invoking the centralized teleport logic.

```java
// Example from a sub-command's execution logic
public class WarpGoCommand extends AbstractSubCommand {
    // ...
    @Override
    public void execute(@Nonnull CommandContext context, ...) {
        // Retrieve arguments from the context
        String warpName = ...;
        Ref<EntityStore> entityRef = ...;
        Store<EntityStore> entityStore = ...;

        // Delegate to the centralized logic handler
        WarpCommand.tryGo(context, warpName, entityRef, entityStore);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WarpCommand()`. The command system manages its lifecycle. Creating a new instance has no effect and will not be registered to handle commands.
-   **Asynchronous Execution:** Never call the **tryGo** method from a separate thread. All interactions with the **EntityStore** and game world must occur on the main server thread to prevent catastrophic concurrency failures.
-   **State Assumption:** Do not assume warps are loaded. The **tryGo** method correctly checks `TeleportPlugin.get().isWarpsLoaded()`, and any custom integration must perform the same check.

## Data Pipeline
The **WarpCommand** orchestrates a data flow that begins with user input and ends with a component mutation, which is later processed by a dedicated engine system.

> Flow:
> Player Command (`/warp ...`) -> Command System -> **WarpCommand** Sub-Command -> **WarpCommand.tryGo** -> Read **TeleportPlugin** State -> Read **TransformComponent** from Entity -> Write **TeleportHistory** to Entity -> Write **Teleport** component to Entity -> Send Message to Player

