---
description: Architectural reference for NPCRoleCommand
---

# NPCRoleCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Command Handler

## Definition
```java
// Signature
public class NPCRoleCommand extends NPCWorldCommandBase {
```

## Architecture & Concepts

The NPCRoleCommand class is an implementation of the Command Pattern, serving as a user-facing entry point for server administrators and developers to interact with the Non-Player Character (NPC) role system. It is registered with the server's central command processing system and is invoked in response to specific text-based commands entered by a user.

Architecturally, this class acts as a thin translation layer. It is responsible for:
1.  Defining the command's syntax, including its name ("role") and required arguments (the target role).
2.  Parsing and validating the user-provided arguments using the command system's argument parsing framework.
3.  Retrieving the necessary context, such as the target NPCEntity and the World it resides in, via its base class NPCWorldCommandBase.
4.  Delegating the core business logic to a dedicated system, in this case, the RoleChangeSystem.

This separation of concerns is critical. The command class does not contain the complex logic for how an NPC's role is changed. Instead, it acts as a secure and structured gateway to the underlying game system, preventing direct, uncontrolled manipulation of game state. The class also includes a nested GetRoleCommand variant, which follows the same architectural pattern for read-only operations.

## Lifecycle & Ownership

-   **Creation:** A single instance of NPCRoleCommand is created by the server's command registry during the server bootstrap phase. The system scans for classes that implement the command interface and instantiates them once.
-   **Scope:** The command object is a long-lived singleton for the duration of the server's runtime. It is stored in a central command map or registry and is not garbage collected until server shutdown. The *execution* of the command, however, is ephemeral and tied to a transient CommandContext object for each invocation.
-   **Destruction:** The instance is destroyed when the server shuts down and the command registry is cleared. No manual cleanup is required.

## Internal State & Concurrency

-   **State:** The NPCRoleCommand instance holds immutable state related to its definition, specifically the RequiredArg object for the role argument. This state is configured in the constructor and never changes. The execute method itself is stateless; it operates exclusively on the arguments passed into it and does not modify its own instance fields.
-   **Thread Safety:** **This class is not thread-safe and must only be accessed by the main server thread.** Command execution is synchronized with the server's primary game loop (tick). All interactions with the World, EntityStore, and NPCEntity are inherently unsafe if performed from a concurrent thread. The design relies entirely on the single-threaded execution model of the server core.

## API Surface

The primary contract is the protected execute method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NPCRoleCommand() | constructor | O(1) | Initializes the command definition, name, and required arguments. |
| execute(...) | protected void | O(1) | Parses context and delegates the role change request to the RoleChangeSystem. |

## Integration Patterns

### Standard Usage

This class is not intended for direct invocation in game logic code. It is designed to be discovered and executed exclusively by the server's command handling system. A user, such as a server administrator, would invoke it via the server console or in-game chat.

*Example user command:*
`/npc select <npc_id> role <role_asset_name>`

The system-level interaction that invokes the command would conceptually look like this:

```java
// Conceptual example of how the Command System invokes this class.
// DO NOT call this method directly in your own code.
CommandContext context = createFromPlayerInput("/npc role ...");
NPCRoleCommand command = commandRegistry.find("npc role");

// The system finds the target NPC and then executes the command
NPCEntity targetNpc = findNpcFromContext(context);
World world = targetNpc.getWorld();
// ... and so on
command.execute(context, targetNpc, world, store, ref);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new NPCRoleCommand()` in your application logic. The command registry handles the lifecycle of this object.
-   **Bypassing the Command System:** Do not attempt to call the `execute` method directly. This bypasses critical infrastructure for permission checks, argument parsing, and context setup, leading to unpredictable behavior and potential server instability.
-   **Replicating Logic:** If you need to change an NPC's role programmatically from another game system, do not copy the logic from this command. Instead, directly interface with the authoritative system it uses.

    ```java
    // BAD: Re-implementing the command's logic
    BuilderInfo roleInfo = findRoleByName("someRole");
    RoleChangeSystem.requestRoleChange(ref, npc.getRole(), roleInfo.getIndex(), true, store);

    // GOOD: The above logic is correct, but demonstrates that the command
    // is merely a wrapper. Use the RoleChangeSystem directly for system-to-system integrations.
    ```

## Data Pipeline

The NPCRoleCommand functions as a control point in a user-initiated data flow. It translates a user's text command into a systemic state change request.

> Flow:
> User Text Input -> Server Command Parser -> **NPCRoleCommand** -> RoleChangeSystem -> Entity Component System (ECS) -> NPCEntity State Updated

