---
description: Architectural reference for HitboxCollisionRemoveCommand
---

# HitboxCollisionRemoveCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.component.hitboxcollision
**Type:** Definition / Transient

## Definition
```java
// Signature
public class HitboxCollisionRemoveCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The HitboxCollisionRemoveCommand class is a structural component within the server's command processing system. It functions as a **Command Collection**, a container that groups related sub-commands under a common namespace. It does not implement any direct game logic itself; instead, it serves as the entry point for the `/hitboxcollision remove` command, delegating execution to one of its specialized inner classes.

This class and its sub-commands are a direct interface to the server's Entity-Component-System (ECS). Their primary function is to locate a specific entity within a world's EntityStore and surgically remove its HitboxCollision component. This action directly alters the physics and interaction behavior of the target entity by disabling its collision detection capabilities.

The design follows a **Composite Pattern**, where the parent command collection and its child sub-commands are treated uniformly by the command registration and dispatching system. This provides a clean, hierarchical command structure for developers and server administrators.

The two primary execution paths are:
1.  **HitboxCollisionRemoveEntityCommand:** Targets an arbitrary entity by its unique identifier.
2.  **HitboxCollisionRemoveSelfCommand:** A convenience command that targets the entity associated with the command's executor (the player).

## Lifecycle & Ownership

-   **Creation:** An instance of HitboxCollisionRemoveCommand is created a single time by the server's command registration service during the initial bootstrap sequence. The system scans for command definitions and instantiates them to build the command tree.
-   **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process. It is held as a reference within the central command registry.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only upon server shutdown when the command registry is cleared.

## Internal State & Concurrency

-   **State:** This class is effectively **immutable** after its constructor completes. It holds a static reference to a pre-translated message string and a list of its sub-commands, both of which are configured at creation and do not change during runtime. The class itself holds no per-execution or session-specific state.

-   **Thread Safety:** The HitboxCollisionRemoveCommand object itself is thread-safe due to its immutable nature. However, the execution of its sub-commands is **not** guaranteed to be safe if invoked from arbitrary threads. The command system's dispatcher ensures that the `execute` methods are called on the appropriate world's main thread. This synchronization is critical to prevent race conditions and concurrent modification exceptions when interacting with the EntityStore.

    **Warning:** Never invoke the `execute` methods directly from a separate thread. All command execution must be routed through the server's command dispatcher.

## API Surface

The primary API is the command-line interface it exposes. The programmatic surface is minimal and intended for use only by the command system. The core logic resides in the `execute` methods of the inner classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HitboxCollisionRemoveEntityCommand.execute(...) | void | O(log N) | Removes the HitboxCollision component from a specified entity. Complexity depends on the EntityStore's component lookup implementation. |
| HitboxCollisionRemoveSelfCommand.execute(...) | void | O(log N) | Removes the HitboxCollision component from the command executor's entity. Complexity is identical to the entity-specific version. |

## Integration Patterns

### Standard Usage

This class is not intended for direct programmatic use. It is invoked by a privileged user (developer or administrator) through the server console or in-game chat.

**Example Console Invocation:**
```sh
# Removes the hitbox from the entity with the specified ID
/hitboxcollision remove entity 12345

# A player removes their own hitbox
/hitboxcollision remove self
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not create instances using `new HitboxCollisionRemoveCommand()`. The command system handles the lifecycle of command objects. Manual instantiation serves no purpose and the instance will not be registered.

-   **Direct Method Invocation:** Bypassing the command dispatcher and calling an `execute` method directly is a severe architectural violation. This will skip critical setup, including context creation, permission validation, and argument parsing, and will likely lead to thread-safety issues and server instability.

    ```java
    // DANGEROUS: Bypasses the entire command system
    // This will fail because the context and other parameters are not available.
    HitboxCollisionRemoveEntityCommand cmd = new HitboxCollisionRemoveEntityCommand();
    // cmd.execute(context, world, store); // <-- DO NOT DO THIS
    ```

## Data Pipeline

The execution of this command follows a well-defined data flow from user input to state modification within the ECS.

> Flow:
> User Input (`/hitboxcollision remove entity 123`) -> Network Packet -> Server Command Parser -> Command Dispatcher -> **HitboxCollisionRemoveEntityCommand** -> `execute` method invoked -> `store.tryRemoveComponent` call -> EntityStore State Modified -> Feedback Message sent via `CommandContext` -> Network Packet -> User Console Output

