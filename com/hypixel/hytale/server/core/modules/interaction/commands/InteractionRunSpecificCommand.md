---
description: Architectural reference for InteractionRunSpecificCommand
---

# InteractionRunSpecificCommand

**Package:** com.hypixel.hytale.server.core.modules.interaction.commands
**Type:** Transient

## Definition
```java
// Signature
public class InteractionRunSpecificCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InteractionRunSpecificCommand class is a concrete implementation of a server-side command, designed to bridge user input with the core Interaction System. It functions as a specialized entry point for administrators and developers to manually trigger a specific interaction asset on a player entity.

Architecturally, this class resides at the edge of the application, translating structured text input into a high-level call to a core game service, the InteractionManager. It is not responsible for the logic of the interaction itself; its sole purpose is to:
1.  Define the required arguments: an InteractionType (e.g., PRIMARY) and a RootInteraction asset.
2.  Parse these arguments from a CommandContext provided by the command system.
3.  Acquire a reference to the server's InteractionManager.
4.  Construct the necessary InteractionContext.
5.  Delegate the creation and execution of the interaction to the InteractionManager.

This command is a critical tool for debugging and testing interaction chains without needing to fulfill their normal in-game trigger conditions.

### Lifecycle & Ownership
-   **Creation:** A single instance is created by the server's command registration system during server bootstrap. It is discovered and registered as part of the broader interaction module.
-   **Scope:** The object is a stateless singleton for the command definition. It persists for the entire server session.
-   **Destruction:** The instance is de-referenced and eligible for garbage collection when the server shuts down or the command is explicitly unregistered.

## Internal State & Concurrency
-   **State:** This class is effectively immutable after construction. Its fields, which define the command's argument structure, are final. The execute method is stateless and operates exclusively on the parameters passed into it, ensuring that each command execution is independent and does not modify the command object itself.

-   **Thread Safety:** **This class is not thread-safe.** The execute method is designed to be called by the server's command system, which operates on the main server thread. It directly accesses and manipulates core game components like the EntityStore and InteractionManager, which are not designed for concurrent access. All invocations must be synchronized with the main server game loop.

## API Surface
The primary contract is the protected `execute` method, invoked by the parent command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses arguments and queues an interaction. The complexity of the command itself is constant time, but it triggers the InteractionManager, whose subsequent operations may be complex. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked by the server's command handler when a privileged user types the command into the console or chat.

*Example command-line invocation:*
```plaintext
/interaction run specific PRIMARY hytale:my_special_interaction
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new InteractionRunSpecificCommand()`. The command system is responsible for the lifecycle of all registered commands.
-   **Manual Execution:** Do not call the `execute` method directly. This bypasses critical infrastructure, including argument parsing, permission checks, and context validation, which can lead to server instability or crashes.

## Data Pipeline
This command acts as a controller that initiates a data flow into the Interaction System.

> Flow:
> User Command String -> Server Command Parser -> **InteractionRunSpecificCommand.execute()** -> InteractionManager.initChain() -> InteractionManager.queueExecuteChain() -> Interaction System Tick -> World State Mutation

