---
description: Architectural reference for TeleportBackCommand
---

# TeleportBackCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient

## Definition
```java
// Signature
public class TeleportBackCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The TeleportBackCommand class is a concrete implementation of the Command Pattern, specifically tailored for server-side player actions. It serves as a direct endpoint for the user-facing `/back` command, translating player input into a state change within the Entity Component System (ECS).

Architecturally, this class is a leaf node in the command processing tree. It is not designed to hold business logic. Instead, it acts as a lightweight adapter that:
1.  Parses command-specific arguments (in this case, an optional count).
2.  Identifies the target entity (the player executing the command).
3.  Retrieves the relevant component (`TeleportHistory`) from the entity.
4.  Delegates the core action to the component.

This design cleanly separates the command input layer from the gameplay logic layer, ensuring that the `TeleportHistory` component can be manipulated by other systems, not just player commands.

### Lifecycle & Ownership
-   **Creation:** A single instance of TeleportBackCommand is instantiated by the server's command registration system during the server bootstrap phase. It is discovered and registered to handle the "back" command identifier.
-   **Scope:** The singleton instance persists for the entire server lifetime. The `execute` method is invoked on this instance for every use of the `/back` command, but the parameters passed to it provide a transient, per-execution context.
-   **Destruction:** The object is dereferenced and eligible for garbage collection only upon server shutdown when the central command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is stateless. Its fields, like `countArg`, are final definitions for the command argument structure, not mutable state. All necessary state for an operation (the player, the world, the ECS store) is provided via method arguments to the `execute` function. This statelessness is critical for reusability and safety in a multi-threaded server environment.
-   **Thread Safety:** The class instance is inherently thread-safe. The `execute` method's operations on the ECS are subject to the concurrency guarantees of the underlying `Store`. Developers must assume that the `Store` and `Ref` objects correctly manage thread safety for component access and modification. This command introduces no independent concurrency hazards.

## API Surface
The public contract is minimal, intended for use only by the command system itself.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | The command's entry point. It orchestrates the retrieval of the TeleportHistory component and delegates the "back" operation to it. The complexity of this method is constant time, though the delegated call to `history.back` may have its own non-trivial complexity. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically registered and executed by the server's command handler. A player triggers its execution by typing the command in chat.

```
// Player-side action (conceptual)
/back 2
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new TeleportBackCommand()`. The command framework is responsible for the lifecycle of all command objects. Direct instantiation will result in a non-registered command that the server cannot execute.
-   **Manual Execution:** Calling the `execute` method directly from other game systems is a severe violation of the architecture. This bypasses critical infrastructure, including permission checks, argument validation, and context setup, which can lead to server instability or security vulnerabilities.

## Data Pipeline
The flow of data for a typical `/back` command execution follows a clear path from network input to world state change.

> Flow:
> Player Chat Input (`/back`) -> Server Network Layer -> Command Dispatcher -> **TeleportBackCommand.execute()** -> ECS Store (`store.ensureAndGetComponent`) -> TeleportHistory.back() -> Entity Transform Component Update -> World State Change -> Network Packet to Clients

