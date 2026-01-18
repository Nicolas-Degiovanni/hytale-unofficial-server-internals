---
description: Architectural reference for WhoAmICommand
---

# WhoAmICommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Registered Handler

## Definition
```java
// Signature
public class WhoAmICommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WhoAmICommand class is a server-side command handler responsible for providing identity information about a player. It is a concrete implementation within the server's Command System, designed to be discovered and registered at startup.

Architecturally, this class demonstrates a powerful composition pattern for defining command variants. The primary class, WhoAmICommand, extends AbstractPlayerCommand. This base class provides a simplified contract for commands that are guaranteed to be executed by a player, injecting the executor's context directly into the *execute* method. This variant handles the zero-argument case, for example, when a player types `/whoami`.

For handling variants with arguments, such as `/whoami <other_player>`, it employs a nested static class, WhoAmIOtherCommand. This inner class extends the more generic CommandBase and is registered as a "usage variant". It is responsible for parsing its own arguments—in this case, a reference to another player.

A critical architectural concept demonstrated here is the interaction with the server's world-based threading model. The WhoAmIOtherCommand must look up a player who may exist in a different world, each managed by a separate thread. It correctly retrieves the target player's world and schedules the final logic via the *world.execute* method, ensuring thread-safe access to entity components.

## Lifecycle & Ownership
-   **Creation:** A single instance of WhoAmICommand is created by the server's command registration system during the bootstrap sequence. The system scans for command classes and instantiates them for its internal registry.
-   **Scope:** The command handler instance is a long-lived object. It persists for the entire duration of the server session, ready to be invoked by the command dispatcher.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is designed to be **stateless**. It contains no mutable instance fields and only holds final references to argument definitions. All state required for execution is passed into the *execute* or *executeSync* methods via the CommandContext.

-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the logic it performs is subject to the engine's strict concurrency rules.
    -   The primary *execute* method, inherited from AbstractPlayerCommand, is invoked on the executing player's world thread, making its access to the player's own data safe.
    -   The nested WhoAmIOtherCommand's *executeSync* method may be called from any thread. It demonstrates the mandatory pattern for cross-world entity interaction: it safely schedules its component lookups on the target world's execution queue.

    **WARNING:** Any logic that accesses or modifies an entity's components **must** be scheduled on that entity's parent world thread. Failure to do so will lead to race conditions, data corruption, and server instability.

## API Surface
The programmatic API is the execution contract with the command system. The user-facing API consists of the command strings and arguments.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Executes the zero-argument variant. Retrieves and sends info for the command's executor. |
| WhoAmIOtherCommand.executeSync(...) | protected void | O(1) | Executes the variant with a player argument. Involves a thread context switch to the target player's world. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly in code. It is invoked by the server's command dispatcher when a player enters the corresponding command in their client.

*A player executes the command in-game:*
```
/whoami
```
*Or, with appropriate permissions:*
```
/whoami OtherPlayerName
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WhoAmICommand()`. The command system handles instantiation and registration. An un-registered instance will never be executed.
-   **Direct Invocation:** Avoid calling the *execute* or *executeSync* methods directly. This bypasses critical infrastructure such as permission checks, argument parsing, and context injection provided by the command dispatcher.
-   **Off-Thread Entity Access:** In any similar command implementation, never access an entity's components directly from an *executeSync* method without scheduling the operation on the entity's world thread, as demonstrated in WhoAmIOtherCommand.

## Data Pipeline
The flow for this command is a standard request-response cycle initiated by a player.

> Flow:
> Player Chat Input -> Client Network Packet -> Server Command Parser -> **Command Dispatcher** -> WhoAmICommand Instance -> EntityStore Lookup -> Message Builder -> Server Network Packet -> Client Chat Display

