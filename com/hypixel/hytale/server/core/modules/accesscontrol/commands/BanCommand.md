---
description: Architectural reference for BanCommand
---

# BanCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient

## Definition
```java
// Signature
public class BanCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The BanCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's central command processing system. It serves as the primary entry point for administrators to enforce player bans.

Architecturally, this class acts as a controller that orchestrates several distinct server subsystems in response to a single user action. Its primary responsibilities are:
1.  **Input Parsing:** Defines and parses required (username) and optional (reason) arguments from the command input string.
2.  **Asynchronous Execution:** Extends AbstractAsyncCommand, ensuring that potentially long-running operations, such as external UUID lookups, do not block the main server thread. This is critical for server performance and stability.
3.  **State Modification:** Interacts with the HytaleBanProvider to persist the ban record, effectively modifying the server's access control state.
4.  **Player Session Management:** Locates the target player within the game Universe and terminates their network session if they are currently online.

This command is a critical component of the server's Access Control module, translating a high-level administrative intent into a series of low-level, asynchronous system calls.

### Lifecycle & Ownership
-   **Creation:** An instance of BanCommand is created during the server's bootstrap phase. The command registration service instantiates it, injecting a singleton instance of HytaleBanProvider. This follows a Dependency Injection pattern.
-   **Scope:** The object instance is a pseudo-singleton, registered with the command system and persisting for the entire lifetime of the server.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The class holds immutable references to its dependencies (HytaleBanProvider) and argument definitions. All mutable state related to a specific ban operation is confined to the local scope of the executeAsync method. The class itself is stateless after construction.
-   **Thread Safety:** This class is thread-safe. The executeAsync method is designed to be invoked by the command system's worker thread pool. All modifications to shared state, specifically the ban list, are delegated to the HytaleBanProvider, which is responsible for its own internal synchronization via its modify method.

## API Surface
The public contract is primarily defined by its constructor for dependency injection and the overridden executeAsync method for command execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BanCommand(HytaleBanProvider) | constructor | O(1) | Constructs the command, injecting the required ban provider dependency. |
| executeAsync(CommandContext) | CompletableFuture<Void> | O(N) | Asynchronously executes the ban logic. Complexity is dominated by network latency for UUID lookup and player disconnection. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command handler when an administrator issues the corresponding command.

The framework interaction is as follows:
1.  An administrator types `/ban PlayerName for breaking rules`.
2.  The command system parses `ban`, identifies this class as the handler, and creates a CommandContext.
3.  The system invokes the `executeAsync` method on the registered BanCommand instance, passing the context.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BanCommand()`. The command system manages its lifecycle and dependency injection. Manual creation will result in a non-functional command that is not registered to handle user input.
-   **Synchronous Blocking:** Do not call `future.get()` or any other blocking operation on the CompletableFuture returned by executeAsync from the main server thread. This defeats the purpose of the asynchronous design and will cause server stalls.
-   **Manual Execution:** Avoid calling the executeAsync method directly. This bypasses the permission checks, input validation, and context setup performed by the command system.

## Data Pipeline
The flow of data and control for a typical ban operation is sequential and asynchronous, crossing multiple system boundaries.

> Flow:
> Admin Command Input -> Command System Parser -> **BanCommand.executeAsync** -> AuthUtil (UUID Lookup) -> HytaleBanProvider (State Write) -> Universe (Player Lookup) -> Player PacketHandler (Network Disconnect) -> CommandContext (Feedback Message)

