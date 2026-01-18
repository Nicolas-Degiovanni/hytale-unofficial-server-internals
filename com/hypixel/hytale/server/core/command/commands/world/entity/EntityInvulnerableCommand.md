---
description: Architectural reference for EntityInvulnerableCommand
---

# EntityInvulnerableCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityInvulnerableCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityInvulnerableCommand is a server-side command implementation that modifies the invulnerability state of one or more game entities. It operates within the server's Command System, a framework designed to process and execute text-based commands from players or the server console.

As a subclass of AbstractTargetEntityCommand, it inherits a significant portion of its functionality. This parent class provides the boilerplate logic for parsing entity selectors (e.g., @p, @a, or specific entity names), resolving them into a concrete list of entity references, and passing them to the execution logic.

This class's primary responsibility is to bridge the command system with the Entity Component System (ECS). Upon execution, it directly manipulates the components of the target entities by either adding or removing the **Invulnerable** component. This design cleanly separates the command parsing and targeting logic from the core game state modification, making the command a focused, single-responsibility class.

### Lifecycle & Ownership
-   **Creation:** A single instance of EntityInvulnerableCommand is created by the server's central command registry during the server bootstrap phase. The system scans for command classes and instantiates them for registration.
-   **Scope:** The command object itself is a singleton that persists for the entire server session. However, its execution is transient; the `execute` method is invoked for a brief moment to handle a specific command request and does not persist state between calls.
-   **Destruction:** The instance is discarded when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The class contains minimal, immutable state. The `removeFlag` field is an instance of FlagArg, which is initialized in the constructor and never modified. The core `execute` method is stateless, operating exclusively on the arguments passed to it, such as the CommandContext and the list of target entities. It does not cache data or maintain state across multiple invocations.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread. The command system ensures that all command executions are serialized and processed within the primary game loop. Direct invocation of the `execute` method from an asynchronous thread would lead to severe concurrency issues, as it performs direct, unsynchronized modifications to the shared ECS `Store`.

## API Surface
The public contract is defined by its role as a command, primarily through the overridden `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | protected void | O(N) | The primary entry point for command logic. Iterates through N target entities, adding or removing the Invulnerable component. Sends a feedback message to the command source. |

## Integration Patterns

### Standard Usage
A developer or user interacts with this class indirectly through the server's command-line interface. The system handles parsing, target selection, and invocation.

```
// Player or Console Input
/invulnerable @p
/invulnerable SomePlayer --remove
```

The above inputs are parsed by the command system, which then invokes the `execute` method on the registered EntityInvulnerableCommand instance with the appropriate context.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EntityInvulnerableCommand()`. The command system manages the lifecycle of all registered commands. Manually creating an instance bypasses registration and will not work.
-   **Manual Invocation:** Avoid calling the `execute` method directly. All command execution must flow through the central command processor to ensure permissions are checked, arguments are correctly parsed, and the execution context is valid. Bypassing this can lead to an inconsistent or corrupt game state.

## Data Pipeline
The data flow for this command is initiated by an external agent (player or console) and results in a direct mutation of the game state.

> **Execution Flow:**
> Player Input (`/invulnerable ...`) -> Network Packet -> Server Command Parser -> Target Selector Resolution -> **EntityInvulnerableCommand.execute()** -> ECS `Store.ensureComponent` / `Store.tryRemoveComponent` -> Game State Mutation

> **Feedback Flow:**
> **EntityInvulnerableCommand.execute()** -> `context.sendMessage()` -> Message Translation System -> Network Packet -> Player Chat UI

